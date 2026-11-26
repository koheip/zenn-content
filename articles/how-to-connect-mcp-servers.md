---
title: "MCPサーバーのつなぎ方。Claude Code、Codex、Cursorでの追加と確認"
emoji: "🔗"
type: "tech"
topics: ["mcp", "claudecode", "codex", "cursor", "aidev"]
published: true
---

## はじめに

ZEN大学で知能情報を学んでいる2年生の松延です。AIエンジニアを目指して、個人開発を続けています。

日常の開発では、Windows 上で Claude Code、Codex CLI、Cursor を使っています。Issue やドキュメント、外部サービスをチャットに何度も貼る代わりに、MCP サーバーをつないで作業することが多いです。一方で、クライアントごとに設定ファイルの場所も、追加コマンドも違います。

この記事は、自分用の手順メモを兼ねて、MCP サーバーのつなぎ方だけをまとめます。コマンド名、設定ファイルの場所、フラグは公式ドキュメントで確認できた範囲に限っています。確認できなかったことは書いていません。設定例のトークン値は、すべてプレースホルダです。

## MCPとは何か

MCP（Model Context Protocol）は、AIアプリケーションと外部ツール・データ源をつなぐための公開されたプロトコルです。ホスト（Claude Code や Cursor などの AI アプリケーション）が、MCP サーバーごとにクライアント接続を持ち、サーバーが提供するツールやリソースを使えるようにします。

公式のアーキテクチャ解説では、次のように整理されています。

- **ホスト**: 接続をまとめる AI アプリケーション
- **クライアント**: 1つの MCP サーバーとの接続を保持する部品
- **サーバー**: ツールやコンテキストを提供するプログラム

「MCP サーバー」は、動く場所ではなく、コンテキストを提供するプログラムを指します。同じマシン上で起動するローカルサーバーも、クラウド上で公開されているリモートサーバーも、どちらも MCP サーバーです。

つなぐ理由はシンプルです。外部システムの情報をチャットに貼る代わりに、Claude や Codex、Cursor がそれらのシステムを直接使えるようにするためです。ただし、接続するサーバーは信頼できるものに限る必要があります。外部の内容を取得するサーバーは、プロンプトインジェクションのリスクもあります。

## つなぎ方の二種類：stdio とリモート（HTTP / OAuth）

公式の MCP 仕様が扱う主な輸送手段は、次の2つです。

| 種類 | 動き方 | 認証の考え方 |
| --- | --- | --- |
| **stdio** | ホストがローカルプロセスを起動し、標準入出力でやり取りする | OAuth の仕様は使わない。必要なキーは環境変数などから渡す |
| **Streamable HTTP** | HTTP でリモート（またはローカルに立てた HTTP エンドポイント）へ接続する | Bearer トークン、API キー、カスタムヘッダ、あるいは OAuth が使える。MCP は OAuth の利用を推奨している |

stdio は、ファイルシステムやローカルコマンドなど、自分のマシンに直接触れるツール向きです。リモート HTTP は、GitHub や Notion のように、サービス側が公開している MCP エンドポイント向きです。

補足が2つあります。

- 一部のクライアントは、古い **SSE** エンドポイントにも接続できます。Claude Code の公式ドキュメントでは、SSE 輸送は非推奨で、使えるなら HTTP を使うよう案内されています。
- リモートサーバーの OAuth は、HTTP ベースの輸送向けです。stdio サーバーに同じ OAuth フローを当てはめるのではなく、環境変数などから資格情報を取る、というのが仕様の考え方です。

サーバーへキーを渡すときも、設定ファイルやコマンド履歴に値を直書きしない方が安全です。Claude Code なら `--env` や `${API_KEY}`、Codex なら `env_vars` や `bearer_token_env_var`、Cursor なら `${env:API_KEY}` のように、環境変数名だけを設定に残します。

## Claude Code での追加方法

Claude Code では、基本は `claude mcp add` です。設定の保存先はスコープで変わります。

| スコープ | 読み込まれる範囲 | チームと共有 | 保存先 |
| --- | --- | --- | --- |
| **local**（デフォルト） | 今のプロジェクトだけ | しない | `~/.claude.json`（そのプロジェクトの項目の下） |
| **project** | 今のプロジェクトだけ | する（バージョン管理経由） | プロジェクトルートの `.mcp.json` |
| **user** | 自分の全プロジェクト | しない | `~/.claude.json` |

