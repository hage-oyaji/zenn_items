---
title: "Claude Managed Agents 調査レポート ─ 永続自走エージェントを支える Memory / Multiagent / Outcomes / Dreaming / Webhooks / Vault"
emoji: "📘"
type: "tech"
topics: ["claude", "anthropic", "ai", "agent", "managedagents"]
published: false
---

# Claude Managed Agents 調査レポート ─ 永続自走エージェント時代の機能群

> 調査日: 2026-05-11 / 情報源: 公式サイト(Anthropic / Claude Platform Docs)・技術記事(Qiita / Zenn / note / Dev.to 他)・Q&A(Hacker News / GitHub Issues 等)・GitHub(公式 SDK / Cookbook)

「**永続自走エージェント Memory Multiagent Outcomes Dreaming WebHooks Vault**」というキーワード群は、Anthropic が 2026 年 4 月 8 日にパブリックベータで公開し、5 月 6 日の **Code with Claude 2026** で機能拡張した **Claude Managed Agents** の中核機能を指しています。本レポートはその全体像を、概要・用途・解決できる問題・セットアップ・使い方の 5 観点で日本語に整理したものです。

| 項目 | 値 |
| --- | --- |
| 種別 | クラウドマネージドのエージェント実行基盤(エージェントハーネス + サンドボックス + 状態管理 + ツール実行を一体提供) |
| 開発元 | Anthropic |
| ライセンス / 提供形態 | マネージドサービス(従量課金)。Memory / Multiagent / Outcomes / Webhooks は **Public Beta**、Dreaming は **Research Preview**(申請制)。Vault は Public Beta。 |
| API バージョン | `anthropic-beta: managed-agents-2026-04-01`(全 API)/ `dreaming-2026-04-21`(Dreams 追加) |
| 料金(本稿時点) | Opus 4.6: 入力 $5 / 出力 $25(per MTok)、Sonnet 4.6: $3 / $15。さらに **アクティブランタイム $0.08 / session-hour**。組み込み Web 検索は $10 / 1000 検索。Batch API 割引は対象外。(出典: Finout / WaveSpeed) |
| 公式アナウンス | [Claude Managed Agents: get to production 10x faster](https://claude.com/blog/claude-managed-agents)(2026-04-08)/ [New in Claude Managed Agents: dreaming, outcomes, and multiagent orchestration](https://claude.com/blog/new-in-claude-managed-agents)(2026-05-06) |
| 公式ドキュメント | [platform.claude.com/docs/en/managed-agents/overview](https://platform.claude.com/docs/en/managed-agents/overview) |
| 公式 SDK(GitHub) | [anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)(★6,790 / MIT / 最新 v0.1.80, 2026-05-09)/ [anthropics/claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript) |

## 概要

Claude Managed Agents は、Anthropic 自身が「**Claude (脳) と実行ハーネス (身体) を分離した、マネージド型のエージェント実行ランタイム**」と説明するサービスです。Messages API が「単発の LLM 呼び出し」を提供するのに対し、Managed Agents は「長時間自走・複数ツール連携・並列実行・耐障害」を備えた **エージェントの実行基盤** を提供します。アーキテクチャは `Agent`(モデル + プロンプト + ツール + MCP + スキル)/ `Environment`(クラウドコンテナ)/ `Session`(実行インスタンス)/ `Events`(状態更新の最小単位)の **4 プリミティブ** で構成されています(出典: 公式 Overview, npaka 解説)。

内部設計は **`Session`(追記専用イベントログ)/ `Harness`(ステートレスループ)/ `Sandbox`(隔離コンテナ)** の 3 層で、コンテナがクラッシュしてもイベントログのリプレイで状態を復元できる耐障害設計になっています(出典: Qiita kai_kou 解説)。

2026 年 5 月 6 日の発表で、永続記憶を司る **Memory**、自己改善型の **Dreaming**、品質ゲートとなる **Outcomes**、リードと専門家を分業させる **Multiagent Orchestration**、長時間セッション向けの **Webhooks**、エンドユーザー単位の認証情報管理を行う **Vault** が一斉に拡充され、「永続的に走り続けて改善し続けるエージェント」という構想が実用域に達しました。導入企業として Harvey(法務)、Netflix、Rakuten、Notion、Wisedocs、Spiral などが公式ブログで紹介されています。

なお全 API 呼び出しに beta ヘッダ `anthropic-beta: managed-agents-2026-04-01` が必須で、Dreams のみ追加で `dreaming-2026-04-21` を付与します。SDK は自動付与しますが、直接 HTTP を叩くケースでは付け忘れに注意が必要です(出典: WaveSpeed Blog)。

## 用途

- **長時間自走する業務エージェント** — 数分〜数時間に渡る法務ドキュメント作成、月次決算、DCF モデリング、KYC 審査、ピッチブック生成など(出典: Harvey / Wisedocs / Anthropic 公式)。
- **大規模並列分析** — Netflix がプラットフォームチームのビルドログを並列解析する用途で Multiagent Orchestration を本番デプロイ(出典: 公式ブログ / BuildFastWithAI)。
- **コードレビュー・自動テスト生成** — Sentry がバグ検出 + パッチ作成、Spiral がリード(Haiku)→ サブ(Opus)へドラフト並列委譲(出典: 公式ブログ)。
- **CS / サポートエージェント** — Galirage のチュートリアル例では Slack ↔ Notion をつないだサポートエージェントを Console ウィザードでノーコード構築(出典: Zenn)。
- **GitHub 連携の PR レビュー自動化** — 個人開発者が「GitHub Webhook → Cloudflare Workers → Managed Agents → Discord 通知」の構成を実装(出典: Zenn kumamo_tone)。
- **HITL(Human-in-the-Loop)非同期ワークフロー** — Webhook の `session.status_idled` をトリガーに人間レビュー → `user.custom_tool_result` を POST で復帰、というプロセス再起動に耐える構造(出典: GitHub anthropics/claude-cookbooks #539)。
- **業務システムの「OS 層」化** — レイヤ分離(OS 層 = Managed Agents、ナレッジ層 = 自社管理)を意識した朝のブリーフィングや月次経理自動化(出典: Zenn GENDA)。

## 解決できる問題

| 課題 | 解決 |
| --- | --- |
| **エージェントループ・サンドボックス・状態管理を自前で組むと数ヶ月かかる** | Managed Agents が `Agent / Environment / Session / Events` の **3 API 呼び出し** で起動できる(Rakuten は部署横断展開を各 1 週間で完了)。 |
| **セッション間でコンテキストが消える** | **Memory**(`/mnt/memory/` にマウントされる永続ファイルシステム)で嗜好・規約・過去の失敗を保存。`memory_version` で監査と point-in-time 復元も可能。 |
| **エージェントが「同じミスを繰り返す」** | **Dreaming** が過去セッションとメモリストアを横断分析し、重複・矛盾・古いエントリを自動キュレート(Harvey でタスク完了率 約 6 倍に改善)。 |
| **LLM 自身に「終わったかどうか」を判定させると主観に引きずられる** | **Outcomes** は独立した context window の grader が Markdown rubric で採点し、`needs_revision` で自己修正ループ。社内検証で docx +8.4pt / pptx +10.1pt の成功率向上。 |
| **単一エージェントの並列化限界** | **Multiagent Orchestration** で coordinator が最大 20 種類の specialist を 25 並列スレッドまでファンアウト(共有ファイルシステム経由で結果統合)。 |
| **長時間セッションを SSE で張り続けるコスト・切断耐性** | **Webhooks** で `session.status_idled` / `outcome_evaluation_ended` などの状態変化のみを HTTPS 通知(5 分のリプレイ保護つき)。 |
| **エンドユーザー毎の OAuth トークン管理が煩雑** | **Vault** が `mcp_oauth` でリフレッシュを自動代行。自前 secret store 不要、認証情報は sandbox のアドレス空間に入らないためプロンプトインジェクション耐性も向上(出典: Pluto Security)。 |

## セットアップ方法

### 前提条件

- **Anthropic Console アカウント + API キー**(Managed Agents は全 API アカウントでデフォルト有効)
- **Dreaming のみアクセス申請制** — `https://claude.com/form/claude-managed-agents` から申請
- レート制限: create 系 300 req/min、read 系 600 req/min(org 単位)

### CLI / SDK のインストール

```bash
# Anthropic CLI (Homebrew)
brew install anthropics/tap/ant

# 各言語の SDK
pip install anthropic                       # Python
npm install @anthropic-ai/sdk               # TypeScript / Node.js
go get github.com/anthropics/anthropic-sdk-go
dotnet add package Anthropic                # .NET
bundle add anthropic                        # Ruby
composer require anthropic-ai/sdk           # PHP

# Claude Code CLI を内包する高レベル SDK (推奨)
pip install claude-agent-sdk
```

### 環境変数

```bash
export ANTHROPIC_API_KEY="your-api-key-here"
# Webhook を使う場合
export ANTHROPIC_WEBHOOK_SIGNING_KEY="whsec_..."   # Console > Settings > Webhooks で発行(初回のみ表示)
```

### beta ヘッダ

```text
anthropic-beta: managed-agents-2026-04-01
# Dreaming を併用する場合
anthropic-beta: managed-agents-2026-04-01,dreaming-2026-04-21
```

> 公式 SDK は beta ヘッダを自動付与します。直接 HTTP を叩く場合は付け忘れに注意してください(出典: WaveSpeed Blog)。

## 使い方

### 最小サンプル(Python・Quickstart 相当)

`agent.create` → `environments.create` → `sessions.create` → `events.stream` で SSE 受信しながら `events.send` でメッセージを送信します。

```python
from anthropic import Anthropic

client = Anthropic()

agent = client.beta.agents.create(
    name="Coding Assistant",
    model="claude-opus-4-7",
    system="You are a helpful coding assistant. Write clean, well-documented code.",
    tools=[
        {"type": "agent_toolset_20260401"},
    ],
)

environment = client.beta.environments.create(
    name="quickstart-env",
    config={
        "type": "cloud",
        "networking": {"type": "unrestricted"},
    },
)

session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    title="Quickstart session",
)

with client.beta.sessions.events.stream(session.id) as stream:
    client.beta.sessions.events.send(
        session.id,
        events=[
            {
                "type": "user.message",
                "content": [
                    {"type": "text",
                     "text": "Create a Python script that generates the first 20 Fibonacci numbers and saves them to fibonacci.txt"},
                ],
            },
        ],
    )

    for event in stream:
        match event.type:
            case "agent.message":
                for block in event.content:
                    print(block.text, end="")
            case "agent.tool_use":
                print(f"\n[Using tool: {event.name}]")
            case "session.status_idle":
                print("\n\nAgent finished.")
                break
```

> `agent_toolset_20260401` を有効にすると **bash / read / write / edit / glob / grep / web_fetch / web_search** の 8 ビルトインツールが一括有効化されます(出典: Zenn kumamo_tone)。

### Memory(永続記憶)

`memory_store` を作成し、セッション作成時の `resources` でアタッチ(コンテナ内 `/mnt/memory/` にマウント)。**セッション作成後の追加・削除は不可** なので、ユーザー別ストアは事前に用意します。

```python
store = client.beta.memory_stores.create(
    name="User Preferences",
    description="Per-user preferences and project context.",
)

client.beta.memory_stores.memories.create(
    store.id,
    path="/formatting_standards.md",
    content="All reports use GAAP formatting. Dates are ISO-8601...",
)

session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    resources=[
        {
            "type": "memory_store",
            "memory_store_id": store.id,
            "access": "read_write",
            "instructions": "User preferences and project context. Check before starting any task.",
        }
    ],
)
```

### Outcomes(rubric ベースの自己評価ループ)

セッション作成後に `user.define_outcome` イベントを送ると、独立 grader が rubric で採点しエージェントが自己修正ループに入ります。

```python
client.beta.sessions.events.send(
    session_id=session.id,
    events=[
        {
            "type": "user.define_outcome",
            "description": "Build a DCF model for Costco in .xlsx",
            "rubric": {"type": "text", "content": RUBRIC},
            # or: "rubric": {"type": "file", "file_id": rubric.id},
            "max_iterations": 5,  # optional; default 3, max 20
        }
    ],
)

session = client.beta.sessions.retrieve(session.id)
for outcome in session.outcome_evaluations:
    print(f"{outcome.outcome_id}: {outcome.result}")
    # 例: outc_01a...: satisfied
```

### Multiagent Orchestration(coordinator + specialist)

`multiagent.type=coordinator` でロスター(最大 20 種、25 並列スレッド、depth=1 のみ)を宣言します。

```python
coordinator = client.beta.agents.create(
    name="Engineering Lead",
    model="claude-opus-4-7",
    system="You coordinate engineering work. Delegate code review to the reviewer agent and test writing to the test agent.",
    tools=[{"type": "agent_toolset_20260401"}],
    multiagent={
        "type": "coordinator",
        "agents": [
            {"type": "agent", "id": reviewer_agent.id},
            {"type": "agent", "id": test_writer_agent.id},
        ],
    },
)

session = client.beta.sessions.create(
    agent=coordinator.id,
    environment_id=environment.id,
)

for thread in client.beta.sessions.threads.list(session.id):
    print(f"[{thread.agent.name}] {thread.status}")
```

### Dreaming(非同期メモリキュレーション・Research Preview)

`dreams.create` で過去最大 100 セッション + 既存メモリストアを入力にし、新しい再編成済みストアを生成します。**入力ストアは不変、出力は別ストア** として返ります。

```python
dream = client.beta.dreams.create(
    inputs=[
        {"type": "memory_store", "memory_store_id": store_id},
        {"type": "sessions", "session_ids": [session_a, session_b]},
    ],
    model="claude-opus-4-7",
    instructions="Focus on coding-style preferences; ignore one-off debugging notes.",
)

while dream.status in ("pending", "running"):
    time.sleep(10)
    dream = client.beta.dreams.retrieve(dream.id)

output_store_id = next(
    o.memory_store_id for o in dream.outputs if o.type == "memory_store"
)

session = client.beta.sessions.create(
    agent=agent_id,
    environment_id=environment_id,
    resources=[{"type": "memory_store", "memory_store_id": output_store_id}],
)
```

### Webhooks(Flask 受信例)

`webhooks.unwrap()` で `X-Webhook-Signature` を検証し、5 分以上前のペイロードは自動で拒否されます。

```python
from flask import Flask, request
import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_WEBHOOK_SIGNING_KEY from env
app = Flask(__name__)


@app.route("/webhook", methods=["POST"])
def webhook():
    try:
        event = client.beta.webhooks.unwrap(
            request.get_data(as_text=True),
            headers=dict(request.headers),
        )
    except Exception:
        return "invalid signature", 400

    if event.data.type == "session.status_idled":
        print("session idled:", event.data.id)
    # 他に session.status_run_started / session.outcome_evaluation_ended /
    # vault_credential.refresh_failed などをハンドル

    return "", 200
```

サポートイベントは `session.status_run_started` / `session.status_idled` / `session.status_terminated` / `session.outcome_evaluation_ended` / `vault.*` / `vault_credential.refresh_failed` 等。**少なくとも 1 回配信(at-least-once)で順序非保証** のため、`event.id` での冪等化と必要に応じた `created_at` ソートを推奨(出典: Hookdeck)。

### Vault(エンドユーザー単位の認証情報)

`vault` を作成 → `mcp_oauth` または `static_bearer` の credential を登録 → セッション作成時に `vault_ids` で参照。Anthropic 側が OAuth リフレッシュを代行します。

```python
vault = client.beta.vaults.create(
    display_name="Alice",
    metadata={"external_user_id": "usr_abc123"},
)

client.beta.vaults.credentials.create(
    vault_id=vault.id,
    display_name="Alice's Slack",
    auth={
        "type": "mcp_oauth",
        "mcp_server_url": "https://mcp.slack.com/mcp",
        "access_token": "xoxp-...",
        "expires_at": "2099-12-31T23:59:59Z",
        "refresh": {
            "token_endpoint": "https://slack.com/api/oauth.v2.access",
            "client_id": "1234567890.0987654321",
            "scope": "channels:read chat:write",
            "refresh_token": "xoxe-1-...",
            "token_endpoint_auth": {"type": "client_secret_post", "client_secret": "abc123..."},
        },
    },
)

client.beta.vaults.credentials.create(
    vault_id=vault.id,
    display_name="Linear API key",
    auth={
        "type": "static_bearer",
        "mcp_server_url": "https://mcp.linear.app/mcp",
        "token": "lin_api_your_linear_key",
    },
)

session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    vault_ids=[vault.id],
    title="Alice's Slack digest",
)
```

## 注意点・つまずきポイント

実ユーザーから報告されている代表的な落とし穴です(出典: Hacker News / Pluto Security / hi120ki / GitHub Issues / 各種レビュー記事)。

- **料金事故**: セッションを `stop` しないと **$0.08 / session-hour** が回り続けます。アイドル状態自体は課金されませんが、停止忘れがコスト事故の典型(出典: Qiita ny7760 / Finout)。
- **ベンダーロックイン**: Bedrock / Vertex AI 経由では使えず、Claude Platform 直アクセスが必須。マルチクラウド要件のある組織は導入不可(出典: Hacker News)。
- **Vault の "Confused Deputy" 問題**: Vault はワークスペース単位で共有されるため、コンソールアクセス権を持つ誰でも他人の Vault を参照したセッションを立ち上げられます。**コンソールアクセスは管理者限定にし、エンドユーザー向けには別 UI を用意** する運用が必須(出典: hi120ki)。
- **OAuth スコープを Vault 側で絞れない**: 広い権限のトークンを保管するとそのまま実行されます。スコープ制限は OAuth クライアント発行段階で行うしかありません(出典: 公式 Docs)。
- **RBAC 欠落**: 1 つの API キーがワークスペース内の全エージェント・全セッション・全イベントログにアクセス可能。API キーは production DB と同等の機密として扱う必要があります(出典: Pluto Security)。
- **Memory アタッチタイミング制約**: `memory_store` はセッション作成時のみ。動的切替えはセッション再作成が必要です。
- **Multiagent の制約**: depth=1(再帰的分解不可)、ロスター 20、並列 25。これを超える設計は事前に分解を検討。
- **Outcomes の "Rubric Theater"**: 「polished, excellent に」のような曖昧な rubric は grader が何でも合格させてしまいます。**観測可能な基準・明示的制約・既知の失敗モードを列挙した rubric** を必ず書くこと(出典: 公式 Cookbook / kenhuangus)。
- **Webhook 検証バグ**: パース後 JSON で署名検証すると失敗します(バイト列が変わるため必ず raw body で検証)。タイミングセーフ比較必須、`whsec_` は初回のみ表示なので即 secrets manager へ(出典: Hookdeck)。
- **デフォルト権限が広すぎる**: 8 ビルトインツール全許可 + ネット制限なし + 自動実行。ハードニングとしてネットワーク制限、不要ツール無効化、`bash` / `write` の `always_ask` 化を推奨(出典: Pluto Security)。
- **カスタム MCP はサンドボックス保護外**: カスタム MCP ツールは自社アプリ側で実行されるため、Anthropic 側のネット制御・Vault 保護は及びません(出典: Pluto Security)。
- **Dreaming のメモリ汚染リスク**: 過去 transcript を横断的に読んで新メモリを書き戻すため、悪性入力が後続セッションへ伝搬する攻撃面があります。**自動反映ではなくレビュー付き反映を推奨**(出典: kenhuangus)。
- **セッション識別子が外部 MCP に見えない**: 外部 MCP サーバ側からは「API キーを持つ何かが来た」しか分からず、A→B→C 委譲の監査連鎖が崩れます。セッションスコープ JWT 等の対応が要望されています(出典: dev.to piiiico)。
- **サイレントツールループ**: 「ツールを呼び続けて最終メッセージが出ない」は `requires_action` への未応答が大半。session trace で `requires_action` の有無を確認(出典: WaveSpeed)。

## 参考リンク

### 公式
- [Claude Managed Agents: get to production 10x faster](https://claude.com/blog/claude-managed-agents) ─ ローンチブログ(2026-04-08)
- [New in Claude Managed Agents: dreaming, outcomes, and multiagent orchestration](https://claude.com/blog/new-in-claude-managed-agents) ─ 5 月新機能発表(2026-05-06)
- [Scaling Managed Agents: Decoupling the brain from the body](https://www.anthropic.com/engineering/managed-agents) ─ Anthropic Engineering
- [Built-in memory for Claude Managed Agents](https://claude.com/blog/claude-managed-agents-memory)
- [Managed Agents Overview (Docs)](https://platform.claude.com/docs/en/managed-agents/overview)
- [Quickstart](https://platform.claude.com/docs/en/managed-agents/quickstart)
- [Using agent memory](https://platform.claude.com/docs/en/managed-agents/memory)
- [Dreams (Research Preview)](https://platform.claude.com/docs/en/managed-agents/dreams)
- [Define outcomes](https://platform.claude.com/docs/en/managed-agents/define-outcomes)
- [Multiagent sessions](https://platform.claude.com/docs/en/managed-agents/multi-agent)
- [Subscribe to webhooks](https://platform.claude.com/docs/en/managed-agents/webhooks)
- [Authenticate with vaults](https://platform.claude.com/docs/en/managed-agents/vaults)
- [Outcomes: agents that verify their own work (Cookbook)](https://platform.claude.com/cookbook/managed-agents-cma-verify-with-outcome-grader)
- [Dreaming アクセス申請フォーム](https://claude.com/form/claude-managed-agents)

### GitHub
- [anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python) ─ ★6,790 / MIT / 最新 v0.1.80(2026-05-09)
- [anthropics/claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript)
- [anthropics/claude-agent-sdk-demos](https://github.com/anthropics/claude-agent-sdk-demos) ─ Email / Research / Excel デモ集
- [anthropics/cwc-long-running-agents](https://github.com/anthropics/cwc-long-running-agents) ─ Code with Claude 2026 長時間エージェントハーネス
- [anthropics/claude-cookbooks](https://github.com/anthropics/claude-cookbooks) ─ Cookbook + Issue Tracker
- [anthropics/claude-quickstarts](https://github.com/anthropics/claude-quickstarts) ─ autonomous-coding ほか
- [Doris26/claude-managed-agents-reference](https://github.com/Doris26/claude-managed-agents-reference) ─ 全機能の網羅的サードパーティリファレンス

### 技術記事
- [Claude Managed Agents を試す](https://note.com/npaka/n/n190b39339bf9) ─ npaka / 2026-05-07
- [【衝撃】AIが「夢を見て」自己進化する時代が来た!Claude Dreamingで完了率6倍の衝撃](https://qiita.com/emi_ndk/items/4fb1f4760d82d26728fc) ─ emi_ndk / 2026-05-08
- [Claude Managed Agents 入門【概要と使い方】](https://zenn.dev/galirage/articles/claude-managed-agents-quickstart) ─ Masumi Morishige (Galirage) / 2026-04-10
- [Claude Managed Agents 完全解説](https://note.com/ai_driven/n/n11978a041fa3) ─ アイドリ (AI-Driven Lab) / 2026-04-10
- [大規模にエージェントを構築する Claude Managed Agents を試してみた](https://azukiazusa.dev/blog/claude-managed-agents/) ─ azukiazusa / 2026-04-09
- [Claude Managed Agents 入門 — 3 層アーキテクチャと Python 実装ガイド](https://qiita.com/kai_kou/items/f93f9e95c357edaa51b6) ─ kai_kou / 2026-04-30
- [Claude Managed Agents を触ってみる](https://qiita.com/ny7760/items/07af9d3facaf4af3d9f2) ─ ny7760 / 2026-04-09
- [Claude Managed Agents を試してみた(GitHub Webhook → Cloudflare Workers → Discord)](https://zenn.dev/kumamo_tone/articles/365845d65e6cf4) ─ Kazumasa KUMAMOTO / 2026-04-11
- [Anthropic shipped webhooks for Claude Managed Agents. Here's what they unlock.](https://hookdeck.com/blog/anthropic-managed-agent-webhooks) ─ Gareth Wilson (Hookdeck) / 2026-05-07
- [Claude Managed Agents Dreaming Explained (2026)](https://www.buildfastwithai.com/blogs/claude-managed-agents-dreaming-explained) ─ BuildFastWithAI / 2026-05-07
- [Securing Claude Managed Agents](https://pluto.security/blog/securing-claude-managed-agents/) ─ Yotam Perkal (Pluto Security) / 2026-04-17
- [Claude Managed Agents で消える層、残る層: 業務自動化エージェントの視点から](https://zenn.dev/genda_jp/articles/8038f227ba9bdf) ─ ikenyal (GENDA) / 2026-05-05
- [Claude Managed Agents のアーキテクチャと既存エージェント実行環境との技術比較](https://qiita.com/tatematsu-k/items/dce3ce5252e24c4585bd) ─ tatematsu-k / 2026-04-09

### Q&A・議論
- [Claude Managed Agents (launch) | Hacker News](https://news.ycombinator.com/item?id=47693047) ─ ローンチ直後の主要スレ
- [Claude Managed Agents Overview | Hacker News](https://news.ycombinator.com/item?id=47697641)
- [Agents are now "dreaming" in Claude Managed Agents | Hacker News](https://news.ycombinator.com/item?id=48039860) ─ Dreaming 発表直後
- [[QUESTION] Managed agents webhooks · Issue #539](https://github.com/anthropics/claude-cookbooks/issues/539) ─ Console UI 不在問題
- [Claude Managed Agents Ships Without Session Identity](https://dev.to/piiiico/claude-managed-agents-ships-without-session-identity-3p56)
- [How Secure Are Claude Managed Agents?](https://hi120ki.github.io/blog/posts/20260413/) ─ Vault Confused Deputy 問題
- [Anthropic's Managed Agents: What's Missing](https://www.arcade.dev/blog/anthropic-managed-agents-missing-hands)
- [Claude Agents Can Now Dream: How AI Engineers Should Use Anthropic's New Agent Features Without Creating New Attack Paths](https://kenhuangus.substack.com/p/claude-agents-can-now-dream-how-ai)
- [Claude Managed Agents Review: Is It Worth It? (2026)](https://www.buildfastwithai.com/blogs/claude-managed-agents-review-2026)
- [Anthropic Just Launched Managed Agents. Let's Talk About How We're Going to Pay for This](https://www.finout.io/blog/anthropic-just-launched-managed-agents.-lets-talk-about-how-were-going-to-pay-for-this)
- [2026 AI Agent Framework Showdown: Claude Agent SDK vs Strands vs LangGraph vs OpenAI Agents SDK](https://qubittool.com/blog/ai-agent-framework-comparison-2026)
- [Claude Managed Agents Pricing and Beta Limits](https://wavespeed.ai/blog/posts/claude-managed-agents-pricing-2026/)

---
*このレポートは 2026-05-11 に Claude Code Agent Teams(Team Lead + 4 並列リサーチャー)によって自動生成されました。情報は調査時点のものです。最新情報は[公式サイト](https://claude.com/blog/new-in-claude-managed-agents)・[公式ドキュメント](https://platform.claude.com/docs/en/managed-agents/overview)をご確認ください。*
