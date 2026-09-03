---
title: "Claude Code コマンド完全リファレンス(v2.1.259 実機抽出版)"
emoji: "🧭"
type: "tech"
topics: ["claudecode", "cli", "ai", "生成AI", "開発環境"]
published: false
---

# Claude Code コマンド完全リファレンス

> **基準日：2026年9月4日**
> **対象バージョン：Claude Code 2.1.259(native build / Windows)**
> **出典：`claude --help` および各サブコマンドの `--help`、ならびに実行バイナリ内のコマンド定義を機械抽出**

Claude Code のコマンドは、公式ドキュメントに載っているものだけではない。実際のバイナリには **140以上のスラッシュコマンド** が登録されており、そのうち `/help` に出てこない隠しコマンドや、プラン・フラグ次第で有効化されるものも多い。

この記事は、その全量を **実機から抽出して分類したリファレンス** である。

---

## まず結論：コマンドは3レイヤーに分かれる

| レイヤー | 呼び出し方 | 数 | 何をするものか |
| --- | --- | --- | --- |
| **CLIコマンド** | `claude <command>` | 19 | セッションの外側。インストール、認証、MCP/プラグイン管理、バックグラウンド制御 |
| **CLIフラグ** | `claude --<flag>` | 60以上 | セッション起動時の挙動。モデル、権限、出力形式、ワークツリー |
| **スラッシュコマンド** | セッション内で `/<name>` | 140以上 | セッションの中身。コンテキスト、設定、拡張、ワークフロー |

さらにスラッシュコマンドは内部的に3種類ある。

| 種別 | 実体 | 例 |
| --- | --- | --- |
| `local` | 即時実行されるロジック | `/clear` `/compact` `/model` |
| `local-jsx` | 対話UIを開くもの | `/config` `/permissions` `/resume` |
| `prompt` / skill | Claude に指示を流し込むもの | `/init` `/code-review` `/commit` |

**skill 型が重要**である。これらは「Claude への定型指示のパッケージ」であり、ユーザーが `/` で呼ぶだけでなく、Claude 自身が必要と判断して自動的に読み込むこともある。

---

## 1. セッション操作・会話管理

対話そのものを切ったり繋いだり分岐させたりする、最も使用頻度の高い群。

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/clear` | 空のコンテキストで新セッション開始 | 直前のセッションはディスクに残り `/resume` で復帰可能。エイリアス `/reset` `/new` | `[name]` |
| `/compact` | 会話を要約してコンテキストを解放 | 要約方針を自然文で指定可能 | `<要約指示(任意)>` |
| `/resume` | 過去の会話を再開 | 会話IDを直接指定、または検索語でピッカーを絞り込み。エイリアス `/continue` | `[会話ID または 検索語]` |
| `/branch` | 現在地点で会話を分岐 | 同じ前提から別方針を試すときに使う。ファイルはそのまま | `[name]` |
| `/fork` | 会話を複製してバックグラウンドへ | 現在の会話は手元に残る。複製側は別セッションとして走る | `[prompt]` |
| `/rewind` | コードや会話を過去の状態へ戻す | チェックポイント復元。エイリアス `/checkpoint` `/undo` | なし |
| `/rename` | 会話に名前を付ける | プロンプトボックス・`/resume` ピッカー・端末タイトルに反映。エイリアス `/name` | `[name]` |
| `/export` | 会話をファイル or クリップボードへ | ファイル名省略時は既定の書き出し先 | `[filename]` |
| `/copy` | 直前の応答をクリップボードへ | `/copy N` でN番目に新しい応答 | `[N]` |
| `/recap` | 現時点のセッション要約を1行生成 | 進捗ログとして使える | なし |
| `/btw` | 本筋を止めずに小さな質問をする | 会話の主脈にコンテキストを混ぜたくないときに有効 | `[question]` |
| `/exit` | CLIを終了 | エイリアス `/quit` | なし |

---

## 2. コンテキスト管理

**Claude Code の性能はコンテキスト管理でほぼ決まる。**ここを触れるかどうかで体験が変わる。

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/context` | コンテキスト使用量をグリッド可視化 | 何がトークンを食っているかを色分けで表示。`all` で全量表示 | `[all]` |
| `/explain-usage` | トークンの行き先を自然文で説明 | `/context` の図が読みにくいときの言語版 | なし |
| `/autocompact` | 自動要約が走る閾値を設定 | `auto`、または 100k〜1M トークンで指定 | `[auto\|<tokens>]` |
| `/skill-doctor` | 読み込み済みスキルの棚卸し | 使われていないのにコンテキストを消費しているスキルを一覧化 | なし |
| `/focus` | フォーカス表示に切り替え | プロンプト・要約・応答だけを表示 | なし |
| `/brief` | brief-only モード切り替え | エージェントからユーザーへの通知を絞る | なし |

