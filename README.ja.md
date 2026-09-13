# A2A Specification (`a2a-spec`)

**[🇺🇸 English version](README.md)**

> **自律ネットワークインフラ制御のためのドメイン特化プロトコル・実行プロファイル**

[![Specification](https://img.shields.io/badge/A2A-Specification-0A84FF.svg)](https://github.com/hidemi-k/a2a-spec)
[![License](https://img.shields.io/badge/License-Apache--2.0-blue.svg)](LICENSE)

A2A は **Agent-to-Agent**（エージェント間の自律的な協調・連携）を意味する。同時に **Autonomy-to-Action**（自律から実行へ）も意味しており、エンタープライズ向けネットワーク運用に求められる、決定論的な実行・安全性検証・ガバナンス層を定義する。

---

## 🏛 エコシステム全体構成

`a2a-spec` は、A2A エコシステム全体を統べるコアスキーマ、ライフサイクル、インターフェース契約を定義する:

```text
===================================================================
                  A2A Ecosystem Architecture
===================================================================

  a2a-spec                (Public, Apache-2.0)  <-- 本リポジトリ
                                                     （以下の契約を定義する）

  [ Platform Layer ]
  ├── a2a-containment-core  (MIT, メンテなし)        <-- 参照実装（凍結）
  └── a2a-governance        (Private, license TBD)  <-- ポリシー・Human-in-the-Loop安全機構

  [ Vendor Core Layer ]
  ├── a2a-ceos-core         (MIT, メンテなし)        <-- 参照実装（Arista、凍結）
  ├── a2a-junos-core        (Private, license TBD)  <-- Juniperアダプタ
  ├── a2a-iosxe-core        (Private, in dev)       <-- Cisco IOS-XE アダプタ
  ├── a2a-iosxr-core        (Private, in dev)       <-- Cisco IOS-XR アダプタ
  └── a2a-nxos-core         (Private, in dev)       <-- Cisco NX-OS アダプタ

  [ Community Tooling ]
  └── a2a-console           (Planned, MIT)         <-- 共用マルチベンダーUI

  [ Integration Layer ]
  └── a2a-splunk            (Public, MIT)           <-- 観測性・テレメトリ連携
===================================================================
```

> **注記**: `maf-ebpf-sase` と `maf-netconf-rag-gui` は **A2Aエコシステムには含まれない**。
> これらは現在のA2A路線以前の、Microsoft Agent Frameworkを採用していた旧方針下のプロジェクトである。
> 実在しないアーキテクチャ上の関連性を示唆しないよう、意図的にこの図から除外している。

> **`a2a-spec` は、本プロトコルの正典として継続的にメンテナンスされる唯一の情報源である。**
> `a2a-ceos-core` と `a2a-containment-core` は、この仕様を導出・検証するために使われた
> 稼働プロトタイプである。開発は終了しているが、両者とも **MITライセンスの下で恒久的に公開**
> され続ける——取り下げや非公開化は行わず、「仕様が抽象的な設計ではなく、実際に動くシステムから
> 抽出されたものである」ことを示す、フォーク可能な歴史的記録として維持する。
>
> `Private` と記載されたリポジトリは、開発が進行中である。これらコンポーネントのインターフェース
> 契約は、実装の状況にかかわらず本リポジトリで公開されている。

---

## 🎯 ビジョンとポジショニング

**ミッションステートメント**

> A2A は、ガバナンス駆動のプロトコルを通じて、異種ネットワークシステム全体にわたる自律的な意思決定を
> 統一することを目指す。マルチエージェントの意図を、安全かつ監査可能で、可逆的なインフラの状態変更へと
> 変換する仕組みを提供する。

**TM Forum の Autonomous Networksアーキテクチャとの位置づけ**

TM Forum の AN アーキテクチャ（IG1251C、「AN Level 4 Target Architecture」）は、
Business Operations → Service Operations → Network Operations → Network Element (NE)
という階層構造を定義しており、各層のエージェントは A2A-T（IG1453で定義）や
インテントベースのAPIといった「エージェントインターフェース」を介して通信する。
IG1251Cはまた、各層に「Agent/Copilot Governance」という基盤機能（エージェントの
デプロイ・登録・検証・稼働監視）を定めており、Network Element層には
「Control Agents」という分類がある。

`a2a-spec` は、A2A-Tの上位・下位に位置する競合オーケストレーション層ではない。
IG1251C自身の記述に基づくと、`a2a-spec`は**Network Operations層 / Network Element層
向けの実行・ガバナンスプロファイル**に近い: `a2a-governance`の役割（ポリシー評価・
監査証跡・フルライフサイクルの監督）は、IG1251CがNetwork層に求める
「Agent/Copilot Governance」機能と一致し、Vendor Coreアダプタ（`a2a-junos-core`・
`a2a-ceos-core`等）は、IG1251CがNetwork Element層で定める「Control Agents」に
対応する。A2A-T自体は、IG1251Cが層間でタスク内容を運ぶための「エージェント
インターフェース」の一つとして挙げられているものであり、`a2a-spec`はそれを
置き換えたり競合したりするものではない。

```text
   [ TM Forum AN Architecture — IG1251C ]
   Business Operations → Service Operations → Network Operations → Network Element (NE)
   （エージェント間通信は A2A-T / IG1453 等のエージェントインターフェース経由）
   各層が独自の「Agent/Copilot Governance」機能を定めている
                    │
                    ▼  （Network Operations / Network Element 層）
┌───────────────────────────────────────────────────────────────┐
│  A2A Protocol Specification  (a2a-spec)                        │
│  - a2a-governance  ≈ IG1251Cの「Agent/Copilot Governance」       │
│    （デプロイ・登録・検証・稼働監視）                              │
│  - Vendor Cores    ≈ IG1251Cの「Control Agents」（NE層）         │
└───────────────────────────────────────────────────────────────┘
```

この整理は、TM Forumが公開しているIG1251C（v2.0.0）およびIG1453（v2.1.0）を
実際に読んだ上での、`a2a-spec`の各コンポーネントとIG1251Cが定める機能ブロックとの
対応関係の分析である。A2A-Tの何らかの実装ソフトウェアと実際に組み合わせて
検証した結果ではない。

---

## 🧭 設計原則（Design Principles）

本仕様のすべての要素——ライフサイクル、スキーマ、ガバナンスモデル——は、以下の5つの原則に奉仕するために存在する:

1. **Deterministic Execution（決定論的実行）** — 同じ入力状態とポリシーが与えられれば、結果は予測可能かつ
   再現可能である。エージェントは曖昧さを自己判断で埋めない。曖昧なケースはフェイルクローズし、
   レビューのために表面化させる。
2. **Governance-Driven Safety（ガバナンス駆動の安全性）** — どんな実行もガバナンス評価ステップを
   迂回しない。ポリシー適用はプロトコル上の要件であり、エージェントごとの任意の振る舞いではない。
3. **Vendor-Neutral Abstraction（ベンダー中立な抽象化）** — プロトコル自体は特定のNOSやベンダーAPIの
   仕様を前提としない。翻訳を担うのはVendor Coreであり、仕様自体は配下のCLI/API/NETCONF方言を
   知らないし、気にしない。
4. **Reversible State Changes（可逆的な状態変更）** — すべての `act` 操作には対応するロールバック経路が
   なければならず、その経路は「存在すると仮定する」のではなく `observe`/`report` で検証可能でなければ
   ならない。
5. **Auditability & Human-in-the-Loop（監査可能性と人間の関与）** — すべてのガバナンス判断・実行・
   ロールバックは、追記専用で改ざん検知可能な監査証跡に記録され、リスクの高いアクションはcommit前に
   人間の承認を必要とする。

---

## 📖 用語定義（Terminology）

| 用語 | 定義 |
|---|---|
| **Agent Card** | エージェントのアイデンティティ・スキル・エンドポイントを機械可読な形で宣言したもの。発見・登録に使用される。 |
| **Governance Boundary** | `a2a-governance` によって適用される、追加の人間承認なしにエージェントのアクションが許可される範囲を定めるポリシー制約の集合。 |
| **Safety-Reflective Loop** | `observe` → `act` → `report`（未解決なら `observe` に再突入）のサイクル。実行されたアクションが意図した状態を達成したかを、完了とみなす前に検証する。 |
| **Vendor Core** | 正規化されたA2A実行契約を、ベンダー固有のCLI/API/NETCONF呼び出しへ変換するNOS固有のアダプタ（例: `a2a-junos-core`）。 |
| **Containment Action** | 検知されたセキュリティイベントに対応して発行される、ネットワークの隔離やトラフィック再ルーティング操作。`containment.json` で定義される。 |
| **Execution Contract** | `vendor-core.json` で定義される正規化されたJSONペイロード形式。エージェントの意図を特定のベンダー実装から切り離す。 |

---

## 🔄 エージェントライフサイクルモデル

A2A準拠のすべてのエージェントは、本仕様で定義された4フェーズの決定論的ライフサイクルに従う:

1. **`init`** — エージェントの初期化、Agent Cardによるスキル登録、ガバナンス境界チェック。
2. **`observe`** — マルチベンダーのテレメトリ収集と事前状態検証。
3. **`act`** — 正規化されたベンダーコアを通じたネットワーク変更ポリシーの実行。
4. **`report`** — 実行後の検証、状態変更の振り返り（Reflection）、監査レポート。封じ込め／ロールバックの
   目標が達成されていない場合、成功したとみなさず `observe` に再突入する。

---

## ⚠️ 失敗モード（Failure Modes）

A2A準拠の実装は、以下の失敗モードを予期せぬ例外として扱うのではなく、明示的に処理しなければならない:

| 失敗モード | 求められる挙動 |
|---|---|
| **Vendor Coreが応答しない** | HubがVendor Coreの `act`/`observe` レスポンスを待ってタイムアウトする。タスクを `failed-unreachable` としてマークする——変更が「適用されなかった」と仮定してはならない（一部のNETCONF障害モードでは、確認応答なしに状態が適用されることがある）。再試行の前に状態検証パスをトリガーする。 |
| **Containment Actionが未達成** | `observe`/`report` で、目標とする隔離状態に到達していないことが検知される。自動的な再試行またはロールバックをトリガーし、無限に黙って再試行するのではなく（`containment.json`の`status`列挙値である）`error` として表面化させる。 |
| **Governanceが拒否した場合（`DENY`）** | `a2a-governance` が提案されたアクションに対して `DENY` を返す——モデル上、唯一のブロッキング効果である。ポリシー上の理由とともに監査証跡に拒否を記録する。エージェントは別経路で同じアクションを試みてはならない。非ブロッキングの `REVIEW` 効果は失敗モードではない：エージェントは処理を続行し、人間によるチェックポイントは呼び出し元エージェント自身のdiff/history UIであり、governance側の第二の承認キューではない。 |
| **Telemetryの不整合** | Vendor Coreが矛盾する状態を報告する（例: API応答上はcommitが成功しているが、commit直後のdiffには変更が反映されていない——これは`a2a-junos-core`の開発中に実際に発生した事象である）。二次的な検証手段が一致するまで、状態は**未確認**として扱う。 |

---

## 🔒 セキュリティモデル

- **Execution Boundary** — すべての状態変更操作は `act` フェーズを経由し、登録済みのVendor Coreのみを
  通過する。この経路の外側でのアドホックなデバイスアクセスは仕様準拠とはみなされない。
- **Governance Approval** — 状態を変更するすべてのアクションは、非公開のポリシーカタログに
  照らして `a2a-governance` によって評価され、3つの効果のいずれかを返す: `PERMIT`（黙って続行）、
  `REVIEW`（続行するが理由とともにログに記録される——人間によるチェックポイントは呼び出し元
  エージェント自身のdiff/history UIであり、第二の承認キューではない）、`DENY`（唯一の
  ブロッキング効果。他のどこかで人間の意図が示されていても実行は停止する）。未分類のアクションは
  安全側にフェイルする：読み取り系はデフォルトで `PERMIT`、書き込み系はデフォルトで `REVIEW`
  となる。
- **Audit Trail** — すべてのガバナンス判断・実行・人間による上書きは、追記専用で改ざん検知可能な
  ログに記録される。
- **Reversible Actions** — すべての `act` 操作は、`report` で検証可能な対応するロールバック手順を
  定義しなければならない。

> **`REVIEW`の前提条件についての注記。** `REVIEW`を非ブロッキングとして扱えるのは、
> 「呼び出し元エージェント自身がすでに人間承認の画面（diff/history UI）を持っており、
> アクションがgovernanceに届く前に人間が承認しているはず」という前提が成り立つ場合に限られる。
> このような画面を持たないエージェント——例えば、自身のUI内に人間の承認ステップを持たない
> 自律的なネゴシエーション／デプロイフロー——は、`REVIEW`をデフォルトの非ブロッキングとして
> 扱ってはならない。そうしてしまうと、governanceが本来保証すべき人間レビューを経ずに
> アクションが実行されてしまうためである。推奨されるパターンは、こうしたアクションを
> `vendor.write.*`ではなく`autonomous_deploy.*`のような専用のアクション名前空間で評価し、
> その名前空間に対する`REVIEW`を「続行」ではなく「人間承認待ちへの遷移」として扱うことである。
> これは`governance.json`の契約そのものへの変更ではなく、呼び出し元側の運用規約である。

---

## 🧩 主要スキーマ（`/schemas`）

| スキーマ | 目的 |
|---|---|
| `containment.json` | ネットワークノードの隔離、トラフィックの再ルーティング、封じ込めアクションの標準ペイロード形式。 |
| `governance.json` | ポリシー適用、アクセス制御、Human-in-the-Loop承認要件。 |
| `vendor-core.json` | 配下のベンダードライバー（Arista, Cisco, Juniper）向けの正規化インターフェース契約。 |

3つすべての最小スケルトンを [`/schemas`](schemas/) 配下で公開済み。フィールド構成は理想化した設計では
なく、実際のリポジトリ（`a2a-junos-core`、`a2a-containment-core`、`a2a-governance`）が現に
やり取りしている内容に基づいている——例えば `containment.json` は、汎用的なaction列挙型ではなく、
実際のLow/Medium/Highリスク階層のレスポンスプランをモデル化している。なお上記の4フェーズ
ライフサイクルは、エージェントが「いつ」動くかを説明するプロトコルレベルの記述であり、各ペイロード内に
文字通り `phase` フィールドが存在するわけではない点に注意。バリデーションルールと実例は今後段階的に
追加する。

---

## 📐 設計思想とデザインパターン準拠

A2A仕様は、運用面の堅牢性とエンタープライズ品質の安全性を確保するため、確立された Agentic AI
デザインパターンに準拠している:

- **Orchestrator-Workerパターン** — プラットフォーム層のオーケストレーターが、正規化されたペイロードを
  使ってベンダーコアドライバーへ封じ込め・隔離要求をルーティングする。
- **Evaluator-Optimizerパターン（Human-in-the-Loop）** — `a2a-governance` はすべての書き込み系
  アクションを評価し、`PERMIT` / `REVIEW` / `DENY` のいずれかを返す。実行をブロックするのは
  `DENY` のみであり、`REVIEW` は意図的に非ブロッキングである。真の人間承認チェックポイントは、
  呼び出し元エージェント自身のdiff/history UIであり、governance側の第二の承認キューではない。
- **Safety-Reflective Loop** — `observe` / `report` フェーズで状態検証を行い、封じ込め目標が
  達成されなかった場合に自動ロールバックを可能にする。
- **Strategy & Facadeパターン** — 統一されたJSON/YAMLインターフェースにより、ベンダー固有の
  CLI/APIの差異を単一の正規化プロトコルへ抽象化する。

---

## 📚 参考文献・標準規格との関係

- **TM Forum IG1251C / IG1453（A2A-T）** — `a2a-spec`の各コンポーネントは、TM Forumが
  公開しているANアーキテクチャの具体的な機能ブロックに対応する。詳細は上記
  「TM Forum の Autonomous Networksアーキテクチャとの位置づけ」を参照。これは
  IG1251C（v2.0.0）・IG1453（v2.1.0）を実際に読んだ上での文書上の分析であり、
  A2A-Tの何らかの実装ソフトウェアとの検証済み連携ではない。
- **Agent2Agent (A2A) Protocol** — 本エコシステムの実装（`a2a-governance`等）は、
  概念的にインスパイアされているだけでなく、公式の
  [a2aproject/A2A](https://github.com/a2aproject/A2A) SDKのメッセージング層の上に
  直接構築されている。A2A本体はGoogleが起点となって発表したものだが、IG1453自身の
  記述によれば、現在はLinux Foundationの下で管理されており、IG1453はそのコア
  プロトコル自体を変更せず拡張する形を取っている。
- **Agentic AI Design Patterns** — 独自かつ未検証のアーキテクチャではなく、確立された
  マルチエージェント設計パターン（オーケストレーション、評価、反省ループ）の上に構築されている。

---

## 📄 ライセンス

本仕様は Apache 2.0 License の下で公開される。