「local スコープ」の保存先はホームディレクトリの `~/.claude.json` です。プロジェクト内の `.claude/settings.local.json` とは別です。Windows では、公式ドキュメントが `~` と書いている場所は、ユーザーのホーム（プロファイル）ディレクトリ配下を指します。

### リモート HTTP サーバー

```bash
claude mcp add --transport http <name> <url>

# 例
claude mcp add --transport http notion https://mcp.notion.com/mcp

# Bearer トークンをヘッダで渡す例（値はプレースホルダ）
claude mcp add --transport http secure-api https://api.example.com/mcp --header "Authorization: Bearer YOUR_TOKEN"
```

JSON で書くときは、`type` に `http` を付けます。`streamable-http` は `http` の別名として受理されます。`url` があるのに `type` がない項目は、stdio サーバーとして読まれて失敗します。

SSE しかないサービスでは `--transport sse` を使います。Claude Code v2.1.265 以降は、`--transport http` で追加しても、相手が受け付けなければ SSE へ切り替える場合があります。ただし公式は、使えるなら HTTP を推奨しています。

### ローカル stdio サーバー

stdio では、Claude Code 側のオプションと、サーバー起動コマンドを `--` で区切ります。`--` の後ろは、そのままサーバー起動に渡されます。

```bash
claude mcp add --env AIRTABLE_API_KEY=YOUR_KEY --transport stdio airtable -- npx -y airtable-mcp-server
```

`--env` の直後にサーバー名を置くと、名前を別の `KEY=value` だと誤読します。`--transport stdio` など、別のオプションを間に置きます。

プロジェクト用の `.mcp.json` や `~/.claude.json` で `${VAR}` を使う場合、Claude Code は起動したサーバーの環境に `CLAUDE_PROJECT_DIR` を渡します。設定ファイル側でこの変数を展開したいときは、`${CLAUDE_PROJECT_DIR:-.}` のようにデフォルト値を付ける必要があります。

### スコープの指定

```bash
# 今のプロジェクトだけ、自分用（デフォルト）
claude mcp add --transport http stripe https://mcp.stripe.com

# チームで共有する .mcp.json に書く
claude mcp add --transport http shared-server --scope project https://example.com/mcp

# 自分の全プロジェクトで使う
claude mcp add --transport http hubspot --scope user https://mcp.hubspot.com/anthropic
```

同じサーバー名が複数スコープにあるときは、エントリ全体をマージせず、優先順位の高い定義だけを使います。優先順位は local → project → user です。

プロジェクトスコープの `.mcp.json` は、対話セッションでは承認が必要です。承認をやり直すときは `claude mcp reset-project-choices` です。クローンしたばかりのリポジトリでは、ワークスペースを信頼するまで、設定ファイルに書かれた承認だけではつながらないことがあります。

### `.mcp.json` で秘密情報を直書きしない

チーム共有の `.mcp.json` には、トークンそのものを書かず、環境変数を参照します。

```json
{
  "mcpServers": {
    "api-server": {
      "type": "http",
      "url": "${API_BASE_URL:-https://api.example.com}/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      }
    }
  }
}
```

`${VAR}` は環境変数 `VAR` の値、`${VAR:-default}` は未設定時に `default` を使います。空文字に設定した変数は「設定済み」と見なされ、デフォルト値は使われません。

注意点として、リモートサーバーの `url` や `headers` では、`ANTHROPIC_API_KEY` や `AWS_BEARER_TOKEN_BEDROCK`、`NPM_TOKEN` など、Claude Code が資格情報と見なす変数名は展開されず空になります。独自の名前の変数にコピーしてから参照します。

別クライアント向けの `mcpServers` JSON を流用するときは、`claude mcp add-json <name> '<エントリのオブジェクト>'` を使います。渡すのは外側の `mcpServers` ではなく、中のオブジェクトです。

## Codex CLI での追加方法

Codex の MCP 設定は `config.toml` にあります。既定はユーザー単位の `~/.codex/config.toml` です。プロジェクト単位にする場合は、信頼済みプロジェクトの `.codex/config.toml` も使えます。ChatGPT デスクトップアプリ、Codex CLI、IDE 拡張は、同じホスト設定を共有します。

### CLI で追加する

