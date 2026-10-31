---
title: "高いGPUを回す前に、安いCPUランタイムで「1タスクだけ」スモークテストしよう"
emoji: "🧯"
type: "tech"
topics: ["python", "testing", "pytest", "gpu", "mlops"]
published: true
---
<!-- planned slug: cheap-runtime-smoke-test -->

## はじめに

こんにちは、ZEN大学の通信課程で学んでいる2年生の松延です。AIエンジニアを目指して、ノートブックやクラウドのランタイムでLLMの出力やコードを評価する実験パイプラインを個人で回しています。

GPUランタイムは便利ですが、時間あたりのコストが高く、使えるコンピュートユニットにも上限があります。私はこれまでに「GPUを確保して本番ジョブを流したら、最初の数分で落ちていた」「落ちたことに気づかず、ランタイムだけが起動しっぱなしになっていた」という失敗を何度かしました。

原因をふり返ると、そのほとんどはGPUがなくても見つけられるものでした。この記事では、高いランタイムに上げる前に、安いCPUランタイムで最小限の確認をする「スモークテスト」の考え方と、すぐ使える小さなスクリプトを紹介します。

## 何が起きやすいか

LLMやコードを評価するパイプラインでは、次のようなトラブルがよく起きます。

- **テスト用の依存パッケージが入っていない**:生成したコードをpytestで採点する仕組みなのに、本番ランタイムにpytestやテスト対象が使うライブラリが入っていない。
- **キャッシュやデータが見つからない**:モデルの重みやデータセットのキャッシュを置いたつもりのパスが、新しいランタイムでは空になっている。そのため毎回ダウンロードが走ったり、オフライン設定のままエラーになったりします。
- **環境変数やパスの設定がずれている**:ローカルでは動いたのに、ランタイムを変えると作業ディレクトリやキャッシュの場所が変わっている。
- **最初のタスクでいきなり例外が出る**:出力ファイルの書き込み先がない、引数の名前を間違えている、といった単純なミスです。

どれもGPUの性能とは関係ありません。それでもGPUランタイム上で起きると、セットアップやモデルの読み込みにかかった時間ごと無駄になります。ジョブを何度もやり直すうちに、気づけばコンピュートユニットを使い切っていた、ということも起こりえます。

## スモークテストの考え方

ポイントはシンプルで、次の3つです。

1. **依存関係は本番と同じにする**:requirementsファイルやロックファイル、コンテナイメージなど、本番と同じ定義から環境を作ります。CPU用に別の依存リストを用意すると、確認したい差分そのものが見えなくなってしまいます(GPU専用のビルドが必要なパッケージだけは例外として分けて扱います)。
2. **安いランタイムで、1タスクだけ動かす**:評価セット全体ではなく1件だけ流します。CPUでは推論が遅くても、「最後まで通るか」を確かめられれば十分です。
3. **通ったら高いランタイムに昇格する**:スモークテストが成功したときだけ、同じコードと同じ設定のままGPUランタイムで本番を流します。

CIのパイプラインにたとえると、重いジョブの前に軽いジョブを置き、そこが通らなければ先に進まない、という段階的なゲートです。

## 最低限チェックする項目

私が毎回確認しているのは次の項目です。

- 主要なパッケージがimportできるか(評価用のpytestも含めます)
- キャッシュやデータのディレクトリが存在し、中身が空でないか
- 必要な環境変数が設定されているか
- 出力先のディレクトリに書き込めるか
- pytestのテストが1件だけでも実際に通るか
- 本番と同じエントリポイントを、件数を1に絞ったdry-runで最後まで実行できるか

全部をきっちりやる必要はありません。「過去に一度でもハマったもの」をリストに足していくだけでも、効果は十分にあります。

## 小さなスクリプト例

以下は、上のチェックをまとめたスクリプトです。`REQUIRED_MODULES` や `REQUIRED_PATHS`、`run_task` は自分のパイプラインに合わせて書き換えてください。

