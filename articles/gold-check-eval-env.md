---
title: "モデルを疑う前に評価環境を疑う：LLMコード評価ハーネスの「gold check」"
emoji: "🥇"
type: "tech"
topics: ["python", "pytest", "testing", "llm", "evaluation"]
published: true
---

## はじめに

こんにちは、ZEN大学2年の松延です。AIエンジニアを目指して、LLMまわりの開発を勉強しながら手を動かしています。

最近、LLMが生成したコードを自動テスト（pytest）で採点する、ローカルの評価ハーネスを作っていました。「タスクを渡す → モデルがコードを書く → テストを流す → 通ったら正解」という、よくある仕組みです。

その中で、**モデルではなく評価環境のほうが原因でテストが落ちていた**ことに気づきました。この記事では、そのときの経験と、対策として取り入れた「gold check」というひと手間を紹介します。

## 何が起きたか

ハーネスを動かしてみると、どうしても通らないタスクがいくつかありました。最初は「モデルがこの種類の問題を苦手にしているのかな」と考えていました。

ところが、試しに**正解（参照解）をそのまま当ててテストを流してみたら、それでも落ちました**。正解のコードなのに、です。

ログを見ると、原因はとても単純でした。

```text
E   ModuleNotFoundError: No module named 'inline_snapshot'
```

テストファイルが import しているライブラリが、評価環境にインストールされていなかったのです。実際に引っかかったのは、たとえば次のようなテスト用ライブラリでした。