---

## 3. モデル・実行制御

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/model` | 使用モデルを切り替え | `opus` `sonnet` `fable` などのエイリアス、または `claude-opus-5` のような正式名 | `[model]` |
| `/effort` | 推論の努力度を設定 | `low` / `medium` / `high` / `xhigh` / `max`。深く考えさせるほど遅く高価になる | `[level]` |
| `/fast` | Fast モードの切り替え | Opus のまま出力を高速化する。小型モデルへの降格ではない | `[on\|off]` |
| `/advisor` | 要所で上位モデルに相談させる | 通常は軽いモデルで走り、判断が重い局面だけ強いモデルを呼ぶ | なし |
| `/goal` | 停止条件を設定 | 条件を満たすまで作業を続けさせる。`clear` で解除 | `[<condition>\|clear]` |
| `/plan` | プランモードの有効化・確認 | `open` で表示、`share` で共有 | `[open\|share\|<description>]` |
| `/verify` | 変更を実際に動かして検証 | テストや型検査だけでなく、影響する動線をエンドツーエンドで動かす。非自明な変更のコミット前に使う | なし |

---

## 4. 権限・セキュリティ

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/permissions` | ツール権限の許可/拒否ルール管理 | allow / deny ルールをUIで編集。エイリアス `/allowed-tools` | なし |
| `/auto-mode-setup` | auto モードに環境を学習させる | 分類器に自環境の前提を教え、ルールを微調整する | `[--request-id <uuid>]` ほか |
| `/sandbox` | サンドボックス実行の設定 | 現在の有効/無効状態を表示し、除外コマンドパターンを登録できる | `exclude "command pattern"` |
| `/fewer-permission-prompts` | 権限プロンプトを減らす | 過去のトランスクリプトを走査し、安全な読み取り専用コマンドを `.claude/settings.json` に許可リスト化 | なし |
| `/security-review` | ブランチの変更にセキュリティレビュー | 未コミット/未マージの差分を対象に脆弱性観点で監査 | なし |
| `/privacy-settings` | プライバシー設定の確認と変更 | データ利用に関する設定 | なし |

---

## 5. 設定・カスタマイズ

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/config` | 設定画面を開く / キー指定で直接設定 | 引数なしでUI、`key=value` で直接書き込み。エイリアス `/settings` | `[key=value]` |
| `/update-config` | 設定を自然文で変更 | hooks・permissions・環境変数を Claude に書き換えさせる skill | なし |
| `/hooks` | フック設定の確認 | ツールイベントごとのフック定義を表示 | なし |
| `/keybindings` | キーバインド定義ファイルを開く | `~/.claude/keybindings.json` を編集 | なし |
| `/theme` | テーマ変更 | ライト/ダーク等 | なし |
| `/color` | プロンプトバーの色を変更 | セッション単位。複数セッションの識別に有効 | なし |
| `/tui` | 端末UIレンダラの切り替え | `default` / `fullscreen` | `[default\|fullscreen]` |
| `/statusline` | ステータスラインをセットアップ | 表示内容を対話的に構成 | なし |
| `/scroll-speed` | マウスホイールのスクロール量調整 | なし | なし |
| `/terminal-setup` | 端末固有のキーバインド設定 | 例:Apple Terminal で Option+Enter による改行を有効化し、ベルを無効化 | なし |
| `/voice` | 音声モードの切り替え | `hold`(押している間)/ `tap` / `off` | `[hold\|tap\|off]` |
| `/wellbeing` | 休憩リマインダーの設定 | 静穏時間の通知も含む。エイリアス `/breaks` `/downtime` | なし |

:::message
`/vim` と `/output-style` は **`/config` に統合済み**で、現在は隠しコマンドとして「moved to /config」と案内するだけになっている。
:::

---

## 6. メモリ / CLAUDE.md

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/init` | CLAUDE.md を生成 | コードベースを読み取り、プロジェクト説明を含む CLAUDE.md(および任意で skills/hooks)を初期化 | なし |
| `/memory` | CLAUDE.md とメモリ設定を編集 | ユーザー/プロジェクト/ローカル各階層のファイルを開く | なし |
| `/pause-memory` | 自動メモリを一時停止 | このセッションだけ記憶を書き込ませない。エイリアス `/memory-pause` | なし |

