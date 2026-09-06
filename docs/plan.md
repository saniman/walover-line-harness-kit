# 配布キット 準備計画

> 作成日: 2026-09-07 / 対象: 次回勉強会（10月上旬想定）での配布
> ステータス: 決定済み・Issue 起票済み（#1〜#4）

---

## 0. 結論（3行）

1. 配布するツールは **本家 [Shudesu/line-harness-oss](https://github.com/Shudesu/line-harness-oss) の `npx create-line-harness@latest`**。WALOVER は自前のセットアップツールを作らない。
2. WALOVER が作るのは **手順書・サポートポリシー・検証記録の3点だけ**。SP 合計 **10**（実働 6〜8日）で、10月上旬の勉強会に間に合う。
3. WALOVER の Fork（`saniman/line-harness-oss`）は**既存顧客専用**として維持し、配布物には一切含めない。
4. **（2026-09-07 追加）** 本家 CLI は Claude 経由で実行できないと判明したため、**ローカル GUI ラッパー**（SP13）を追加する。コードは別リポジトリ [saniman/walover-line-harness-gui](https://github.com/saniman/walover-line-harness-gui) で管理する。ただし**今回の勉強会は手順書で実施し、GUI は次回**に向けて作る。

---

## 1. なぜこの形になったか（判断の記録）

### 検討したが採用しなかった案

| 案 | 内容 | 不採用の理由 |
|---|---|---|
| Workers for Platforms | dispatch namespace で顧客ごとに物理分離 | Workers Paid 前提・運用複雑度が高く、現状の案件数に対して過剰（Aki 判断） |
| 単一デプロイ + `tenant_id` | 1 Worker に全顧客を同居させ、D1 に `tenant_id` 列を足す | **Stripe / freee / Google Calendar のシークレットが Worker の env に固定されている**ため、同居させると他社の入金・記帳が WALOVER 側に来る。DB のクエリ修正（約420文）より、この5つの外部連携の作り替えが本丸。SP 45〜60（2.5〜3ヶ月） |
| Fork に追従した自前 CLI | Fork の `packages/create-line-harness` を現状に合わせて改修 | **本家がすでに直していた**（下記） |

### 決め手: 本家 CLI が現役で保守されている

| | WALOVER Fork が持つ CLI | 本家の最新 |
|---|---|---|
| バージョン | v0.1.13（2026-04-02 で停止） | **v0.2.11** |
| 最終コミット | — | **2026-09-06** |

> **⚠️ #1 の実測で判明した補正**（[`docs/findings/cli-walkthrough.md`](findings/cli-walkthrough.md)）
>
> 上表の `v0.2.11` は **npm パッケージ `create-line-harness`（ランチャー）** の版であり、
> **実際にデプロイされるアプリの版ではない**。CLI は実行時にリリースバンドル
> **`line-oss-crm` v0.24.0** を取得してソースを固定する（`release-bundle.ts` の機構）。
> 配布時・サポート時に控えるべきは **2つの版の両方**。

Fork の CLI を調べて挙げた不具合の多くは、本家では修正済みだった:

| 指摘 | 本家 v0.2.11 |
|---|---|
| migration の失敗を `catch {}` で握りつぶす | **修正済み** — `isBenignSchemaError()` で良性エラーだけスキップし、それ以外は `throw`。`bootstrap.sql` 方式 + 適用後に `line_accounts` の存在を検証 |
| 途中失敗からの再開ができない | **修正済み**（本家 #295, 2026-08-24） |
| `[assets]` の `directory` / `binding` 未設定 | **修正済み**（`dist/client` / `ASSETS`） |

さらに `admin-auth.ts` / `ensure-subdomain.ts` / `release-bundle.ts` / `wrangler-oauth.ts` が新設され、
npm への自動 publish（OIDC）まで整備されている。

→ **WALOVER が CLI を保守する理由がない。** 本家をそのまま使い、WALOVER は「本家が自動化しない部分」だけを埋める。

### 方針転換: ローカル GUI ラッパーを追加する（2026-09-07 決定）

#### なぜ追加するか

当初は「手順書だけ作る」方針だった。追加する理由は2つ。

1. **非エンジニア参加者にターミナルを見せたくない**（特にオフライン参加者）
2. **本家 CLI は Claude 経由で実行できないと判明した**（#1）

2 が決定的だった。CLI は `@clack/prompts` を使っており実 TTY を要求する。
**Claude Code CLI・Claude Desktop の Code タブとも標準入出力が TTY ではない**ため、
Claude に実行させると最初のプロンプトで `uv_tty_init EINVAL` で落ちる。

| Claude 経由で | 可否 |
|---|---|
| 本家 CLI を直接実行 | ❌ TTY が無く不可 |
| **GUI サーバーを起動** | ✅ 可能。**サーバー起動に TTY は不要で、PTY は GUI 側が握る** |

→ GUI は「見た目を整える」ためではなく、**Claude 主導のフローを成立させる唯一の手段**。

#### なぜ再実装ではなくラッパーに留めるか

- 本家 CLI は現役で保守されている（#1 で確認。リリースバンドル方式・再開機構・`[assets]` 設定すべて正常動作）
- 再実装すれば**本家の修正に追従し続ける義務**が発生する。それは「WALOVER が CLI を保守しない」という
  このキットの前提そのものを覆す
- 成果物はあくまで**「フォーム入力 → 本家 CLI への中継」**。migration・assets 設定・デプロイ処理は一切模倣しない

#### なぜサーバー実行ではなくローカルなのか

Cloudflare Worker でツールを実行する案を検討したが、**採らない**。

| | 理由 |
|---|---|
| 技術的に不可 | Workers は V8 アイソレートで、**プロセスを起動できない**（`child_process` なし・FS なし・PTY 不可）。Containers/Sandbox なら可能だが下記が残る |
| **信頼モデルが壊れる** | サーバー実行にすると、参加者の LINE シークレットと Cloudflare の操作権限が **WALOVER 側を通過する**。README の「シークレットを受け取ることも保管することもありません」という約束が崩れる。DB に保存しなければよい、という話ではない |

→ **すべて参加者の PC 内で完結させる。** 画面もローカルサーバーが配信する
（GitHub Pages 上の画面から `http://localhost` を叩くと混在コンテンツ制限に掛かるため）。

#### GUI の本命価値

#1 の実測では、手作業13件のうち **GUI で隠せるのは2件だけ**（CLI 実行中の入力）。
残り11件は LINE と Cloudflare のブラウザ作業で、ターミナルとは無関係。

→ 価値は「ターミナルを隠すこと」より、**「手順書と入力欄が同じ画面に並ぶこと」**にある（#10）。
　 だから **#3 の手順書が先**であり、GUI はその資産を流用する形になる。

---

### なぜ Fork と別リポジトリにしたか

1. 成果物がドキュメント3本で、Fork のコードと無関係
2. Fork の `apps/worker/wrangler.toml` には **WALOVER 本番の D1 ID・Cloudflare アカウント ID・freee 取引先 ID** が入っている。参加者が誤って Fork を clone する経路を、リポジトリを分けるだけで断てる
3. 本家との同期作業（`/upstream-sync`）に勉強会資材が混ざらない

---

## 2. スコープ

### やること

- 本家 CLI を1回通し、**手作業が残る箇所**と**所要時間**を実測する（#1）
- 参加者向けセットアップ手順書（#3）
- サポートポリシー（#2）
- 第三者による通し検証（#4）
- **（2026-09-07 追加）** 本家 CLI をラップする**ローカル GUI**（別リポジトリ [walover-line-harness-gui](https://github.com/saniman/walover-line-harness-gui)）。次回勉強会に向けて

### やらないこと

- LINE Harness 本体のコードを書く / 本家 CLI を改修する
  → 本家側の不具合を見つけたら **本家に Issue を立てる**
- **本家 CLI のロジックを再実装する**（GUI はあくまで「フォーム入力 → 本家 CLI への中継」）
- **参加者のシークレットを WALOVER のサーバーに通す構成**（GUI は参加者の PC 内で完結させる）
- WALOVER の Fork 固有機能（イベント予約 + Stripe 決済 / freee 領収書自動化 / モバイルオーダー / AI週次配信）を配布物に含める
- 参加者のシークレットを WALOVER 側で預かる仕組み

---

## 3. Issue 一覧

| ID | Issue | タイトル | SP | 依存 | 状態 |
|---|---|---|---|---|---|
| 1 | [#1](https://github.com/saniman/walover-line-harness-kit/issues/1) | 調査: 本家最新 CLI を通し実行し、手作業が残る箇所と所要時間を実測する | 2 | — | ✅ **完了** |
| 2 | [#2](https://github.com/saniman/walover-line-harness-kit/issues/2) | docs: サポートポリシー（3ヶ月・セットアップのみ・LINEグループのみ） | 2 | — | 未着手 |
| 3 | [#3](https://github.com/saniman/walover-line-harness-kit/issues/3) | docs: 参加者向けセットアップ手順書 | 3 | #1, #2 | 🟡 1・2・4 公開済み／3 が未作成 |
| 4 | [#4](https://github.com/saniman/walover-line-harness-kit/issues/4) | 第三者（非開発者）による通し検証 | 3 | #3 | 未着手 |
| **計** | | | **10** | | |

SP の目安: **SP1 = 半日（2〜3時間） / SP2 = 1日 / SP3 = 2日**

### 依存グラフ

```
#1（スパイク）─┐
               ├── #3（手順書）── #4（第三者検証）
#2（ポリシー）─┘
```

`#1` と `#2` は独立なので並行できる。

### GUI ラッパー（2026-09-07 追加）

**コードとタスクは別リポジトリで管理する。**
→ [saniman/walover-line-harness-gui](https://github.com/saniman/walover-line-harness-gui)（private）

このリポジトリはドキュメントのみを扱う方針（`CLAUDE.md` §1）のため、
一度こちらに立てた #5〜#11 は移管済みとしてクローズした。

| ID | Issue（GUI リポジトリ） | タイトル | SP | 依存 | 状態 |
|---|---|---|---|---|---|
| G-1 | [gui#1](https://github.com/saniman/walover-line-harness-gui/issues/1) | ローカルサーバーの土台 | 2 | — | ✅ **完了** |
| G-2 | [gui#2](https://github.com/saniman/walover-line-harness-gui/issues/2) | フォーム UI（入力項目・検証・伏字） | 2 | gui#1 | 未着手 |
| G-3 | [gui#3](https://github.com/saniman/walover-line-harness-gui/issues/3) | **PTY 起動＋プロンプト契約に基づく応答** | 3 | gui#1 | 未着手 |
| G-4 | [gui#4](https://github.com/saniman/walover-line-harness-gui/issues/4) | 出力のストリーミング表示とシークレット伏字化 | 2 | gui#3 | 未着手 |
| G-5 | [gui#5](https://github.com/saniman/walover-line-harness-gui/issues/5) | CLI バージョンの固定と不一致警告 | 1 | gui#3 | 未着手 |
| G-6 | [gui#6](https://github.com/saniman/walover-line-harness-gui/issues/6) | **手順書の埋め込み** | 2 | gui#2, #3（本リポジトリ・完了） | 未着手 |
| G-7 | [gui#7](https://github.com/saniman/walover-line-harness-gui/issues/7) | 失敗時のステートファイル検知と警告 | 1 | gui#3 | 未着手 |
| **計** | | | **13** | | |

```
gui#1 ─┬─ gui#2 ──── gui#6（本リポジトリ #3 にも依存）
       └─ gui#3 ─┬── gui#4
                 ├── gui#5
                 └── gui#7
```

#### 全 Issue 共通の制約

- **本家 CLI のロジックを再実装しない**
- **シークレットをディスク・ログ・DB・localStorage に保存しない**（メモリのみ）
- **すべて参加者の PC 内で動く**。WALOVER のサーバーを経由させない
- サーバーは `127.0.0.1` にのみ bind する

#### 特に注意する Issue

**gui#3（PTY＋プロンプト契約）が最難関かつ最も危険。**
順番だけでプロンプトにマッチさせると、本家のバージョンアップでプロンプトが1つ増えただけで
値が1つずつズレる。最悪のケースは**チャネルシークレットがプロジェクト名の欄に入る**ことで、
その値は画面にエコーされ Worker 名として Cloudflare に残る。機能不全ではなく**情報漏洩**になる。

→ **文言を照合し、一致した場合のみ自動応答する。未知のプロンプトには答えず人間に委ねる**（フェイルクローズ）。

#### スケジュールへの影響

| | SP |
|---|---|
| 当初の配布キット | 10 |
| GUI ラッパー | **13** |
| 合計 | **23** |

**GUI は10月上旬の勉強会には間に合わない。**

→ **今回の勉強会は手順書で実施し、GUI は次回に向けて作る。**
　 手順書はすでに4ページ公開済みで、#1 の通し実行で実績がある（1時間以内で完走）。

---

### #1 完了（2026-09-07）

記録: [`docs/findings/cli-walkthrough.md`](findings/cli-walkthrough.md)

**結論: 配布可能。** 本家 CLI は無料枠のまま完走し、疎通確認まで通った。
**所要時間は全体で1時間以内**（実測者は開発者）。
→ 「#1 の結果が悪ければデモのみ＋次回配布に切り替える」という分岐は**不要**。当初計画どおり進める。

主な成果:

| | 内容 |
|---|---|
| 重点確認① migration | 🟡 正常系のみ通過。異常系は未検証 |
| 重点確認② 再開機構 | ✅ **実地で検証**。デプロイ失敗後の再実行で D1・R2 が作り直されず、LINE の5値も再入力不要 |
| 重点確認③ `[assets]` | ✅ `dist/client` / `ASSETS` / `run_worker_first` を確認 |
| 重点確認④ 各 step 実装 | ✅ サブドメイン確保・管理画面デプロイ・認証すべて機能 |
| 手作業インベントリ | **13件**（M-1〜M-7 準備 / P-1〜P-6 仕上げ）を確定 |
| 本家 Issue 候補 | **B-1〜B-7**（未起票） |
| 手順書必須項目 | **C-1〜C-20** |

判明した主な罠:

- Apple Silicon で x86_64 Node だと依存インストールが **SIGILL** で落ちる
- **カード登録済みでも R2 は別途有効化が必要**（code 10042）
- cron は1インストール2本・無料枠5本 → **同一アカウントに2つまで**
- LIFF の友だち追加オプションは既定 `On (Normal)`、要 `On (Aggressive)`
- Callback URL 未登録だと **PC からの友だち追加が無言で失敗**
- CLI は **実 TTY を要求**する（パイプ経由の stdin では起動しない）

配布前に必須:

1. 手順書「3. セットアップツールの実行」ページの作成（#3 の残り）
2. **#4（非開発者による検証）** — 1時間以内は開発者の数字。参加者には **2〜3時間**を提示する

---

## 4. スケジュール

| 1日の作業時間 | 完了目安 |
|---|---|
| 6時間 | 約 1.5 週間 |
| 3時間 | 約 3 週間 |
| 1.5時間 | 約 5〜6 週間 |

**10月上旬の勉強会には、1日1.5時間でも間に合う。**

> **GUI ラッパー（SP13）は上表に含まない。** 合計 SP23 となり10月上旬には間に合わないため、
> **今回は手順書で実施し、GUI は次回**に向けて作る。

進め方の推奨:
1. まず **#1（スパイク）** を1日で通す — ここで「本家CLIが今そのまま使えるか」が確定する
2. #1 の結果が悪ければ、その時点で「デモのみ＋次回配布」に切り替える判断ができる
3. #1 と並行して **#2（サポートポリシー）** を書く — 他の作業に依存しない

---

## 5. 未決の項目

- [ ] **このリポジトリのライセンス表記**（手順書・ポリシーは WALOVER の著作物）。MIT / CC BY / All rights reserved のどれにするか未決。public リポジトリなので配布前に決める
  - 判断材料: 前回勉強会の `saniman/line-harness-workshop` は **MIT（Copyright 2026 WALOVER LLC）** で公開済み。同種の資料なので揃えるのが自然
- [ ] 勉強会の具体的な日程
- [ ] 参加者への配布方法（このリポジトリの URL を渡すのか、LINE で配るのか）
- [ ] サポート用 LINE グループの作成と招待導線

## 6. 関連

- 本家: https://github.com/Shudesu/line-harness-oss
- 本家 CLI（npm）: `create-line-harness`
- WALOVER Fork（既存顧客専用・**配布対象外**）: https://github.com/saniman/line-harness-oss
- 前回勉強会の公開資料（WALOVER 自身の成果物・MIT）: https://github.com/saniman/line-harness-workshop
- **GUI ラッパー（コード・private）**: https://github.com/saniman/walover-line-harness-gui