```bash
# stdio
codex mcp add <server-name> --env VAR1=VALUE1 --env VAR2=VALUE2 -- <stdio server-command>

# 例
codex mcp add context7 -- npx -y @upstash/context7-mcp

# リモート HTTP（OAuth クライアント ID を付ける例）
codex mcp add example --url https://mcp.example.com --oauth-client-id my-client
```

OAuth が必要なサーバーでは `codex mcp login <server-name>` を使います。ブラウザが使えない端末では `codex mcp login <server-name> --no-browser` を使い、表示された認可 URL を開いて、コールバックの URL をそのままターミナルへ貼ります。OAuth ログインは Streamable HTTP サーバー向けです。

### `config.toml` に書く

stdio サーバーの例です。キーの値は直書きせず、環境変数名を列挙して転送します。

```toml
[mcp_servers.context7]
command = "npx"
args = ["-y", "@upstash/context7-mcp"]
env_vars = ["LOCAL_TOKEN"]

[mcp_servers.context7.env]
MY_ENV_VAR = "MY_ENV_VALUE"
```

リモート（Streamable HTTP）の例です。Bearer トークンは値そのものではなく、環境変数名を指定します。

```toml
[mcp_servers.figma]
url = "https://mcp.figma.com/mcp"
bearer_token_env_var = "FIGMA_OAUTH_TOKEN"
http_headers = { "X-Figma-Region" = "us-east-1" }
```

よく使う項目は次のとおりです。

- stdio: `command`（必須）、`args`、`env`、`env_vars`、`cwd`
- HTTP: `url`（必須）、`bearer_token_env_var`、`http_headers`、`env_http_headers`、`auth`（`oauth` がデフォルト）
- 共通の任意項目: `enabled`、`startup_timeout_sec`（既定 10 秒）、`tool_timeout_sec`、`enabled_tools`、`disabled_tools`

OAuth のクライアントシークレットを `codex mcp add` に渡すと、`mcp_servers.<name>.oauth.client_secret` として `config.toml` に保存されます。このファイルは非公開にし、バージョン管理に入れません。

## Cursor での追加方法

Cursor では、Customize 画面から追加するか、`mcp.json` を手で書きます。対応している輸送は stdio、SSE、Streamable HTTP です。リモートは OAuth にも対応しています。

| 置き場所 | 用途 |
| --- | --- |
| プロジェクトの `.cursor/mcp.json` | そのプロジェクト用。リポジトリにコミットするとチームで共有できる |
| ホームの `~/.cursor/mcp.json` | エディタと CLI で開く全プロジェクト用 |

両方があるときはマージされ、同じサーバー名ならプロジェクト側が優先されます。ホストされた Cloud Agent は、マシン上の `~/.cursor/mcp.json` を読みません。