---

## 7. 拡張:MCP / プラグイン / スキル

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/mcp` | MCPサーバーの管理 | 再接続、個別/全体の有効化・無効化 | `[reconnect\|enable\|disable [<server>\|all]]` |
| `/plugin` | プラグイン管理 | インストール、有効化、マーケットプレイス操作。エイリアス `/plugins` `/marketplace` | なし |
| `/reload-plugins` | 保留中のプラグイン変更を反映 | 再起動せずに現セッションへ適用。`--force` で強制 | `[--force]` |
| `/plugin-types` | プラグインAPIの型定義を書き出し | `claude-code.d.ts` と `claude-code-mcp.d.ts` を生成し、hooks モジュールに型を付ける | `[dir]` |
| `/skills` | 利用可能なスキル一覧 | ビルトイン+ユーザー+プラグイン由来 | なし |
| `/reload-skills` | ディスク上のスキル変更を取り込む | セッション中にスキルを編集したとき用 | なし |
| `/run-skill-generator` | 「このアプリの起動方法」を知るスキルを生成 | プロジェクト固有の起動手順をスキル化する | なし |

---

## 8. サブエージェント・並列実行・バックグラウンド

Claude Code を「1人のアシスタント」から「チーム」に変える群。

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/subtask` | フルコンテキスト付きでサブエージェントに委任 | 結果はこの会話に返ってくる | `<task>` |
| `/background` | 現セッションを背景へ送る | 端末を解放しつつ処理は継続。エイリアス `/bg` | `[prompt]` |
| `/stop` | バックグラウンドセッションを停止 | トランスクリプトとワークツリーは保持される | なし |
| `/tasks` | 背景で走っている全てを一覧・管理 | エイリアス `/bashes` | なし |
| `/list-agents` | 送信可能な相手を列挙 | サブエージェント、チームメイト、他の Claude セッション。エイリアス `/peers` | なし |
| `/batch` | 大規模変更を並列実行 | 調査・計画のうえ、5〜30の隔離ワークツリーエージェントに分割し、各々がPRを開く | `<instruction>` |
| `/workflows` | 実行中・完了済みワークフローを閲覧 | マルチエージェント実行の進捗確認 | なし |
| `/loop` | プロンプト/コマンドを定期実行 | 例 `/loop 5m /foo`。間隔省略でモデルが自己ペース制御。エイリアス `/proactive` | `[interval] <prompt>` |
| `/loops` | ループの一覧・作成・削除 | なし | なし |
| `/schedule` | クラウドエージェントを定期実行 | cron スケジュールでのルーティン管理。エイリアス `/routines` | なし |
| `/daemon` | バックグラウンドサービスとルーティン管理 | 常駐プロセス側の制御 | なし |

---

