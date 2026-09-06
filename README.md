# WALOVER LINE Harness 配布キット

WALOVER 勉強会の参加者が、**自分の Cloudflare アカウント・自分の LINE 公式アカウント**で
LINE Harness（LINE公式アカウント向けのオープンソース CRM）を立ち上げるための手順書とサポート情報を置くリポジトリです。

📖 **ブラウザで読む: <https://saniman.github.io/walover-line-harness-kit/>**

## 手順書

| 手順 | 状態 |
|---|---|
| [1. Cloudflare の準備](docs/setup/cloudflare.md) | ✅ 公開中 |
| [2. LINE 側の準備](docs/setup/line.md) | ✅ 公開中 |
| 3. セットアップツールの実行 | 🚧 準備中（[#3](https://github.com/saniman/walover-line-harness-kit/issues/3)） |
| [4. 仕上げの設定](docs/setup/finish.md) | ✅ 公開中 |

> 全体の公開は勉強会の開催前を予定しています。

---

## このリポジトリは何で、何ではないか

| | |
|---|---|
| **ここにあるもの** | セットアップ手順書 / サポートポリシー / 動作検証の記録 |
| **ここに無いもの** | LINE Harness 本体のコード |

**ツール本体は WALOVER が作ったものではありません。**
本家 [Shudesu/line-harness-oss](https://github.com/Shudesu/line-harness-oss)（MIT）が公開している
`create-line-harness` を、そのまま使います。

```bash
npx create-line-harness@latest setup
```

WALOVER が提供するのは、**この1行の前後で必要になる LINE 側の設定手順**と、
**セットアップでつまずいたときのサポート**です。

## シークレットは預かりません

セットアップは参加者自身の PC で実行され、
LINE のチャネルシークレット・アクセストークン・API キーは
**参加者自身の Cloudflare アカウント**にのみ保存されます。

WALOVER がこれらを受け取ることも、保管することもありません。
質問の際も、**シークレットは絶対に貼らないでください**。

## サポート範囲

配布日から **3ヶ月**・**セットアップに関するもののみ**・**LINEグループでの質問対応のみ**。

詳細は準備中です（[#2](https://github.com/saniman/walover-line-harness-kit/issues/2)）。

> 本家のコードに起因する不具合は WALOVER では修正できません。
> その場合は [本家のリポジトリ](https://github.com/Shudesu/line-harness-oss/issues) へご報告いただく形になります。

## 計画

準備の進め方は [`docs/plan.md`](docs/plan.md) にまとめています。

## 運営

WALOVER合同会社（沖縄県うるま市）
