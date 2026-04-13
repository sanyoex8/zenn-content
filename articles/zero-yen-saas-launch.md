---
title: "個人開発SaaSを¥0で海外ローンチする完全ガイド — Product Hunt / Show HN 実践編"
emoji: "🚀"
type: "idea"
topics: ["saas", "producthunt", "marketing", "startup", "nextjs"]
published: false
---

:::message
この記事は [CodeSensei](https://codesensei-iota.vercel.app) のローンチ準備で実際にやったことをまとめたものです。広告費¥0、1人で全作業を完結しています。
:::

## なぜ海外ローンチか

個人開発 SaaS の最大の課題は **認知** です。どんなに良いプロダクトでも、知られなければ使われません。

日本の開発者コミュニティは Zenn / Qiita / Twitter で届きますが、海外にはそれぞれ「定番チャネル」があります：

| チャネル | 特徴 | コスト |
|---------|------|--------|
| **Product Hunt** | プロダクト発見の最大プラットフォーム | ¥0 |
| **Show HN** | Hacker News。技術者の品評会 | ¥0 |
| **Twitter/X** | バイラル。日英両方狙える | ¥0 |
| **Zenn/Qiita** | 日本の開発者に直接届く | ¥0 |

全部 **無料** です。必要なのはコピーライティングの時間だけ。

## 準備するもの一覧

ローンチ前に用意すべき素材を一覧にしました。全部テキストファイルで管理しています。

```
marketing/
├── launch.md          # Show HN + PH の投稿文・Q&A想定問答
├── twitter-bank.md    # Twitter投稿30本の貯金
├── zenn-article.md    # Zenn技術記事の下書き
├── kpi.md             # KPI定義・利益目標・判断基準
└── assets/
    └── gallery/       # PH用ギャラリー画像6枚（1270x760）
```

### 1. Product Hunt 用素材

| 項目 | 文字数制限 | 準備すること |
|------|-----------|-------------|
| Product名 | — | そのまま |
| Tagline | 60文字 | 1文でプロダクトの本質を伝える |
| Description | 500文字 | 機能一覧 + 差別化ポイント |
| Gallery | 6枚推奨 | 1270×760px、PNG |
| First Comment | 制限なし | Maker としての自己紹介 + フィードバック依頼 |
| Topics | 3つ | Developer Tools / AI / Education 等 |
| Logo | 正方形 | 240×240以上 |

**Tagline が最重要**。PH のフィードに流れた時、ここだけ見てクリックするかどうかが決まります。

良い Tagline の条件：
- 「何ができるか」が一瞬でわかる
- 具体的な数字が入っている
- 技術的すぎない

```
✅ Your code becomes your textbook — learn CS from 95 classics
❌ AI-powered learning platform with structured knowledge base
```

### 2. Show HN 用素材

Show HN の投稿は **プレーンテキストのみ**。画像も太字もリンクプレビューもない。

構成テンプレート：

```
Title: Show HN: [プロダクト名] – [一言説明]

Hi HN,

[1段落] なぜ作ったか（個人の動機）
[1段落] 何ができるか（機能の要約）
[1段落] 技術的な差別化ポイント
[1段落] スタック紹介
[箇条書き] フィードバックが欲しい点（3つ）

Live: [URL]
```

HN で伸びる投稿の共通点：
- **技術的に面白い選択がある**（「なぜ X ではなく Y を選んだか」）
- **正直である**（「まだ未完成だけど」「ここは妥協した」）
- **フィードバックを具体的に求める**（「使い心地どうですか？」ではなく「Free tier は月200回で十分か？」）

### 3. Gallery画像

PH の Gallery は **最初の1枚がサムネイル** になります。ここで勝負が決まる。

1枚目に入れるべき要素：
- プロダクト名 + ロゴ
- 1行のバリュープロップ
- 実際のUIスクリーンショット（モックアップより実物）

私は Next.js の `ImageResponse` API（`next/og`）で動的に生成しました：

```typescript
// /api/gallery/[slug]/route.tsx
import { ImageResponse } from "next/og";

export async function GET(request: Request) {
  return new ImageResponse(
    <div style={{ width: 1270, height: 760, display: "flex", ... }}>
      {/* JSXでレイアウト */}
    </div>,
    { width: 1270, height: 760 }
  );
}
```

Figma を開かなくても、コードだけで PH 品質の画像が作れます。

## ローンチのタイミング

### Product Hunt

- **曜日**: 火・水・木（月曜は低い、金曜は寿命が短い）
- **時刻**: 12:01 AM PT（= 16:01 JST）に自動ローンチ
- **提出**: 前日の 23:59 PT までに submit
- **24時間**: ローンチから24時間がランキング対象

### Show HN

- **曜日**: 火・水・木
- **時刻**: 8-10 AM ET（= 21-23 JST）
- **理由**: 米国東海岸の就業開始時間。目に止まりやすい

### 両方同日にやる理由

PH と Show HN を同日にぶつけると：
1. PH の upvote → Show HN のコメントで相互言及
2. 「今日 PH に出てた」という口コミ効果
3. 1日で集中的に対応できる（2日に分けると疲弊する）

## Twitter戦略

### 投稿バンク方式

30本の投稿を **事前に全部書いておく**。3カテゴリに分類：

| カテゴリ | 目的 | 本数 |
|---------|------|------|
| **Tips** | SEO + バリュー提供 | 10 |
| **Feature** | プロダクト紹介 | 10 |
| **Before/After** | 共感 + 問題提起 | 10 |

投稿スケジュール：
- 平日朝7時: Tips（通勤時間帯）
- 平日昼12時: Feature（昼休み）
- 平日夜21時: Before/After（リラックスタイム）

30本 ÷ 1日2本 = **15日分のストック**。ローンチ後2週間はネタに困りません。

### Zenn記事 → Twitter の動線

Zenn記事は **Twitter の引用ツイートで告知** します。

```
[記事の要点を1-2行]

詳しくはZennに書きました👇
[URL]

#個人開発 #NextJS
```

記事の内容をそのまま書くのではなく、**最も刺さる1行だけ** 切り出してツイートにする。

## KPI の設計

ローンチで重要なのは「盛り上がったかどうか」ではなく、**数値で判断できること** です。

### 初動（Week 1）で見る指標

| 指標 | 目標 | 測定方法 |
|------|------|---------|
| PH upvotes | 50+ | PH ダッシュボード |
| HN points | 20+ | HN |
| サインアップ | 100+ | Supabase |
| Zenn いいね | 30+ | Zenn |
| Twitter フォロワー | 50+ | X |

### Month 1 で見る指標

| 指標 | 目標 | 意味 |
|------|------|------|
| WAU（週次アクティブ） | 30+ | 定着しているか |
| AI会話数/ユーザー | 5+/月 | 価値を感じているか |
| レッスン完了率 | 20%+ | 学習が継続しているか |

## ¥0ローンチの限界と割り切り

正直に書くと、¥0でできないこともあります：

- **カスタムドメイン**: `.vercel.app` のまま（ドメイン代は¥0ではない）
- **デモ動画**: 編集ソフトを持っていないので Gallery 画像で代替
- **プレスリリース**: 個人開発に PR TIMES は過剰
- **インフルエンサーマーケ**: 知り合いがいないのでコールドDMのみ

割り切るべきは割り切って、**テキストの力だけで勝負する** のが¥0ローンチの本質です。

## チェックリスト

最後に、ローンチ前の確認リストです：

```
[ ] ロゴ PNG（正方形、240px以上）
[ ] Gallery 画像（1270×760、6枚）
[ ] PH Tagline（60文字以内）
[ ] PH Description（500文字以内）
[ ] PH First Comment（Maker紹介）
[ ] Show HN タイトル（80文字以内）
[ ] Show HN 本文
[ ] Q&A想定問答（5-6問）
[ ] Twitter bio + pinned tweet
[ ] Twitter投稿バンク（最低10本）
[ ] Zenn記事（先行公開用）
[ ] 英語ページの表示確認
[ ] API予算アラート設定
[ ] Free tier の動作確認
[ ] Analytics 導入済み
```

## まとめ

個人開発 SaaS の海外ローンチは、お金ではなく **準備の質** で決まります。

1. **素材を全部テキストファイルで管理** — コピペで即投稿
2. **投稿バンク方式** — ネタ切れしない
3. **PH + HN 同日ローンチ** — 集中投下
4. **KPI を先に決める** — 感情ではなく数値で判断

これを読んでいるあなたが個人開発者なら、¥0で海外に出すことは **今日から始められます**。

---

CodeSensei は 4/15 に Product Hunt + Show HN で同時ローンチします。応援していただけると嬉しいです 🙏

https://codesensei-iota.vercel.app

https://zenn.dev/ze1ny/articles/234e0935faf940