## 9. 開発ワークフロー(Git / PR / レビュー)

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/commit` | Git コミットを作成 | git コンテキストを収集し、メッセージ規約・ステージング規則・attribution を適用 | `[guidance]` |
| `/pr` | プルリクエストを作成 | ブランチ状況を収集し、`gh` CLI でタイトル/本文の規約に沿って作成 | `[guidance]` |
| `/code-review` | 差分やPRをレビュー | バグと整理点を検出。`/code-review ultra` でクラウド多エージェントレビュー。エイリアス `/review` | `[low\|medium\|high\|max\|ultra] [target] [--comment\|--fix]` |
| `/simplify` | 変更コードを整理 | 再利用・単純化・効率・抽象度の観点で修正まで適用。**バグ探索はしない** | `[<target>]` |
| `/ultrareview` | クラウドでブランチのバグを検出・検証 | 多エージェントで探索し、確度を検証したうえで報告 | `[PR番号]` |
| `/ultraplan` | クラウドで編集可能なプランを作成 | Claude Code on the web が下書きし、こちらで承認する | `<prompt>` |
| `/autofix-pr` | 現在のPRの問題を監視し自動修正 | CI失敗やレビュー指摘に追随 | なし |
| `/diff` | 未コミット変更のdiffパネル切り替え | ターン毎の差分も確認できる | なし |
| `/run` | このプロジェクトのアプリを起動 | 変更が実際に動くところを確認する | なし |
| `/debug` | デバッグログを有効化して調査 | 問題の切り分けを Claude に任せる | `[issue description]` |

---

## 10. Artifact / デザイン

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/artifacts` | 公開・共有済み Artifact を閲覧 | 過去に発行したページの一覧 | なし |
| `/prototype` | アイデアを動く Artifact に | 触れるプロトタイプとして公開 | なし |
| `/whiteboard` | ホワイトボード Artifact で共同作業 | ユーザーが描き、Claude がその上で回答する | なし |
| `/whiteboard-mp` | ライブ共同ホワイトボード | 複数人+Claude が同時に描き込む | なし |
| `/workshop` | 意思決定を1つずつ積み上げて設計 | 対話しながらデザインを固めていく | なし |
| `/plan-artifact` | プランを共有可能な Artifact として公開 | レビュー用にURL化 | なし |
| `/artifact-pr-review` | PRレビュー用ブリーフィングを Artifact 化 | テンプレートから生成 | `[PR番号 or URL]` |
| `/artifact-components` | 再利用コンポーネントを Artifact に埋め込む | なし | なし |
| `/dataviz` | チャート・ダッシュボード設計ガイダンス | 配色・軸・凡例まで含む設計方針を読み込む | なし |
| `/design` | Claude Design 連携 | 作成・取り込み・書き出し・同期・ログイン | `[sync\|login\|consent\|revoke\|import\|export\|status\|<prompt>]` |
| `/design-sync` | デザインシステムを claude.ai/design へ | コード側のコンポーネントを押し出す | `[プロジェクト名のヒント]` |
| `/design-login` | デザインシステムアクセスを認可 | `/design-sync` の前提となる認証 | なし |

---

## 11. クラウド・リモート・他アプリ連携

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/teleport` | セッションをクラウドへ送る/戻す | claude.ai 側で再開できる。エイリアス `/tp` | `[session]` |
| `/session` | クラウドセッションのURLとQRを表示 | エイリアス `/remote` | なし |
| `/remote-control` | スマホや claude.ai/code から操作 | エイリアス `/rc` | なし |
| `/remote-env` | クラウドエージェントの既定環境を選択 | セルフホスト環境の指定を含む | なし |
| `/cloud-plugins` | クラウドセッションのプラグイン方針 | ローカルで有効なプラグインを持ち込むか選ぶ | なし |
| `/mobile` | モバイルアプリのQRを表示 | エイリアス `/ios` `/android` | なし |
| `/desktop` | デスクトップアプリで継続 | エイリアス `/app` | なし |
| `/ide` | IDE連携の管理と状態表示 | `open` でIDE側を開く | `[open]` |
| `/chrome` | Claude in Chrome の設定 | 拡張連携の設定画面 | なし |
| `/claude-in-chrome` | Chrome を操作させる | クリック、フォーム入力、スクショ、コンソール読み取り、ページ遷移。サイト単位の権限が必要 | なし |
| `/install-github-app` | GitHub Actions 連携を設定 | リポジトリに Claude を組み込む | なし |
| `/install-slack-app` | Slack アプリを導入 | Claude Tag(Slack内Claude)を有効化 | なし |
| `/web-setup` | Claude Code on the web を設定 | GitHub アカウントと接続 | なし |

---

## 12. 診断・状態確認

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/doctor` | 設定の健康診断と修復 | 重複インストール、PATH、壊れた設定/エージェント定義、未使用スキル・MCP・プラグイン、肥大化した CLAUDE.md、遅いフック、バージョン鮮度までを点検し修正案を出す。エイリアス `/checkup` | なし |
| `/status` | 現在の状態を表示 | バージョン、モデル、アカウント、API疎通、各ツールの状態 | なし |
| `/version` | このセッションのバージョンを表示 | 自動更新で落ちてきた新版ではなく「今動いている版」 | なし |
| `/release-notes` | リリースノートを表示 | なし | なし |
| `/insights` | 自分のセッションを分析したレポート生成 | 使い方の傾向を可視化 | なし |
| `/help` | ヘルプとコマンド一覧 | なし | なし |
| `/claude-code-docs` | Claude Code 自体への質問に回答 | 機能・設定に関する質問を、動作中のビルドを根拠に回答 | `[question]` |
| `/powerup` | 対話型レッスンで機能を学ぶ | 短いレッスン形式 | なし |
| `/team-onboarding` | チーム向け導入ガイドを生成 | 自分の使用実績からガイドを作る | なし |
| `/bug` | バグ報告・会話の共有 | エイリアス `/share` | `[report]` |
| `/feedback` | Anthropic へフィードバック送信 | なし | `[report]` |

