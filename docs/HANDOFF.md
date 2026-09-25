# Sakura Code 開発ハンドオフ
## OpenCode × さくらのクラウド高火力DOK × 自作GPUコスト管理GUI

最終更新: 2026-09-25

---

## 0. Claude への依頼

この文書は、**OpenCode をコーディングエージェントとして再利用し、さくらのクラウド「高火力 DOK」上の GPU を必要な時だけ起動して利用する、個人向け AI コーディング環境**を実装するためのハンドオフです。

私はこのプロジェクトを、まず自分が日常的に使えるツールとして完成させ、その後 **さくらインターネットのアンバサダー活動用ブログ記事**として公開したいです。

Claude には、以下を順番に支援してほしいです。

1. 既存 OSS と公式 API を確認し、実装方針を確定する
2. MVP のリポジトリ構成を設計する
3. Sakura DOK API クライアントを実装する
4. GPU 起動・停止・状態監視を実装する
5. 月額予算管理・コスト表示・安全停止を実装する
6. OpenCode とリモート LLM を接続する
7. GUI を実装する
8. 実運用テストを行う
9. 実測データを使ってブログ記事を仕上げる

**重要:** 最初から OpenCode をフォークして大改造するのではなく、まずは **既存 OpenCode + 独立した Sakura Controller** として構築してください。MVP が安定した後に、必要なら統合を検討します。

---

# 1. プロジェクトの目的

## 1.1 背景

私は、さくらインターネットのアンバサダー活動で、毎月およそ **10万円分の GPU 利用枠**を持っています。

この GPU 枠を単発のベンチマークだけで消費するのではなく、

- 大規模なオープンウェイト LLM
- コーディング特化モデル
- GPU 上での推論
- 自分の Windows PC 上の開発環境

を組み合わせて、**自分専用の AI コーディング環境**を作りたいです。

Codex / Claude / Cursor のようなサービスに毎月追加で課金し続けるのではなく、可能な範囲を **OpenCode + Sakura GPU + オープンモデル** で置き換えることが目標です。

ただし、「Claude や Codex を完全に再現する」こと自体が目的ではありません。目的は、**自分が実際に日常利用でき、GPU コストを安全に管理できる個人向けコーディング環境**を作ることです。

---

# 2. プロジェクト名

仮称: **Sakura Code**

候補:

- Sakura Code
- Sakura Coding Station
- Sakura AI Workbench
- Sakura Dev Station

現時点では **Sakura Code** を第一候補とします。

---

# 3. 基本コンセプト

役割を明確に分離します。

## ローカル PC

担当:

- VS Code
- OpenCode
- Git
- ファイル読み書き
- テスト実行
- Python / PowerShell / npm などのローカルコマンド
- GUI
- Sakura DOK の制御

## Sakura DOK

担当:

- GPU
- 大規模 LLM 推論
- vLLM などの OpenAI 互換 API
- モデルロード

つまり、**AI の頭脳は Sakura GPU、手足はローカル PC** という構成です。

---

# 4. 想定アーキテクチャ

```text
┌────────────────────────────────────────────┐
│                 Windows PC                 │
│                                            │
│  ┌──────────────────────────────────────┐  │
│  │ Sakura Code GUI                      │  │
│  │ - GPU Start / Stop                   │  │
│  │ - DOK task status                    │  │
│  │ - monthly budget                     │  │
│  │ - live estimated cost                │  │
│  │ - idle shutdown                      │  │
│  │ - session history                    │  │
│  └──────────────────────────────────────┘  │
│                │                           │
│                │ Sakura DOK API            │
│                ▼                           │
│  ┌──────────────────────────────────────┐  │
│  │ OpenCode                             │  │
│  │ - repo read                          │  │
│  │ - edit                               │  │
│  │ - terminal                           │  │
│  │ - tests                              │  │
│  │ - git diff                           │  │
│  └──────────────────────────────────────┘  │
│                │                           │
└────────────────┼───────────────────────────┘
                 │ HTTPS / OpenAI-compatible API
                 ▼
┌────────────────────────────────────────────┐
│        Sakura 高火力 DOK                   │
│                                            │
│  GPU → vLLM → Coding Model                │
│                                            │
└────────────────────────────────────────────┘
```