- [inline-snapshot](https://pypi.org/project/inline-snapshot/)（[GitHub](https://github.com/15r10nk/inline-snapshot)）：テストコード中にスナップショットの期待値を書き込めるライブラリ
- [dirty-equals](https://pypi.org/project/dirty-equals/)（[GitHub](https://github.com/samuelcolvin/dirty-equals)）：`IsInt` や `IsNow` のような「ゆるい比較」をするためのライブラリ

どちらも実在する普通のPyPIパッケージですが、プロジェクト本体の依存関係には入っておらず、テストのときだけ必要になるタイプのものです。そのため、本体の依存だけを入れた環境では、**テストの収集段階でエラーになってしまいます**。

怖いのは、この状態でも評価自体は「最後まで動いてしまう」ことです。エラーで止まるわけではなく、単に「不正解」として数えられるだけなので、気づかないままローカルの評価結果が実際より低く出続けます。そして、その原因を「モデルの実力不足」と誤解してしまいます。

## gold checkとは

gold check は、**モデルを評価する前に、正解（gold / reference）の解答を当ててテストが全部通るかを確認する**作業のことです。

考え方はシンプルで、

- 正解を当ててもテストが通らないなら、そのタスクは**評価環境かタスク自体に問題がある**
- 正解で通ることが確認できたタスクだけが、**モデルの評価に使える**

ということです。テストを「ものさし」として使う以上、まずはものさしが正しく目盛りを刻めるかを確かめよう、というイメージです。

## やり方（コード例）

ここでは、各タスクが次のような構成になっていると仮定します。

```text
tasks/
  <task_id>/
    repo/        # 対象リポジトリ（git管理されている）
    gold.patch   # 正解の差分
```

タスクごとに作業用ディレクトリへコピーし、正解パッチを当てて pytest を実行、結果を分類するスクリプトです。

```python:gold_check.py
"""gold check: 正解パッチを当ててテストが通るかを確認する"""
import json
import re
import shutil
import subprocess
import sys
import tempfile
from pathlib import Path

TASKS_DIR = Path("tasks")  # tasks/<task_id>/{repo/, gold.patch}

MISSING_MODULE = re.compile(r"ModuleNotFoundError: No module named '([^']+)'")
IMPORT_ERROR = re.compile(r"ImportError: (.+)")


def run_gold(task_dir: Path) -> dict:
    with tempfile.TemporaryDirectory() as tmp:
        work = Path(tmp) / "repo"
        shutil.copytree(task_dir / "repo", work)

        # 1. 正解パッチを当てる
        apply = subprocess.run(
            ["git", "apply", str((task_dir / "gold.patch").resolve())],
            cwd=work, capture_output=True, text=True,
        )
        if apply.returncode != 0:
            return {"status": "patch_failed", "detail": apply.stderr.strip()}

        # 2. テストを実行する
        try:
            result = subprocess.run(
                [sys.executable, "-m", "pytest", "-q", "-p", "no:cacheprovider"],
                cwd=work, capture_output=True, text=True, timeout=300,
            )
        except subprocess.TimeoutExpired:
            return {"status": "timeout", "detail": ""}

    return classify(result.returncode, result.stdout + result.stderr)


def classify(returncode: int, output: str) -> dict:
    if returncode == 0:
        return {"status": "ok", "detail": ""}

    # 3. 環境エラーかどうかを先に判定する
    missing = sorted(set(MISSING_MODULE.findall(output)))
    if missing:
        return {"status": "env_missing_module", "detail": ", ".join(missing)}
    imp = IMPORT_ERROR.search(output)
    if imp:
        return {"status": "env_import_error", "detail": imp.group(1).strip()}
    if returncode == 5:
        return {"status": "no_tests_collected", "detail": ""}

    # 4. それ以外は「正解でも落ちる」本当の問題
    return {"status": "test_failed", "detail": output.strip().splitlines()[-1]}


def main() -> int:
    report = {}
    for task_dir in sorted(p for p in TASKS_DIR.iterdir() if p.is_dir()):
        report[task_dir.name] = run_gold(task_dir)
        print(f"{task_dir.name:30} {report[task_dir.name]['status']:20} "
              f"{report[task_dir.name]['detail']}")

    Path("gold_check_report.json").write_text(
        json.dumps(report, ensure_ascii=False, indent=2)
    )
    bad = [k for k, v in report.items() if v["status"] != "ok"]
    return 1 if bad else 0


if __name__ == "__main__":
    sys.exit(main())
```

ポイントは次の3つです。

- `sys.executable -m pytest` で、**評価に使うのと同じPython環境**でテストを動かす
- 元の `repo/` を汚さないよう、一時ディレクトリにコピーしてからパッチを当てる
- 1件でも `ok` 以外があれば終了コード `1` を返すので、CIでそのまま使える

手元で試すと、出力はこんな感じになります。

```text
t1_ok                          ok
t2_env                         env_missing_module   dirty_equals
t3_fail                        test_failed          1 failed in 0.01s
```

## 環境エラーと本当の失敗を分ける

gold check で一番大事なのは、落ちたタスクを**「環境のせい」と「本当に失敗している」に分けること**です。

### ModuleNotFoundError / ImportError を拾う

テストの import 時点で落ちている場合、pytest の出力には `ModuleNotFoundError: No module named '...'` が含まれます。これを正規表現で拾えば、「どのモジュールが足りないか」まで自動で分かります。足りないモジュールの一覧を集めれば、そのまま「インストールすべきものリスト」になります。

ひとつ注意点として、**import名とPyPIのパッケージ名は一致しないことがあります**。今回の例でも、import は `inline_snapshot` / `dirty_equals`（アンダースコア）ですが、インストールするときは `inline-snapshot` / `dirty-equals`（ハイフン）です。PyPIでは両者は同じものとして扱われますが、`import cv2` に対して `opencv-python` のように名前がまったく違う場合もあるので、一覧から機械的に `pip install` する前に確認しておくと安心です。

### pytest の終了コードも参考にする

pytest は[終了コード](https://docs.pytest.org/en/stable/reference/exit-codes.html)で状態を返してくれます。`0` は全件成功、`1` はテスト失敗、`2` は中断（収集時のエラーなど）、`5` はテストが1件も集まらなかった、という意味です。特に `5` は「テストのパス指定が間違っている」といった環境側の問題であることが多いので、分けて扱うと原因が追いやすくなります。

### 依存関係を固定する

足りなかったライブラリは、評価環境の `requirements.txt` などに明記してバージョンを固定しておきます。

```text:requirements-eval.txt
pytest==8.3.3
inline-snapshot==0.36.1
dirty-equals==0.11
```

※バージョンは一例です。実際には対象リポジトリのテストが前提としているバージョンに合わせてください。

「たまたま手元に入っていたから動いた」という状態をなくしておくと、別のマシンやCIでも同じ結果が再現できます。

### CIでモデル評価の前に回す

最後に、gold check を**モデル評価の前段**としてCIに組み込みます。

```yaml:.github/workflows/eval.yml
name: eval
on: [push]
jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements-eval.txt
      - name: gold check
        run: python gold_check.py
      - name: model eval
        run: python run_model_eval.py
```

gold check が失敗した時点でジョブが止まるので、「環境が壊れたままモデルを評価してしまう」ことを防げます。依存を追加したり、タスクを増やしたりしたときにも自動で確認されるので安心です。

## まとめ

- LLMのコードを自動テストで評価するときは、**まず正解を当ててテストが通るか（gold check）**を確認しましょう
- 正解でも落ちる原因は、テスト用ライブラリの入れ忘れのような**単純な環境の問題**であることがあります
- `ModuleNotFoundError` / `ImportError` を拾って、**環境エラーと本当の失敗を分けて**記録すると原因がすぐ分かります
- 依存は固定し、gold check は**CIでモデル評価の前に**回すのがおすすめです

評価結果が思ったより低いとき、つい「モデルが弱いのかな」と考えてしまいがちですが、その前に一度ものさしのほうを疑ってみてください。ここまで読んでいただきありがとうございました！