---

## 13. アカウント・課金・使用量

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `/login` | サインイン / アカウント切り替え | 既ログイン時は切り替えUIになる | なし |
| `/logout` | サインアウト | なし | なし |
| `/usage` | コスト・プラン使用量・上限への寄与を表示 | エイリアス `/cost` `/stats` | なし |
| `/usage-credits` | 使用クレジットの設定・申請 | 上限到達時に管理者へ申請できる | なし |
| `/upgrade` | Max プランへのアップグレード | レート上限と Opus 枠の拡大 | なし |
| `/passes` | 友人に無料週を配る | 紹介でクレジットを得る | なし |
| `/setup-bedrock` | Amazon Bedrock 認証の再設定 | リージョンやモデルピンも変更可 | なし |
| `/setup-vertex` | Google Vertex AI 認証の再設定 | プロジェクト、リージョン、モデルピン | なし |

---

## 14. その他・隠しコマンド

`/help` には出てこないが定義されているもの。挙動が変わりやすいので **常用は非推奨**。

| コマンド | 概要 | 状態 |
| --- | --- | --- |
| `/agents` | サブエージェント管理 | **廃止**。`.claude/agents/` を直接編集するか Claude に依頼する |
| `/vim` / `/output-style` | エディタモード / 出力スタイル | **`/config` へ統合** |
| `/extra-usage` | 追加使用枠 | **`/usage-credits` へ改称** |
| `/update` | 最新版へ切り替え | 隠し。エイリアス `/restart` |
| `/heapdump` | JSヒープを `~/Desktop` へダンプ | 隠し。障害解析用 |
| `/install` | ネイティブビルドを導入 | `[options]` |
| `/import` | 他のAIコーディングエージェントから設定移行 | `[codex\|gemini] [--dry-run]` |
| `/add-dir` | 作業ディレクトリを追加 | `<path>` |
| `/cd` | 作業ディレクトリを移動 | `<path>` |
| `/setup-cowork` | ガイド付き初期設定 | ロール選択→プラグイン導入→スキル体験→ツール接続 |
| `/radio` | Claude FM(lo-fi)を再生 | 実用性はない |
| `/stickers` | ステッカーを注文 | 実用性はない |

---

## 15. CLIコマンド(`claude <command>`)