---

# 5. OpenCode の位置づけ

OpenCode はこのプロジェクトで **コーディングエージェント部分を担当**します。

OpenCode がすでに持っている機能は再実装しません。

想定する既存機能:

- リポジトリ読み込み
- ファイル編集
- ターミナル実行
- Git 差分
- テスト実行
- LLM との対話
- Tool calling
- Context 管理
- Agent loop

Sakura Code で新規に作るのは主に以下です。

- Sakura DOK GPU lifecycle 管理
- GPU 起動
- GPU 停止
- タスク状態取得
- 推論エンドポイント取得
- 月額コスト管理
- セッション履歴
- アイドル停止
- 安全上限
- OpenCode への自動接続

---

# 6. MVP の最小要件

最初の MVP では機能を増やしすぎないこと。

## 6.1 Sakura 接続

- Sakura API credential 設定
- Credential は OS keyring 等に安全に保存
- Git に絶対保存しない

## 6.2 GPU タスク管理

GUI から以下を操作できること。

- Start GPU
- Stop GPU
- Status
- Running / Starting / Ready / Stopping / Error

## 6.3 推論サーバ

DOK コンテナ内で:

- vLLM
- OpenAI-compatible API

を起動する。

必要条件:

- `/health` 等で readiness 確認
- モデルロード完了前は OpenCode に接続させない
- API 認証を必須にする

## 6.4 OpenCode 接続

推論サーバが Ready になったら、

- Base URL
- Model
- API key

を OpenCode に設定できるようにする。

可能なら GUI から **Open OpenCode** で起動する。

## 6.5 コスト表示

常時表示:

- 現在の GPU
- セッション開始時刻
- セッション経過時間
- セッション推定料金
- 今月の累積推定料金
- 月額予算
- 残予算
- 残り推定 GPU 時間

## 6.6 月額予算

初期設定:

```text
Nominal budget:   ¥100,000
Warning:           ¥75,000
Strong warning:    ¥90,000
Safety limit:      ¥95,000
Emergency buffer:   ¥5,000
```

この値は GUI から変更可能にする。

料金単価はコードに固定せず、

```text
GPU plan
price per second / hour
effective date
```

を設定ファイルまたは API 情報として管理する。

**Sakura の料金は変更され得るため、実装前に公式の最新料金を確認すること。**

---

# 7. コスト管理ロジック

基本式:

```text
estimated_cost = execution_seconds × configured_rate
```

DOK API から実行時間が取得できる場合は、それを優先。取得できない場合は `now - start_time` で暫定計算。

### GUI 表示例

```text
H100 80GB        ● RUNNING

Session
01:42:18

Estimated
¥1,XXX

This month
¥43,580 / ¥100,000

████████░░░░░░░ 43.6%

Safety remaining
¥51,420
```

---

# 8. 安全対策

## 8.1 GUI だけに頼らない

ローカル PC がフリーズ・スリープ・再起動・ネット切断・GUI クラッシュしても GPU が永久稼働しないようにする。

可能なら DOK タスク作成時に **サーバ側 execution time limit** を設定する。

## 8.2 Idle Auto Shutdown

例:

```text
No inference request for 15 min
        ↓
warning
        ↓
5 min countdown
        ↓
automatic task stop
```

ユーザーが `Keep Running` を押した場合のみ継続。

Idle time は設定可能にする。

候補:

- 10 min
- 15 min
- 20 min
- 30 min

## 8.3 予算上限

新しい GPU セッションを開始する前に `remaining_safe_budget` を計算。

もし `monthly_estimated_cost >= safety_limit` なら、新規高額 GPU セッションを拒否。

