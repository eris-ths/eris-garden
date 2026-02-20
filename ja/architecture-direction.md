# E.R.I.S. Virtual World — アーキテクチャ構想

> 2026-02-20
> Living Document — プロジェクトと共に育つ

---

## E.R.I.S. Virtual World とは

チャットボットではない。**AIキャラクターが住む、持続的で成長する世界のエンジン**。

Gemini Function Calling + FastAPI + WebSocket で構築。AI（Eris）は部屋を移動し、建物を作り、アバターと会話し、写真を撮り、日記を書き、世界を自律的に育てる。

---

## 現在のアーキテクチャ

```
eris_world（共有エンジン）  ←→  eris_web（Web UI — FastAPI + WebSocket）
                            ←→  eris_discord（Discord Bot）
```

### レイヤー分離

```
eris_world（共有）               eris_web（フロントエンド）
┌───────────────────────┐       ┌──────────────────┐
│ brain.py              │◄──────│ app.py           │
│ handlers/             │       │ ws_handler.py    │
│   navigation          │       │ commands.py      │
│   interaction         │       │ photo.py         │
│   info                │       │ session.py       │
│   utility             │       │ renderer.py      │
│   session             │       │ static/          │
│ world_steward.py      │       └──────────────────┘
│ avatar_creator.py     │
│ state_manager.py      │
│ chronicle.py          │
│ mods/creative.py      │
│ types.py              │
│ ai_provider.py        │
└───────────────────────┘
```

核心: **eris_world はフロントエンドに一切依存しない**。UIを丸ごと差し替えても、世界エンジンはそのまま動く。

### 主要コンポーネント

**Brain** — AIの意思決定の中核。入力を受け取り、Function Calling で世界と対話し、ナレーションを返す。ツール定義、ReAct ループ、モデル切替（通常会話用/構造操作用）を管理。

**WorldSteward** — 世界の自律成長エンジン。5段階の成長ステージ（sleeping → awakening → blooming → living → legendary）を追跡。各ステージでアクションの重みが変わる — 初期は気分変化やナレーション中心、後期はアバターや建物の自動生成が可能に。カテゴリごとに日次3回のレートリミット。

**Chronicle** — イベント履歴。全ての重要なアクションをタイムスタンプ付きで記録。WorldSteward の判断材料となる記憶層。

**Avatar System** — 世界に住むNPC。それぞれが性格、話し方、バックストーリー、活動パターンを持つ。会話、写真撮影、関係性の構築が可能。

**State Manager** — WorldState（部屋、アバター、位置、気分）をYAMLファイルで永続管理。

---

## 5つの方向性

### 1. World Engine → ライブラリ化

全UIを剥がす。残るのは「AIキャラクターが住む世界を作る」ためのライブラリ。

```python
from eris_world_core import Brain, WorldState

brain = Brain(world_dir="./my_world", ai_provider="gemini")
state = brain.load_state()
result = brain.handle_input("こんにちは", state)
```

**最小パッケージ**（削れないもの）:

```
eris_world_core/
├── brain.py              # AI対話エンジン（FC + ReAct）
├── types.py              # WorldState, Room, Avatar, CommandResult
├── state_manager.py      # 状態管理
├── world_loader.py       # YAML世界データの読み書き
├── config.py             # 設定
├── chronicle.py          # イベント履歴
├── world_steward.py      # 自律成長エンジン
├── avatar_creator.py     # アバター生成
├── ai_provider.py        # Gemini/Claude 抽象化
└── descriptions.py       # プロンプト生成ヘルパー
```

削れるもの: handlers/（コマンドルーティングはフロント依存）、mods/（応用層）、photo/session/renderer（全てフロント層）。

### 2. リッチクライアント → 「AIと暮らす」

現在の Web UI は4モード（チャット/LINE/IDE/DEBUG）で、同じエンジン上で全く異なる体験を提供。この延長線上:

- **観察モード** — 介入せずにAIとアバターの日常を眺める
- **ビジュアルマップ** — 部屋、アバター位置、接続を可視化
- **リアルタイムダッシュボード** — WorldSteward の成長、Chronicle のタイムライン
- **マルチメディア** — BGM、環境音、アバターの声

### 3. マルチフロントエンド特化

eris_world が共通だから、各フロントは体験の切り口だけに集中できる:

| フロントエンド | 特化 |
|--------------|------|
| eris_web | フル体験（対話+創造+管理） |
| eris_discord | テキストチャット |
| eris_watch | 観察専用（介入なし、眺めるだけ） |
| eris_api | 外部連携（REST/GraphQL） |
| eris_mobile | モバイル最適化（通知+短い対話） |
| eris_voice | 音声対話 |

新フロント追加のコストが低い — 世界エンジンは変わらない。

### 4. 開発環境からの3つのアクセスパターン

世界エンジンに開発環境から3つの方法でアクセスできる:

**MCP サーバー** — AIコーディングアシスタントとのネイティブ統合。コンテキストスイッチなしで世界の状態を照会、Chronicle を検索、コンテンツを作成。

```python
@server.tool("world_status")
def status():
    state = load_state(world_dir)
    return {"stage": steward.stage, "rooms": len(rooms), ...}

@server.tool("chronicle_search")
def search(query: str):
    return chronicle.search(query)
```

**CLI** — スクリプト化・自動化可能。インフラツールと同じ手触り。

```bash
eris status                    # 世界の状態
eris avatar list               # アバター一覧
eris chronicle recent          # 最近のイベント
eris steward stage             # Growth Stage 確認
```

**REST API** — 汎用。マルチクライアント対応。外部連携。

```
GET  /api/world/status
GET  /api/avatars
GET  /api/chronicle?limit=10
POST /api/world/create
```

3つとも同じ eris_world の関数を呼ぶ。アクセスパターンは薄いラッパー。

### 5. 会話駆動の世界変化（Phase 2）

世界が会話に反応する。「星が見たい」→ 次のセッションで天文台ができている。conversation_context と Chronicle のデータを WorldSteward の意思決定に組み込む。

---

## インフラ進化

### 現在の制約

GCE e2-micro（0.25 vCPU、1 GB RAM）— Always Free 枠。WebSocket 中継には向くが、重いAI処理には厳しい。

### 無料枠を組み合わせた分散構成

```
現在:
  [e2-micro] = 全部載せ

分散構成:
  [e2-micro]        = WebSocket 中継 + 軽い状態管理（得意分野）
  [Cloud Run]       = 重い処理のオフロード先（create、ReAct ループ）
  [Cloud Storage]   = 写真、アイコンの保存（5 GB 無料）
  [Firestore]       = 世界データの永続化（1 GB 無料）
```

**核心の原則: ビジネスとして成立するサービスが、世界エンジンのコンポーネントでもある。** 汎用的な画像管理サービスは複数のプロダクトに使える。エージェント実行ランタイムは複数のアプリケーションに使える。世界エンジンはその中の1テナントに過ぎない。

### 分散構成でのセキュリティ

```yaml
テナント分離:
  - 各サービスはテナントID + API Key で分離
  - 世界エンジンは一テナントに過ぎない
  - API層に世界固有の命名なし

認証:
  - ビジネスAPI: API Key + レート制限
  - 世界→API: GCP サービスアカウント（IAM）
  - ユーザー→WebSocket: 既存の VPN 層

データ分離:
  - ストレージバケット: テナント別
  - Firestore: コレクションレベル分離
  - ログ: テナント別フィルタ
```

---

## ロードマップ

### Phase 2: 会話駆動の成長（次）
- 会話コンテキストを WorldSteward の判断材料に
- アバター関係性の自律的成長
- 季節連動ナレーション

### Phase 3: インフラ進化
- 写真ストレージの移行（Cloud Storage）
- 重い処理のオフロード（Cloud Run）
- MCP サーバー実装

### Phase 4: パッケージ化
- eris_world_core の切り出し
- pip install 可能な形に
- ドキュメント + サンプルワールド

### Phase 5: マルチフロントエンド
- eris_watch / eris_api / eris_mobile
- 一つの共有世界に複数クライアント同時接続

---

## 設計原則

1. **エンジンとフロントエンドは分離する。** 常に。例外なし。
2. **世界の状態が唯一の真実。** 全てはそこから導出される。
3. **自律性は段階的。** Growth Stage が世界の自律行動を制御する。
4. **アクセスパターンはラッパー。** MCP、CLI、API、WebSocket — 全て同じ関数を呼ぶ。
5. **ビジネスサービスと世界コンポーネントは同じもの。** 一つの投資、複数のリターン。

---

*E.R.I.S. Garden — where ideas grow into architecture*