| コマンド | 概要 | 詳細 | パラメータ |
| --- | --- | --- | --- |
| `claude` | 対話セッション開始 | 引数にプロンプトを渡すと初手として投入される | `[prompt]` |
| `claude agents` | バックグラウンドエージェントの管理 | エージェントビューを開く。`--json` でスクリプト向けに一覧出力 | `--json` `--all` `--cwd` `--model` `--effort` `--permission-mode` ほか |
| `claude attach <id>` | バックグラウンドセッションを端末に開く | `--bg` が出力した短いIDを指定 | `<id>` |
| `claude auth` | 認証管理 | `login` / `logout` / `status` | `login [--console\|--claudeai\|--sso\|--email]`、`status [--json\|--text]` |
| `claude doctor` | インストールの健全性チェック | 信頼プロンプトなしで設定を読む。修復も伴う点検はセッション内 `/doctor` | なし |
| `claude gateway` | エンタープライズ用ゲートウェイ起動 | 認証・テレメトリの中継 | `--config <path>` |
| `claude import [source]` | 他エージェントから設定移行 | Codex / Gemini の設定を取り込む | `[codex\|gemini]` `--dry-run` `--yes` |
| `claude install [target]` | ネイティブビルドを導入 | `stable` / `latest` / 特定バージョン | `[target]` `--force` |
| `claude logs <id>` | 背景セッションの直近出力を表示 | なし | `<id>` |
| `claude mcp` | MCPサーバー管理 | 追加・一覧・認証・削除・自身をMCPサーバー化 | 後述 |
| `claude plugin` | プラグイン管理 | インストール、検証、eval、マーケットプレイス | 後述 |
| `claude project purge` | プロジェクト状態を全削除 | トランスクリプト、タスク、ファイル履歴、設定エントリ | `[path]` `--all` `--dry-run` `-i` `-y` |
| `claude respawn [id]` | 背景セッションを再起動 | 現在の Claude Code バージョンで走り直す | `[id]` `--all` |
| `claude rm <id>` | 背景セッションを削除 | 安全ならワークツリーも削除。終了済みにも使える | `<id>` |
| `claude setup-token` | 長期認証トークンを作成 | Claude サブスクリプションが必要 | なし |
| `claude stop <id>` | 背景セッションを停止 | 会話は保持。`attach` で再開可能。エイリアス `kill` | `<id>` |
| `claude ultrareview [target]` | クラウド多エージェントレビュー | 現ブランチ、PR番号、ベースブランチを対象にできる | `--json` `--post` `--no-post` `--timeout <分>` |
| `claude update` | 更新確認とインストール | エイリアス `upgrade` | なし |
| `claude auto-mode` | auto モード分類器の確認・初期化 | `config` / `defaults` / `critique` / `reset` | なし |

### `claude mcp` サブコマンド

| サブコマンド | 概要 | パラメータ |
| --- | --- | --- |
| `add <name> <commandOrUrl> [args...]` | MCPサーバーを追加 | `-t/--transport <stdio\|sse\|http>`、`-e/--env KEY=value`、`-H/--header`、`-s/--scope <local\|user\|project>`、`--client-id`、`--client-secret`、`--callback-port` |
| `add-json <name> <json>` | JSON文字列で追加 | `-s/--scope`、`--client-secret` |
| `add-from-claude-desktop` | Claude Desktop から取り込み | Mac / WSL のみ |
| `list` | 設定済みサーバー一覧 | 未承認の `.mcp.json` は「⏸ Pending approval」表示 |
| `get <name>` | サーバー詳細 | 承認済みはヘルスチェックされる |
| `login <name>` | サーバーに認証 | `--no-browser`(SSH/ヘッドレス向け) |
| `logout <name>` | OAuth資格情報を破棄 | なし |
| `remove <name>` | 削除 | `-s/--scope` |
| `reset-project-choices` | プロジェクトスコープの承認/拒否をリセット | なし |
| `serve` | Claude Code 自身をMCPサーバーとして起動 | `-d/--debug`、`--verbose` |

```bash
# HTTPサーバーを追加
claude mcp add --transport http sentry https://mcp.sentry.dev/mcp

# ヘッダ付き
claude mcp add --transport http corridor https://app.corridor.dev/api/mcp \
  --header "Authorization: Bearer ..."

# stdio + 環境変数
claude mcp add my-server -e API_KEY=xxx -- npx my-mcp-server
```

### `claude plugin` サブコマンド

| サブコマンド | 概要 | パラメータ |
| --- | --- | --- |
| `install <plugin>` | マーケットプレイスから導入 | `-s/--scope <user\|project\|local>`、`-y/--yes`、`--config key=value` |
| `uninstall <plugin>` | 削除 | エイリアス `remove` |
| `enable` / `disable` | 有効化 / 無効化 | `[plugin]` |
| `update <plugin>` | 最新へ更新 | 反映には再起動が必要 |
| `list` | 導入済み一覧 | `--json`、`--available`(要 `--json`) |
| `details <name>` | 構成要素とトークンコスト見積 | `<name>` |
| `init <name>` | 新規プラグインの雛形生成 | `--with skills,agents,hooks,mcp,lsp,output-style,channel`、`--author`、`--description`、`-f` |
| `validate <path>` | マニフェストや構成を検証 | `--json`、`--strict`(CIで警告をエラー扱い) |
| `eval [target]` | eval ケースを実行し採点 | `--runs`、`--case`、`--tag`、`--threshold`、`--judge-model`、`--ablation`、`--max-cost-usd`、`--report`、`--json` |
| `tag [path]` | リリース用 git タグを作成 | `{name}--v{version}` 形式で整合性検証つき |
| `prune` | 不要になった依存を削除 | エイリアス `autoremove` |
| `marketplace` | マーケットプレイス管理 | `add <source>` / `list` / `remove <name>` / `update [name]` |