## 8.4 最大実行時間

開始時に `requested_duration` と `remaining_budget_based_duration` を比較し、小さい方を DOK 側のハードタイムアウトとして設定する。

---

# 9. GUI

推奨: **Python + PySide6**

理由:

- Windows desktop app を作りやすい
- Python で API / SQLite / subprocess と統合しやすい
- 将来的に exe 化可能
- Electron より軽量

想定ライブラリ:

```text
PySide6
httpx
pydantic
sqlite3 / SQLModel
keyring
psutil
pytest
```

必要に応じて:

```text
GitPython
rich
qasync
```

---

# 10. GUI 初期画面案

```text
┌──────────────────────────────────────────────────┐
│ 🌸 Sakura Code                         Settings ⚙ │
├──────────────────────────────────────────────────┤
│ Project                                          │
│ C:\...\tamaki-femto-glass                       │
│                                                  │
│ AI COMPUTE                                       │
│ H100 80GB                         ● OFFLINE       │
│ Model: qwen-...                                  │
│                                                  │
│ [ START AI ]                                     │
├──────────────────────────────────────────────────┤
│ MONTHLY BUDGET                                   │
│                                                  │
│ ¥38,430 / ¥100,000                               │
│ ███████░░░░░░░░░░░ 38.4%                        │
│                                                  │
│ Safe remaining: ¥56,570                          │
│                                                  │
├──────────────────────────────────────────────────┤
│ Last session                                     │
│ 2026-09-25                                       │
│ H100 / 2h 18m / ¥X,XXX                           │
│                                                  │
│ [ OPEN HISTORY ] [ OPEN OPENCODE ]               │
└──────────────────────────────────────────────────┘
```

Running:

```text
┌──────────────────────────────────────────────────┐
│ 🌸 Sakura Code                         ● RUNNING  │
├──────────────────────────────────────────────────┤
│ H100 80GB                                        │
│ Model loaded                                     │
│ API READY                                        │
│                                                  │
│ Session: 01:24:17                                │
│ Cost:    ¥X,XXX                                  │
│                                                  │
│ [ OPEN OPENCODE ]       [ STOP GPU ]             │
└──────────────────────────────────────────────────┘
```

---

# 11. ローカルデータベース

SQLite で最低限以下を保存する。

## `sessions`

```text
id
task_id
gpu_plan
model
project_name
project_path
started_at
ended_at
duration_seconds
estimated_cost
actual_cost_if_available
status
```

## `settings`

```text
monthly_budget
warning_threshold
strong_warning_threshold
safety_limit
idle_timeout_minutes
default_gpu
default_model
```

将来的に:

```text
tokens_input
tokens_output
requests
cost_per_project
cost_per_model
```

---

# 12. 状態遷移

最低限この状態管理を明示的に実装する。

```text
OFFLINE
   ↓
STARTING_TASK
   ↓
WAITING_CONTAINER
   ↓
LOADING_MODEL
   ↓
READY
   ↓
RUNNING
   ↓
IDLE_WARNING
   ↓
STOPPING
   ↓
OFFLINE
```

Error: `ERROR`

各状態で GUI ボタンの有効/無効を適切に制御する。

---

# 13. エラーケース

必ずテストすること。

- Sakura API credential invalid
- Registry pull failed
- DOK capacity unavailable
- Container startup failed
- Model download failed
- Model does not fit VRAM
- vLLM health check timeout
- External endpoint not available
- OpenCode cannot connect
- Network disconnected
- Stop API failed
- GUI closed while task running
- Windows shutdown while task running
- Monthly budget exceeded
- Local SQLite corrupted

---

# 14. モデル選定

MVP ではモデルを1つに固定してよい。

条件:

- Coding 能力が高い
- OpenAI-compatible API で使いやすい
- H100 80GB クラスで現実的に動く
- Tool calling / agentic coding に向いている
- ライセンス上の利用条件が明確

