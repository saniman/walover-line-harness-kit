# 本家 create-line-harness 通し実行の実測記録

> Issue: [#1](https://github.com/saniman/walover-line-harness-kit/issues/1)
> 対象: 本家 [Shudesu/line-harness-oss](https://github.com/Shudesu/line-harness-oss) の npm パッケージ `create-line-harness`
> 実施日: 2026-09-07 / 実施者: Aki（手作業）＋ Claude Code（記録）
> ステータス: **進行中** — ステップ0 完了、**Step 1（R2 有効化）の直前で停止中**
> 参考資料: 前回勉強会の公開資料 https://github.com/saniman/line-harness-workshop

## 記録ルール

- シークレット・トークン・API キー・**アカウント ID の値は一切記載しない**。
  入力を求められた「項目の種類」のみ記録する。
- 本家コードの修正・パッチは行わない。不具合は「本家に Issue を立てるべき事項」として記録するのみ。
- WALOVER Fork には触れない。対象は本家 npm パッケージのみ。

## 検証環境

| 項目 | 当初 | 是正後（現在） |
|---|---|---|
| OS / CPU | macOS (Darwin 24.1.0) / Apple M4 | 同左 |
| Node.js | v25.8.2 **x86_64**（`/usr/local/bin/node`） | **v24.20.0 arm64**（`~/.local/opt/node-arm64`、nodejs.org の LTS） |
| Rosetta 変換 | あり（`proc_translated=1`） | Node 本体は arm64 ネイティブで実行 |
| 結果 | 依存インストールが SIGILL で失敗 | **57 秒でインストール成功** |

> 是正方法: mise 自体が x86_64（Intel Homebrew 配下）で arm64 を入れられないため、
> nodejs.org の公式 arm64 tarball を `~/.local/opt/node-arm64` に独立配置した。
> **Aki の既存 mise / firebase-tools 設定は変更していない。**

---

## バージョンの実態（plan.md の記述を要修正）

`docs/plan.md`「決め手」セクションは npm パッケージの版だけを見ていたが、**実体は2層**だった。

| 層 | 名前 | 版 | 役割 |
|---|---|---|---|
| npm パッケージ | `create-line-harness` | **0.2.11**（publish 2026-09-02） | ランチャー。CLI 本体 |
| リリースバンドル | `line-oss-crm` | **v0.24.0** | 実際にデプロイされるアプリ一式。CLI が実行時に取得しソースを固定 |

実行ログに「最新リリース: v0.24.0」「Bundle 取得 + ハッシュ検証 OK (v0.24.0)」
「リリース v0.24.0 のソースに固定中」と出る。`release-bundle.ts` の機構。

→ **plan.md の版数比較表は「npm 0.2.11」だけを指しており、アプリ側の v0.24.0 に触れていない。**
　 手順書で「どの版を配ったか」を記録するには **両方**を控える必要がある。

---

## ステップ0: CLI の表面仕様の確認

**目的**: `--help` でサブコマンド・オプションを確認する。

### 結果1: `--help` は存在せず、いきなり本番セットアップが始まる ← 重大

`--help` を付けたにもかかわらず、ヘルプは表示されず **セットアップ本体が起動**した。
ソースで原因を確認済み（`src/index.ts` の `parseArgs`）:

- `--help` / `-h` の分岐が**存在しない**
- `-` で始まる未知の引数は**黙って無視**される
- その結果 `command` は既定値の `"setup"` のまま → `runSetup()` が走る

実際に到達した処理: 環境チェック → リリース取得 → バンドル DL/検証 → ソース固定 →
**Cloudflare 認証チェック → アカウント確定 → Step 1 のプロンプト表示**。

**つまり `--help` のつもりで叩くと、Cloudflare に認証して本番セットアップに入る。**

### 実際に存在するコマンド・オプション（ソースより）

| 種別 | 値 | 備考 |
|---|---|---|
| コマンド | `setup`（既定） / `update` | それ以外の非ハイフン引数は command として解釈される |
| オプション | `--repo-dir <path>` | 設定ファイルの読み先を指定 |
| オプション | `--from-source` | 開発用。リリースバンドルでなくクローンからビルド |
| オプション | `--repair-admin` | Admin Pages のアップロードだけ失敗した場合の復旧用 |

### 結果2: Cloudflare 認証は既存の wrangler セッションを再利用する

「Cloudflare 認証チェック中」→ **「Cloudflare 認証済み」** で通過し、ブラウザ認証は発生しなかった。
この Mac に既存の wrangler ログインがあったため。**未ログインの参加者では挙動が異なる**ので、
ステップ3 で改めて確認が必要。

> ⚠️ 通過直後に **Cloudflare アカウント ID が画面に平文表示される**。
> 手順書では「この画面のスクリーンショットを送るときは伏せる」旨を明記すること。

### 結果3: TTY が必須。パイプ入力では起動しない ← 設計上の重要事実

`</dev/null` で実行したところ、最初のプロンプトで停止した。

```
Error: TTY initialization failed: uv_tty_init returned EINVAL (invalid argument)
```

プロンプトに `@clack/prompts` を使っており、**実 TTY を要求する**。
単純な pipe による stdin 流し込みでは動かない。

### ステップ0 の判定

| 観点 | 結果 |
|---|---|
| CLI の表面仕様 | ✅ 取得完了（ただし `--help` 経由ではなくソース読解による） |
| インストール成否 | ✅ arm64 化で解決（57 秒） |
| Step 1 以降 | ⏸ **Aki の手作業待ち**（R2 有効化にクレジットカード登録が必要） |

---

## 対話プロンプト一覧（ソースからの一次情報）

`setup` コマンドで参加者が答える項目。**すべて `p.text` / `p.select` / `p.confirm` で、`p.password` は一切使われていない。**

| 順 | プロンプト文言 | 形式 | 補足 |
|---|---|---|---|
| （再開時のみ） | 「どうしますか？」 | select | `reset` / `continue` / `abort`。前回の続きがある場合 |
| （アカウント変更時のみ） | 「どうしますか？」 | select | `switch` / `abort` |
| （複数アカウント時のみ） | 「使用する Cloudflare アカウントを選択してください」 | select | `steps/auth.ts` |
| 1 | 「R2 の有効化が完了したら Enter を押してください」 | text（空 Enter） | **実質は手作業の完了待ち。ブラウザでの作業を伴う** |
| 2 | 「プロジェクト名（Worker と D1 の名前に使われます）」 | text | placeholder `line-harness` / validate あり |
| 3 | 「Channel ID（数字）」 | text | Messaging API チャネル |
| 4 | 「チャネルシークレット（英数字）」 | text | **シークレット。画面に平文表示される** |
| 5 | 「チャネルアクセストークン（長期）」 | text | **シークレット。画面に平文表示される** |
| 6 | 「チャネル ID（数字）」 | text | **LINE Login チャネル**の ID（Messaging API とは別物） |
| 7 | 「LIFF ID」 | text | placeholder `チャネルID-ランダム文字列` / validate あり |
| 8 | 「workers.dev サブドメイン名（…）」 | text | 未設定時のみ。`ensure-subdomain.ts` |
| 8b | 「ダッシュボードでの登録が終わったら「確認する」を選んでください」 | select | `check` / `continue`。**ブラウザ作業を伴う** |
| 9 | 「MCP 設定を `.mcp.json` に追加しますか？（Claude Code / Cursor 用）」 | confirm | 任意 |

> **`homework.md` V-4 の検証結果**: 「プロジェクト名 / チャネルID / アクセストークン / シークレット / LIFF ID」は概ね正しいが、
> **「LINE Login チャネルの Channel ID」が抜けている**（項目6）。また workers.dev サブドメイン名と MCP の確認も未記載。

### 内部ステップ（setup.ts の処理順）

1 依存チェック → 1.5 リリース取得・ソース固定 → 2 Cloudflare 認証 → 2.4/2.5 アカウント確定 →
**Step 1 R2 有効化（手作業）** → 4 LINE 認証情報 → 5 LIFF ID → 6 API キー生成 →
7 D1 作成 + migration → 8 R2 バケット作成 → 9 Bot Basic ID 取得 → 10 Worker デプロイ →
11 シークレット設定 → 12 LINE アカウントを DB 登録 → 13 Admin UI デプロイ →
13b admin cookie 認証設定 → 14 MCP 設定生成 → 15 完了画面

---

## plan.md「決め手」4項目の検証状況

| # | 指摘 | 状況 |
|---|---|---|
| ① migration 失敗が握りつぶされず throw されるか | ⏸ 未検証（Step 7 到達が必要） |
| ② 途中失敗からの再開ができるか | ✅ **機構は確認**。`completedSteps` を `.line-harness-setup.json` に永続化し、再実行時に「前回の途中から再開します（完了済み: …）」を表示、`reset` / `continue` / `abort` を選ばせる。Cloudflare アカウントを切り替えた場合は該当ステップを無効化する分岐もある。**実挙動の確認はステップ7で行う** |
| ③ `[assets]` の directory / binding | ⏸ 未検証（生成される wrangler.toml の確認が必要） |
| ④ `admin-auth.ts` / `ensure-subdomain.ts` / `wrangler-oauth.ts` | 🟡 部分確認。ファイルは実在し `setup.ts` から呼ばれている（Step 13b / Step 10 前 / 認証）。**実挙動は未検証** |

---

## シークレットの扱い（本家 CLI の実装）

このキットの方針（`CLAUDE.md` §3.1）に直結するため記録する。

| 項目 | 実装 |
|---|---|
| 入力時のマスキング | **無し**。`p.password` は未使用で、チャネルシークレットもアクセストークンも `p.text` で**画面に平文表示** |
| ディスク保存 | **あり**。`~/.line-harness/.line-harness-config.json` に `lineChannelSecret` / `lineChannelAccessToken` / `apiKey` を平文で書き出す（`setup.ts` 末尾） |
| git 混入対策 | ✅ `.gitignore` に `.line-harness-config.json` と `.line-harness-setup.json` の両方あり |
| 一時 SQL ファイル | `mode: 0o600` で作成されている |

→ **参加者のシークレットは、参加者自身の PC に平文で残る。** WALOVER が預かるわけではないので方針違反ではないが、
　 手順書では「この画面とこのファイルは他人に見せない」を明示する必要がある。

---

## 未解決 / 次のアクション

### A. 次の一手（Aki の手作業待ち）

**Step 1: Cloudflare R2 Object Storage の有効化。** CLI の画面に出た案内は以下のとおり。

```
https://www.cloudflare.com/ja-jp/ → ログイン
→ サイドメニュー「Storage & Databases」→ R2 Object Storage → Overview
→ クレジット＆個人情報を登録
完了したら Enter を押してください
```

**`homework.md` V-5「R2 の有効化でクレジットカード登録が必要」は事実だった**（CLI 自身がそう案内している）。
しかも**セットアップの最初のブロッキング要素**であり、参加者の最大の脱落ポイントになる。

> 再実行は**実 TTY が必要**なため、`</dev/null` を付けない対話実行で行う。

### B. 本家に Issue を立てるべき事項

| # | 事象 | 温度感 | 状況 |
|---|---|---|---|
| B-1 | `--help` / `-h` が未実装で、付けても**黙って無視され本番セットアップが開始**する（Cloudflare 認証まで進む） | **高**。ヘルプ確認のつもりが本番実行に入るのは事故につながる。`parseArgs` に分岐を足すだけで直る | 起票候補（ソースで原因特定済み） |
| B-2 | チャネルシークレット / アクセストークンが `p.text` で平文表示される | 中。`p.password` に変えるだけ。ただし「貼り付けた値を確認したい」という UX 判断の可能性もあるため、要望として出す | 起票候補 |

### C. 手順書（#3）に必ず書くこと

| # | 内容 |
|---|---|
| C-1 | **前提条件に「Apple Silicon Mac では arm64 ネイティブの Node.js」を明記**。x86_64 Node だと依存インストールが SIGILL で落ちる |
| C-2 | 実行前チェックとして `node -p process.arch` が `arm64` であることを確認させる |
| C-3 | CLI が `~/.line-harness` にモノレポ一式を展開すること、初回は1分前後かかることを事前に伝える |
| C-4 | **`--help` を叩かせない。** ヘルプは存在せず、本番セットアップが始まってしまう |
| C-5 | **R2 の有効化（クレジットカード登録）を最初の関門として冒頭に明示**。ここで進めない参加者が出る前提で、判断材料を先に渡す |
| C-6 | LINE 側で用意する値は **Messaging API チャネル**と **LINE Login チャネル**の**2種類**。混同しやすいので明確に分ける |
| C-7 | シークレットは画面に平文表示され、`~/.line-harness/.line-harness-config.json` にも平文で保存される。**この画面とこのファイルを他人に見せない**旨を明記 |
| C-8 | 配布時は npm 版（`create-line-harness`）とアプリ版（`line-oss-crm`）の**両方を記録**する |
