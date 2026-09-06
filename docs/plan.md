# 配布キット 準備計画

> 作成日: 2026-09-07 / 対象: 次回勉強会（10月上旬想定）での配布
> ステータス: 決定済み・Issue 起票済み（#1〜#4）

---

## 0. 結論（3行）

1. 配布するツールは **本家 [Shudesu/line-harness-oss](https://github.com/Shudesu/line-harness-oss) の `npx create-line-harness@latest`**。WALOVER は自前のセットアップツールを作らない。
2. WALOVER が作るのは **手順書・サポートポリシー・検証記録の3点だけ**。SP 合計 **10**（実働 6〜8日）で、10月上旬の勉強会に間に合う。
3. WALOVER の Fork（`saniman/line-harness-oss`）は**既存顧客専用**として維持し、配布物には一切含めない。

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

### やらないこと

- LINE Harness 本体のコードを書く / 本家 CLI を改修する
  → 本家側の不具合を見つけたら **本家に Issue を立てる**
- WALOVER の Fork 固有機能（イベント予約 + Stripe 決済 / freee 領収書自動化 / モバイルオーダー / AI週次配信）を配布物に含める
- 参加者のシークレットを WALOVER 側で預かる仕組み

---

## 3. Issue 一覧

| ID | Issue | タイトル | SP | 依存 |
|---|---|---|---|---|
| 1 | [#1](https://github.com/saniman/walover-line-harness-kit/issues/1) | 調査: 本家最新 CLI を通し実行し、手作業が残る箇所と所要時間を実測する | 2 | — |
| 2 | [#2](https://github.com/saniman/walover-line-harness-kit/issues/2) | docs: サポートポリシー（3ヶ月・セットアップのみ・LINEグループのみ） | 2 | — |
| 3 | [#3](https://github.com/saniman/walover-line-harness-kit/issues/3) | docs: 参加者向けセットアップ手順書 | 3 | #1, #2 |
| 4 | [#4](https://github.com/saniman/walover-line-harness-kit/issues/4) | 第三者（非開発者）による通し検証 | 3 | #3 |
| **計** | | | **10** | |

SP の目安: **SP1 = 半日（2〜3時間） / SP2 = 1日 / SP3 = 2日**

### 依存グラフ

```
#1（スパイク）─┐
               ├── #3（手順書）── #4（第三者検証）
#2（ポリシー）─┘
```

`#1` と `#2` は独立なので並行できる。

### #1 の進捗（2026-09-07 時点）

記録先: [`docs/findings/cli-walkthrough.md`](findings/cli-walkthrough.md)

- ステップ0（CLI の表面仕様確認）完了。**Step 1「R2 有効化」の直前で停止中**
- 停止理由: R2 の有効化に**クレジットカード＋個人情報の登録**が必要。Aki の手作業待ち

判明した主なもの:

| | 内容 |
|---|---|
| 環境の罠 | Apple Silicon + x86_64 Node だと依存インストールが **SIGILL** で落ちる。arm64 Node で解決（57秒で成功）。**前回勉強会でも別原因と誤認して踏んでいた** |
| `--help` が無い | `parseArgs` に分岐が無く、未知のハイフン引数は黙って無視 → **既定の `setup` が走り、Cloudflare 認証まで進む**。本家 Issue 候補（温度感: 高） |
| TTY 必須 | `@clack/prompts` が実 TTY を要求。**パイプ経由の stdin 流し込みでは起動しない**（`uv_tty_init EINVAL`） |
| 入力項目 | 全13項目をソースから一次情報として抽出済み。前回資料に**「LINE Login チャネルの Channel ID」が抜けていた** |
| シークレット | `p.password` 未使用で**画面に平文表示**。`.line-harness-config.json` にも平文保存（`.gitignore` 済み） |
| 再開機構 | `completedSteps` を永続化し `reset` / `continue` / `abort` を選ばせる実装を確認（実挙動は未検証） |

---

## 4. スケジュール

| 1日の作業時間 | 完了目安 |
|---|---|
| 6時間 | 約 1.5 週間 |
| 3時間 | 約 3 週間 |
| 1.5時間 | 約 5〜6 週間 |

**10月上旬の勉強会には、1日1.5時間でも間に合う。**

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