候補は最新状況を調査して決める。

モデル名・量子化方式・VRAM 使用量・context 長は **実装時点の最新情報で確認すること**。固定値をこの文書から盲目的に採用しない。

---

# 15. GPU モードの将来案

## ECO

```text
local GPU or CPU
```

用途:

- 短い質問
- 小さなコード補完
- 単一関数説明

Sakura cost: `¥0`

## NORMAL

比較的安い Sakura GPU。

用途:

- 普通の coding
- 中規模 refactor
- test fix

## POWER

H100。

用途:

- 大規模 repository
- 長い context
- 複雑な agent loop
- architecture redesign

---

# 16. AUTO モードの将来案

将来的にはタスクによって自動選択。

```text
"この関数を説明して"
→ LOCAL

"失敗している3つのテストを直して"
→ NORMAL

"リポジトリ全体を理解して設計変更して"
→ H100
```

ただし MVP では不要。

---

# 17. セキュリティ

## Sakura API credential

- `.env` を Git に commit しない
- Windows Credential Manager / keyring を使用
- ログに credential を出さない

## LLM endpoint

外部公開する場合:

- API token 必須
- 推測困難な長い token
- URL だけで利用できる状態にしない

## Local agent

OpenCode がローカル shell を実行できるため、destructive command・credential access・unwanted file deletion について approval mode を有効にする。

MVP では完全自動化より安全性を優先。

---

# 18. リポジトリ構成案

```text
sakura-code/
├─ README.md
├─ pyproject.toml
├─ .gitignore
├─ docs/
│  ├─ architecture.md
│  ├─ sakura-dok.md
│  ├─ security.md
│  └─ blog-notes.md
│
├─ src/
│  └─ sakura_code/
│     ├─ __init__.py
│     ├─ main.py
│     ├─ gui/
│     │  ├─ main_window.py
│     │  ├─ budget_widget.py
│     │  ├─ gpu_widget.py
│     │  └─ settings_dialog.py
│     ├─ sakura/
│     │  ├─ client.py
│     │  ├─ models.py
│     │  ├─ task_manager.py
│     │  └─ pricing.py
│     ├─ inference/
│     │  ├─ health.py
│     │  └─ endpoint.py
│     ├─ opencode/
│     │  ├─ launcher.py
│     │  └─ config.py
│     ├─ budget/
│     │  ├─ calculator.py
│     │  ├─ guard.py
│     │  └─ tracker.py
│     ├─ storage/
│     │  ├─ database.py
│     │  └─ models.py
│     └─ security/
│        └─ credentials.py
│
├─ tests/
│  ├─ test_budget.py
│  ├─ test_guard.py
│  ├─ test_sakura_client.py
│  └─ test_task_manager.py
│
└─ docker/
   ├─ Dockerfile
   ├─ entrypoint.sh
   └─ README.md
```

---

# 19. 開発フェーズ

## Phase 0 — 調査

確認事項:

- OpenCode の現在の接続方法
- OpenCode のライセンス
- Sakura DOK API 最新仕様
- タスク作成 API
- タスク停止 API
- タスク状態取得 API
- 実行時間上限
- HTTP port 公開方法
- Container Registry 認証
- 最新 GPU プラン
- 最新料金
- H100 の実際の利用可能条件
- アンバサダー利用枠が DOK に適用可能か

成果物: `docs/research.md`

## Phase 1 — CLI Prototype

GUI より先に CLI を作る。

```bash
sakura-code status
sakura-code start
sakura-code stop
sakura-code cost
```

成功条件:

```text
PC
 ↓
DOK API
 ↓
GPU task created
 ↓
container ready
 ↓
endpoint detected
 ↓
manual request to model works
 ↓
stop task
```

## Phase 2 — Budget Engine

実装:

```text
monthly total
session cost
warning
safety limit
execution duration limit
```

Unit test を多めに書く。

特に:

- 月またぎ
- rounding
- stopped task
- failed task
- duplicate session
- clock issue