### stdio とリモートの書き方

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "mcp-server"],
      "env": {
        "API_KEY": "${env:API_KEY}"
      }
    }
  }
}
```

```json
{
  "mcpServers": {
    "server-name": {
      "url": "https://mcp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:MY_SERVICE_TOKEN}"
      }
    }
  }
}
```

`command`、`args`、`env`、`url`、`headers` では、`${env:NAME}` のような変数展開が使えます。秘密情報はファイルに直書きせず、シェルやシステムの環境変数を参照します。`envFile` は stdio サーバー専用で、リモート（HTTP/SSE）では使えません。

保存したあとは Cursor を再起動します。公式ドキュメントでも、秘密情報は環境変数に置き、権限は最小限にし、センシティブなサーバーはローカルの stdio で動かす、と案内されています。

## つながったかの確認方法

追加しただけでは、接続できたとは限りません。各クライアントで状態を確認します。

**Claude Code**

```bash
claude mcp list
claude mcp get notion
```

セッション内では `/mcp` でも確認できます。`claude mcp add` が `Added ...` と出しても、それは設定が書けたという意味です。接続の成否は、その後の状態で見ます。

`claude mcp list` には、実際に接続を試した結果として `✔ Connected`、`! Needs authentication`、`✘ Failed to connect` などが出ます。一方、次の表示は接続を試す前の設定状態です。

- `⏸ Pending approval (run claude to approve)`: プロジェクトの `.mcp.json` にあるサーバーで、まだ承認していない
- `⊘ Disabled for this project (re-enable via /mcp)`: このプロジェクトで無効になっている。`/mcp` から戻せる
- `✘ Rejected` やポリシーによるブロック: 設定や組織の方針で拒否されている

`⏸ Pending approval` と出たら、そのプロジェクトで `claude` を対話起動して、ワークスペースの信頼とサーバーの承認を済ませます。リモートサーバーが認証待ちなら、`/mcp` か `claude mcp login <name>` で OAuth を進めます。ブラウザが使えないときは `claude mcp login <name> --no-browser` です。

削除する前に、認可サーバー側でも取り消したい場合は、先に `claude mcp logout <name>` を実行します。`claude mcp remove` は Claude Code が保存したトークンを消しますが、発行元での取り消しまでは行いません。

**Codex CLI**

```bash
codex mcp list
codex mcp --help
```

TUI では `/mcp` で、今つながっているサーバーを確認できます。OAuth が必要なサーバーは、`codex mcp login <server-name>` のあとに、もう一度一覧を見て状態を確認します。

**Cursor**

設定を保存したあと、Cursor を再起動します。Customize 画面で、サーバーが有効か、OAuth のサインインが必要かを確認できます。一時的に無効化したいときも、この画面のトグルを使います。うまくつながらないときは、Output パネルで「MCP Logs」を選び、接続エラーや認証失敗、サーバーの異常終了を見ます。

## ハマりどころ

よく出会うのは、次のようなずれです。

**設定はあるのに接続できない**

- Claude Code の `.mcp.json` で、`url` があるのに `type` が無い。stdio サーバーとして読まれて失敗する
- プロジェクトスコープのサーバーが未承認（`⏸ Pending approval`）
- Cursor の `mcp.json` を保存したあとに再起動していない
- stdio サーバーの起動コマンド（`npx` や `python`）が、その環境の PATH に無い
- 環境変数を今のセッションにだけ設定して、エージェントや別ウィンドウからは見えていない。Windows の PowerShell では、`$Env:API_KEY = "YOUR_KEY"` は現在のプロセス限りです
- Claude Code のリモート設定で、予約された資格情報変数名を `${ANTHROPIC_API_KEY}` のように参照してしまい、空文字になっている
- 同じサーバー名を複数スコープに置いて、意図しない方の定義が使われている。Claude Code では `claude mcp list` や `/mcp` に競合の警告が出ます

**キーの渡し方**

サーバーが必要とするキーは、コミットする設定ファイルに直書きしません。Claude Code の `.mcp.json` では `${API_KEY}`、Cursor では `${env:API_KEY}`、Codex では `bearer_token_env_var` や `env_vars` を使い、値は各環境の環境変数に置きます。コマンドラインの引数に値を書くと、履歴に残ることがあります。

**Windows でつまずきやすい点**

公式ドキュメントの多くは bash 前提です。PowerShell では `export API_KEY=...` ではなく `$Env:API_KEY = "..."` を使います。セッション限定の設定を、別のターミナルで動く Claude Code や Cursor が読めない、というずれが起きやすいです。

ホーム配下の設定ファイル（`~/.claude.json`、`~/.codex/config.toml`、`~/.cursor/mcp.json`）は、Windows ではユーザープロファイルの下にあります。パスを手で書くときは、今の環境のホームを確認してからにします。

stdio の MCP サーバーは、ホストがコマンドを直接起動します。そのターミナルでは `npx` が使えるのにサーバーだけ起動失敗するときは、エージェント側から見た PATH を疑います。ここは環境差が出やすいので、各クライアントのログ（Claude Code なら `/mcp` や `claude mcp get`、Cursor なら MCP Logs、Codex なら `codex mcp list`）で原因を見ます。

## 参考にした公式ドキュメント

- [MCP Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture)
- [MCP Authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)
- [Connect Claude Code to tools via MCP](https://code.claude.com/docs/en/mcp)
- [Codex: Model Context Protocol](https://learn.chatgpt.com/docs/extend/mcp)
- [Codex configuration reference](https://learn.chatgpt.com/docs/config-file/config-reference)
- [Cursor: Model Context Protocol (MCP)](https://cursor.com/docs/mcp)

## おわりに

MCP サーバーのつなぎ方で毎回確認しているのは、輸送（stdio か HTTP か）、スコープ（今のプロジェクトだけか、全プロジェクトか）、そして本当につながったかの確認です。Windows で Claude Code、Codex、Cursor を併用していると、設定ファイルの場所と、環境変数が見える範囲を取り違えることが多いので、そこだけは先に見るようにしています。

手順に誤りや更新があれば、公式ドキュメントを優先してください。