---

## 16. CLIフラグ(主要なもの)

### 起動モードと入出力

| フラグ | 概要 | パラメータ |
| --- | --- | --- |
| `-p, --print` | 応答を出力して終了 | パイプ処理向け。非対話なので信頼済みディレクトリでのみ使う |
| `--output-format` | 出力形式 | `text` / `json` / `stream-json` |
| `--input-format` | 入力形式 | `text` / `stream-json` |
| `--include-partial-messages` | 部分メッセージも流す | `--print` + `stream-json` 限定 |
| `--json-schema <schema>` | 構造化出力のスキーマ検証 | JSON Schema |
| `--max-budget-usd <amount>` | API呼び出しの上限金額 | `--print` 限定 |
| `-c, --continue` | 直近の会話を継続 | なし |
| `-r, --resume [value]` | セッションIDまたは検索でピッカー | `[session-id\|検索語]` |
| `--fork-session` | 再開時に新しいIDを発行 | `--resume` / `--continue` と併用 |
| `--session-id <uuid>` | セッションIDを指定 | 有効なUUID |
| `-n, --name <name>` | セッション表示名 | プロンプトボックスと端末タイトルに反映 |

### モデルと実行

| フラグ | 概要 | パラメータ |
| --- | --- | --- |
| `--model <model>` | 使用モデル | `fable` `opus` `sonnet` などのエイリアス、または正式名 |
| `--fallback-model <model>` | 過負荷時のフォールバック | カンマ区切りで順に試行。`--print` 限定 |
| `--effort <level>` | 努力度 | `low` `medium` `high` `xhigh` `max` |
| `--agent <agent>` | このセッションのエージェント | 設定値を上書き |
| `--agents <json>` | カスタムエージェントを定義 | JSONオブジェクト |
| `--autocompact <auto\|tokens>` | 自動要約の窓 | `auto` または 100k〜1M |
| `--system-prompt <prompt>` | システムプロンプトを差し替え | 文字列 |
| `--append-system-prompt <prompt>` | 既定プロンプトに追記 | 文字列 |
| `--system-prompt-snapshot <on\|off>` | プロンプトを1回記録して再利用 | 推奨 `on` |

### 権限とツール

| フラグ | 概要 | パラメータ |
| --- | --- | --- |
| `--permission-mode <mode>` | 権限モード | `acceptEdits` `auto` `bypassPermissions` `manual` `dontAsk` `plan` |
| `--dangerously-skip-permissions` | 全権限チェックを回避 | **ネットワーク遮断済みサンドボックス専用** |
| `--allow-dangerously-skip-permissions` | 回避を「選択肢として」有効化 | 既定にはしない |
| `--allowedTools <tools...>` | 許可ツール | 例 `"Bash(git *) Edit"` |
| `--disallowedTools <tools...>` | 拒否ツール | 同上 |
| `--tools <tools...>` | 利用可能ツールを限定 | `""` で全無効、`default` で全有効 |
| `--restricted` | 制限モード | コード実行系ツールと WebFetch を除去し、ユーザー/プロジェクト設定を無視。ファイル操作も作業ディレクトリ内に限定 |
| `--add-dir <directories...>` | 追加の作業ディレクトリ | 複数指定可 |

### 設定・拡張の読み込み