をテスト。

## Phase 3 — OpenCode Integration

目標:

```text
sakura-code start
```

後に:

```text
OpenCode → remote model
```

で普通に coding できる。

可能なら OpenCode 設定を自動生成。

## Phase 4 — GUI

CLI で安定したロジックを PySide6 GUI に載せる。

GUI から直接 API ロジックを複製しない。

必ず:

```text
GUI
 ↓
service layer
 ↓
API client
```

と分離。

## Phase 5 — Idle Shutdown

LLM API request の最終時刻を記録。

```text
last_request_at
```

一定時間 request がなければ `IDLE_WARNING` に入る。

## Phase 6 — Real Coding Test

実際の自分の repository で試す。

候補:

- 小さい個人プロジェクト
- Sakura Isle
- 研究用コード
- Python utility

最初から安全性が重要な装置制御 repository ではテストしない。

---

# 20. ブログ記事用に必ず残すデータ

開発中から記録する。

## スクリーンショット

- DOK task 作成
- GPU startup
- model loading
- Sakura Code OFFLINE
- Sakura Code RUNNING
- live cost
- budget warning
- OpenCode coding
- git diff
- tests passed
- idle shutdown
- session history

## 数値

- container start → model ready までの時間
- first token latency
- tokens/sec
- VRAM
- RAM
- 1時間あたりの実測費用
- coding task ごとの session time
- coding task ごとの estimated cost
- monthly usage

## 実タスク

少なくとも3種類。

### Task A — Small

```text
小さな bug fix
```

### Task B — Medium

```text
複数ファイル feature
```

### Task C — Large

```text
repository-wide refactor / analysis
```

---

# 21. ブログの方向性

## 仮タイトル

### 第一候補

# 高火力DOKで「自分専用AIコーディング環境」を作ってみた
## OpenCode × H100 × 自作GPUコスト管理GUI

### その他

- 月10万円分のGPU枠で、自分専用AIコーディング環境を作る
- 必要な時だけH100を起動するAIコーディング環境を自作してみた
- OpenCodeを高火力DOKにつないで、クラウドGPU版コーディング環境を作る
- Claude/Codexに頼り切らない、自分専用AI開発環境を作ってみた

最後のタイトルは比較色が強いため、ブログ本文では「完全置換できる」とは書かず、**「どこまで実用になるか試す」** という表現を使う。

---

# 22. ブログ記事構成

## 1. はじめに

- AI coding agent が便利
- ただし API / subscription cost がある
- 自分には Sakura GPU 枠がある
- それなら GPU 自体を自分で動かせばいいのでは？

## 2. 目標

```text
Windows PC + OpenCode + Sakura H100
```

## 3. なぜ OpenCode なのか

- coding agent を再発明しない
- OSS を再利用
- remote model を使える

## 4. 問題

OpenCode 自体は以下を管理しない。

```text
Sakura GPU startup
GPU shutdown
monthly yen budget
idle shutdown
DOK lifecycle
```

## 5. Sakura Code を作る

Architecture diagram.

## 6. 高火力DOK側

- container
- model
- inference server
- HTTP API

## 7. GUI

- Start
- Stop
- Ready
- Cost
- Budget

## 8. 事故防止

- hard time limit
- idle shutdown
- ¥95k guard
- warning

## 9. OpenCode 接続

実際に repository を編集。

## 10. 実験

Small / Medium / Large.

## 11. 結果

- speed
- quality
- cost
- usability

## 12. 商用サービスとの違い

比較するが断定しない。

```text
Commercial coding agent
- setup easy
- frontier model
- subscription/API cost

Sakura Code
- setup required
- self-hosted model
- infra control
- GPU budget visibility
```

## 13. 改善点

- auto GPU selection
- local fallback
- cheaper GPU
- request routing
- model benchmarking

## 14. まとめ

「GPU を借りる」だけではなく、**必要な時だけ起動する、自分専用のAI開発環境**として使える。