```python:smoke_test.py
#!/usr/bin/env python3
"""高いランタイムに上げる前に、安いランタイムで回すスモークテスト。"""
from __future__ import annotations

import argparse
import importlib
import json
import os
import subprocess
import sys
from pathlib import Path

# 本番と同じ依存スタックで確認したいモジュール
REQUIRED_MODULES = ["numpy", "pytest", "transformers"]
# 存在して中身があってほしいキャッシュやデータのパス
REQUIRED_PATHS = [
    Path(os.environ.get("HF_HOME", "~/.cache/huggingface")).expanduser(),
    Path("data"),
]
REQUIRED_ENV = ["HF_HOME"]


def check_imports(modules: list[str]) -> list[str]:
    errors = []
    for name in modules:
        try:
            importlib.import_module(name)
        except Exception as e:  # ImportError 以外の初期化エラーも拾う
            errors.append(f"import失敗: {name} ({type(e).__name__}: {e})")
    return errors


def check_paths(paths: list[Path]) -> list[str]:
    errors = []
    for p in paths:
        if not p.exists():
            errors.append(f"パスがない: {p}")
        elif p.is_dir() and not any(p.iterdir()):
            errors.append(f"ディレクトリが空: {p}")
    return errors


def check_env(names: list[str]) -> list[str]:
    return [f"環境変数が未設定: {n}" for n in names if not os.environ.get(n)]


def check_writable(out_dir: Path) -> list[str]:
    try:
        out_dir.mkdir(parents=True, exist_ok=True)
        probe = out_dir / ".smoke_write_test"
        probe.write_text("ok", encoding="utf-8")
        probe.unlink()
    except OSError as e:
        return [f"書き込めない: {out_dir} ({e})"]
    return []


def run_one_pytest(test_id: str) -> list[str]:
    # 例: tests/test_eval.py::test_single_case
    result = subprocess.run([sys.executable, "-m", "pytest", "-q", "-x", test_id])
    # 終了コード5は「テストが1件も集まらなかった」なので失敗扱いにする
    if result.returncode != 0:
        return [f"pytestが失敗: {test_id} (exit={result.returncode})"]
    return []


def load_tasks(path: Path) -> list[dict]:
    with path.open(encoding="utf-8") as f:
        return [json.loads(line) for line in f if line.strip()]


def run_task(task: dict) -> dict:
    # ここを本番と同じ処理(推論や採点)に置き換える
    return {"id": task.get("id"), "status": "ok"}


def main() -> int:
    parser = argparse.ArgumentParser(description="cheap runtime smoke test")
    parser.add_argument("--tasks", type=Path, help="1行1タスクのJSONLファイル")
    parser.add_argument("--out", type=Path, default=Path("outputs"))
    parser.add_argument("--pytest", metavar="TEST_ID", help="1件だけ実行するテストID")
    parser.add_argument("--dry-run", action="store_true", help="タスクを1件だけ実行する")
    args = parser.parse_args()

    errors: list[str] = []
    errors += check_imports(REQUIRED_MODULES)
    errors += check_paths(REQUIRED_PATHS)
    errors += check_env(REQUIRED_ENV)
    errors += check_writable(args.out)
    if args.pytest:
        errors += run_one_pytest(args.pytest)

    if errors:
        print("[NG] 事前チェックで問題が見つかりました:")
        for e in errors:
            print("  -", e)
        return 1

    if args.tasks:
        tasks = load_tasks(args.tasks)
        if args.dry_run:
            tasks = tasks[:1]
        for task in tasks:
            result = run_task(task)
            print(json.dumps(result, ensure_ascii=False))

    print("[OK] スモークテスト通過。高いランタイムに昇格してOKです。")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

安いCPUランタイムでは、次のように実行します。

```bash
python smoke_test.py --tasks data/tasks.jsonl --dry-run \
  --pytest tests/test_eval.py::test_single_case
```

終了コードが0のときだけ本番を流すようにしておくと、ノートブックでもCIでも同じように使えます。

```bash
python smoke_test.py --tasks data/tasks.jsonl --dry-run && python run_full_eval.py
```

## まとめ

- 高いGPUジョブが落ちる原因の多くは、依存パッケージ・キャッシュ・パス設定といった「GPUと関係ない部分」です。
- 本番と同じ依存スタックのまま、安いCPUランタイムで1タスクだけ通し、成功したら高いランタイムに昇格させます。
- チェック項目は、自分がハマったものを少しずつ足していけば十分です。

数分のスモークテストで、コンピュートユニットと気力の両方を守れます。同じように個人で実験を回している方の参考になればうれしいです。