| フラグ | 概要 | パラメータ |
| --- | --- | --- |
| `--settings <file-or-json>` | 追加設定の読み込み | パスまたはJSON文字列 |
| `--setting-sources <sources>` | 読み込む設定ソース | `user,project,local` |
| `--mcp-config <configs...>` | MCP設定の読み込み | JSONファイル/文字列 |
| `--strict-mcp-config` | `--mcp-config` 以外のMCPを無視 | なし |
| `--plugin-dir <path>` | ディレクトリ/zipからプラグイン読み込み | 反復指定可 |
| `--plugin-url <url>` | URLからプラグインzipを取得 | 反復指定可 |
| `--disable-slash-commands` | 全スキルを無効化 | なし |
| `--safe-mode` | 全カスタマイズを無効化 | 設定が壊れたときの切り分け用。ポリシー設定は残る |
| `--bare` | 最小モード | フック、LSP、プラグイン同期、自動メモリ、CLAUDE.md 自動探索などを一切行わない |

### バックグラウンド・クラウド・ワークツリー

| フラグ | 概要 | パラメータ |
| --- | --- | --- |
| `--bg, --background` | 背景で起動しIDを返す | `attach` / `logs` / `stop` / `rm` で操作 |
| `-w, --worktree [name]` | 新しい git ワークツリーを作成 | 名前省略可 |
| `--tmux` | ワークツリー用に tmux セッション作成 | `--worktree` 必須。`--tmux=classic` で従来型 |
| `--cloud [desc\|id\|url]` | クラウドセッションを作成/接続 | 説明文、セッションID、claude.ai/code のURL |
| `--environment <id>` | セルフホスト環境で起動 | `ccpool_...` |
| `--teleport [session]` | teleport セッションを再開 | `[session]` |
| `--from-pr [value]` | PRに紐づくセッションを再開 | PR番号 / URL / 検索語 |
| `--remote-control [name]` | Remote Control を有効化して起動 | 名前は任意 |

### デバッグ

| フラグ | 概要 | パラメータ |
| --- | --- | --- |
| `-d, --debug [filter]` | デバッグモード | カテゴリ絞り込み可(`"api,hooks"` や `"!1p,!file"`) |
| `--debug-file <path>` | デバッグログの出力先 | 指定するとデバッグモードも有効化 |
| `--verbose` | 詳細出力 | 設定を上書き |
| `--include-hook-events` | フックのライフサイクルイベントも出力 | `--output-format=stream-json` 限定 |

---

## 17. スラッシュ以外の特殊入力

コマンドではないが、覚えておくと入力量が大きく減る。

| 入力 | 意味 |
| --- | --- |
| `!<command>` | シェルコマンドをその場で実行し、出力を会話に取り込む |
| `@<path>` | ファイル/ディレクトリを参照として添付 |
| `#<text>` | メモリ(CLAUDE.md)への追記 |
| `/<name>` | スラッシュコマンド / スキル呼び出し |

---

## まとめ:最初に覚える12個

全部を覚える必要はない。実務で効くのは以下に集約される。

| 優先度 | コマンド | 理由 |
| --- | --- | --- |
| ★★★ | `/context` | コンテキストが何に食われているかを常に見る癖をつける |
| ★★★ | `/clear` `/compact` | 長期化したセッションの性能劣化を防ぐ |
| ★★★ | `/model` `/effort` | タスクの重さに合わせてコストと精度を調整する |
| ★★★ | `/permissions` | 承認プロンプト地獄を抜ける最短経路 |
| ★★☆ | `/doctor` | 設定が壊れたとき、まずこれ |
| ★★☆ | `/rewind` | 失敗した変更からの復帰。`git` より速い |
| ★★☆ | `/code-review` `/simplify` | コミット前の二段構え(バグ検出と整理は別物) |
| ★★☆ | `/subtask` | 調べ物を本流のコンテキストから隔離する |
| ★☆☆ | `/memory` `/init` | プロジェクト知識の蓄積 |
| ★☆☆ | `/skill-doctor` | スキルが増えてきたときの棚卸し |

---

## 注記

- 本記事のコマンド定義は **v2.1.259 のバイナリから機械抽出** している。バージョンによって増減する。
- プラン(Pro / Max / Team / Enterprise)、フィーチャーフラグ、プラットフォームによって **有効にならないコマンドがある**。手元で `/help` を実行したときに出ないものは、その環境では無効と考えてよい。
- 隠しコマンド(`isHidden`)は将来予告なく変更・削除される可能性が高い。自動化スクリプトに組み込むのは避けたほうがよい。