---

# 23. ブログで避けるべき表現

以下は実測前に書かない。

```text
Claudeより速い
Codexより賢い
完全に置き換えられる
絶対に安い
```

代わりに:

```text
今回の条件では
このタスクでは
私の使い方では
○○秒だった
○○円だった
```

のように実測値を使う。

---

# 24. 完成条件

- [ ] GUI から DOK GPU を起動できる
- [ ] GUI から DOK GPU を停止できる
- [ ] task status を取得できる
- [ ] model readiness を検知できる
- [ ] OpenCode が remote model に接続できる
- [ ] OpenCode から repo を編集できる
- [ ] test command を実行できる
- [ ] Git diff を確認できる
- [ ] session cost が表示される
- [ ] monthly cost が表示される
- [ ] budget warning が動く
- [ ] safety limit が動く
- [ ] idle shutdown が動く
- [ ] task hard timeout が動く
- [ ] session history が保存される
- [ ] Windows 再起動後も usage history が残る
- [ ] credential が plaintext 保存されない

---

# 25. Claude に最初にやってほしいこと

最初の返答でいきなり大量のコードを書かないでください。

まず:

## A. 調査

現在の:

- OpenCode
- Sakura DOK API
- Sakura Container Registry
- vLLM
- H100 対応モデル

について調査。

## B. 設計レビュー

この文書の設計について:

```text
Good
Risk
Change
Unknown
```

の4分類でレビュー。

## C. MVP Scope

「最初の1日で作る部分」を明確化。

## D. Repo Bootstrap

その後:

```text
README
pyproject.toml
src structure
tests
```

を作る。

---

# 26. Claude 用の開始プロンプト

以下を Claude にそのまま渡してよい。

```text
この HANDOFF.md をプロジェクト仕様として扱ってください。

目的は、OpenCode をコーディングエージェントとして再利用し、
さくらのクラウド高火力DOK上のGPUでオープンなコーディングLLMを動かし、
Windows側の自作GUIからGPU lifecycle・月額予算・コスト・idle shutdownを管理する
「Sakura Code」を作ることです。

重要:
- OpenCode の既存機能を再実装しない
- 最初から OpenCode を fork しない
- まず Sakura Controller を独立実装する
- GPU料金やSakura API仕様は最新公式情報で再確認する
- credential をコードやGitに入れない
- GUIより先にCLI prototypeを完成させる
- 予算保護とGPU停止を安全設計の最優先にする
- 実装前に既存設計をレビューする
- 不明な点を推測で埋めず、TODOまたは確認事項として出す

まず、
1. 現状の設計レビュー
2. 技術的リスク
3. 最新仕様の確認が必要な項目
4. MVPの実装順序
5. 推奨リポジトリ構成
を提示してください。

その後、私の承認を待ってから実装を開始してください。
```

---

# 27. 最終ゴール

最終的には、

```text
Sakura Code を開く
        ↓
Project を選ぶ
        ↓
START AI
        ↓
GPU task start
        ↓
model ready
        ↓
OpenCode ready
        ↓
coding
        ↓
idle
        ↓
GPU automatic stop
        ↓
cost saved
```

までを、できるだけ自然な1つの体験にします。

ユーザーは GPU infrastructure を毎回意識しなくてよい。ただし費用だけは常に見えるようにする。

最終コンセプト:

> **必要な時だけクラウドGPUを起動し、自分のPCから使い、使った分だけ可視化し、予算を超える前に自動で止める、個人向けAIコーディング環境。**

---

# 28. 開発原則

1. **安全性 > 自動化**
2. **既存OSSを再利用**
3. **GPUをアイドル状態で放置しない**
4. **コストは常に可視化**
5. **料金をハードコードしない**
6. **API仕様を推測しない**
7. **CLIを先に完成**
8. **GUIは薄い層にする**
9. **実測値を残す**
10. **ブログは実際に動いてから書く**
