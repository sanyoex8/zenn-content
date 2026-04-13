---
title: "Next.js 16 + Supabase + Claude API で AI学習 SaaS を 2 日で作った話"
emoji: "📚"
type: "tech"
topics: ["nextjs", "supabase", "claude", "typescript", "ai"]
published: false
---

:::message
この記事は [CodeSensei](https://codesensei-iota.vercel.app) の開発記録です。個人開発で、着想から本番デプロイまで2日間で Phase 1〜3 + ユーザー獲得施策までを一気に構築しました。技術選定と「早く出す」ための意思決定を共有します。
:::

## 何を作ったか

**CodeSensei — あなたのコードが教科書になる AI 学習プラットフォーム**

- 15コース・188レッスン・95冊の技術書の知見を内蔵
- ユーザーの GitHub リポジトリ（or 手動ペースト）を題材に、Claude AI が技術書の概念を解説
- 「コードから学ぶ」「バグから学ぶ」「カスタムカリキュラム」など AI 機能を 5 種類搭載
- 日本語 + 英語の i18n、チーム機能、Stripe 決済まで実装済み

https://codesensei-iota.vercel.app

## なぜ作ったか

Claude Code や Cursor を使って開発していて、ある違和感がありました。

「コードは前より10倍速く書けるのに、『なぜそう書くか』の理解が薄くなっている」

本棚には『Clean Code』『リファクタリング』『SICP』『Designing Data-Intensive Applications』が並んでいるけど、読み切れたのは数冊。読んだはずの章も、実際のコードでは活かせていない。

問題は「本を読む動機」じゃなくて、**本の例と自分のコードが繋がっていない**ことでした。

だったら、自分のコードを本の例の代わりにしたらどうなる？

これが CodeSensei の発想です。

## 技術スタック

| 層 | 技術 | 理由 |
|---|---|---|
| Framework | Next.js 16 + React 19 | App Router、RSC、Edge Runtime、ISR を全部使いたい |
| Styling | Tailwind CSS 4 | CSS 変数ネイティブ統合 + @theme inline で DX が跳ね上がった |
| Auth | Supabase Auth | GitHub OAuth + メール、middleware との統合が素直 |
| DB | Supabase PostgreSQL | RLS で権限設計を DB 層に閉じ込められる |
| AI | Claude API (Sonnet 4 / Opus 4.6) | ストリーミング、長文理解力、コストのバランス |
| Cache/Rate Limit | Upstash Redis | Edge Runtime から叩ける、Vercel との相性 |
| i18n | next-intl | RSC 対応、URL based locale |
| Payments | Stripe | webhook の取り回しが安定 |
| Deploy | Vercel | Next.js との統合、プレビューデプロイ |

**選定の基準**: 自分が既に触ったことがあるか、ドキュメントが最新か、Edge Runtime で動くか。

新しい技術を学ぶ時間は 0 秒。全部「知ってるもの」で組み立てました。個人開発の鉄則です。

## 設計判断

### 1. 「コード品質チェック」ではなく「学び」を軸にする

CodeSensei は静的解析でコードの問題を指摘するツールではありません。

**目的**: ユーザーが「このコードから Clean Code の第 3 章が体感できる」と感じること。
**非目的**: ユーザーに「ここがダメ」と指摘すること。

この差は小さく見えて、プロンプト設計から UI コピーまで全部に影響します。

### 2. GitHub 接続を必須にしない

GitHub App の権限周りは複雑だし、「会社のコードは貼れない」人もいる。だから手動ペーストを第一級サポートにしました。

```tsx
// レッスン画面: コード入力も GitHub も同じレールに流す
<textarea
  value={codeInput}
  onChange={(e) => setCodeInput(e.target.value)}
  placeholder="コードを貼り付け..."
/>
```

### 3. AI モデルをユーザー選択制にする

Sonnet と Opus、両方使えるようにしました。

- **Sonnet 4**: デフォルト、日常レッスン
- **Opus 4.6**: 「深く学ぶ」ボタンで切り替え、複雑な概念向け

コスト差は5倍ですが、「ここぞ」で Opus を使うと解説の深みが段違い。ユーザーに選ばせることで、コストも自分でコントロールできます。

### 4. プロンプト階層化

トークンコストを抑えるために、プロンプトを 4 層に分けました。

```
1. ベースプロンプト（常時）      ~500トークン  役割 + ルール
2. コース固有（レッスン時）       ~1500トークン 書籍リスト + クロスブック接続
3. ユーザーコード（分析時）       可変          GitHub API で取得
4. バグマッピング（debug時）      可変          エラー種別→概念テーブル
```

全部毎回送るのではなく、モードに応じて必要な層だけ送る。これで 1 リクエストあたり数千トークンを削減しています。

### 5. 著作権対応

95 冊の書籍を題材にする以上、著作権の扱いは最初に固めました。

- 書籍タイトル・著者名 → **使用する**（事実情報、保護対象外）
- 章構成・目次 → **使わない**。独自カリキュラム体系で再構成
- 本文 → **引用しない**。概念を完全に自分の言葉で再解説
- 各レッスンに「もっと深く学ぶなら → 書籍アフィリエイトリンク」

著者にもメリットがある Win-Win 構造を意識しました。

## 2 日間の実装スケジュール

### Day 1: Phase 1 MVP

| 時間 | 作業 |
|---|---|
| 午前 | Plan mode で全体設計（ページ構成・DB・プロンプト階層） |
| 昼 | Supabase 17 テーブル + RLS + トリガー |
| 午後 | 知識ベース移行（TypeScript 構造化データ 94 冊 + 15 コース + 188 レッスン） |
| 夕方 | 認証・レイアウト・18 ルート |
| 夜 | AI 会話 API（ストリーミング + モデル選択）+ 初デプロイ |

Plan mode で先に全体設計を確定させたのが効きました。「途中で気が変わる」をゼロにできます。

### Day 2: Phase 2/3 + ユーザー獲得施策

| 時間 | 作業 |
|---|---|
| 午前 | バグから学ぶ / レベル診断 / 学習履歴 / クロスブック可視化 |
| 昼 | Stripe 決済 / カスタムカリキュラム / 料金ページ |
| 午後 | SEO / ブログ / アーリーアクセス特典 |
| 夕方 | チーム機能 / GitHub Webhook |
| 夜 | E2E テスト → バグ修正 4 件 → 本番確認 |
| 深夜 | Vercel Analytics / シェアボタン / ソーシャルプルーフ / ブログ 5 本 / ロゴ |

Phase をスプリントのように細かく区切って、1 サイクルごとに「ビルド → コミット → push → DEVLOG 記録」を回しました。

## ハマりどころ

### 1. RLS 無限再帰

チーム機能で `team_members` テーブルを参照する RLS を書いたら、循環参照で無限再帰エラー。

**解決**: `SECURITY DEFINER` 関数を作ってポリシー内から呼び出す形に変更。

```sql
CREATE FUNCTION user_is_team_member(team_id UUID)
RETURNS BOOLEAN
LANGUAGE sql
SECURITY DEFINER
SET search_path = public
AS $$
  SELECT EXISTS (
    SELECT 1 FROM team_members
    WHERE team_members.team_id = $1 AND user_id = auth.uid()
  );
$$;
```

### 2. RSC prefetch が 503

Next.js 16 の RSC prefetch が middleware で認証リダイレクトされて 503。

**解決**: middleware で RSC prefetch リクエストを検出して認証スキップ。

```ts
const isRscPrefetch = request.headers.get("rsc") === "1";
if (isRscPrefetch) return NextResponse.next();
```

### 3. ブログ個別ページが DYNAMIC_SERVER_USAGE エラー

マークダウンを動的読み込みしていたら、ビルド時に 500。

**解決**: マークダウンを TS モジュール化して静的インポート。`dynamic = "force-dynamic"` を併用。

### 4. Supabase 無料枠 2 プロジェクト制限

既存の車検アプリで 2 枠使い切っていたため、CodeSensei 用に 1 枠確保する必要がありました。

**解決**: 車検アプリの DB を Docker PostgreSQL に移行。Prisma マイグレーション適用 + データダンプ + インポートで完結。

## 95 冊の書籍をどう構造化したか

CodeSensei の核心は `src/lib/knowledge/` 以下の TypeScript 構造化データです。

```ts
// books.ts
export type Book = {
  id: string;
  title_ja: string;
  title_en: string;
  author: string;
  coreConcepts: string[];  // クロスブック接続の元
  estimatedReadTime: number;
};

// lessons.ts
export type Lesson = {
  id: string;
  courseId: string;
  level: 1 | 2 | 3;
  coreConcept_ja: string;
  sourceReference: string;
  relatedBooks: string[];
  crossBookConnections: string[];
};
```

もともと Claude Code の個人スキル `/study` として Markdown で管理していたものを、TypeScript に移植しました。型で縛ることで、プロンプト構築時の「存在しない書籍を参照」などのバグを防げます。

## 学んだこと

### 1. Plan mode は「思考を言語化するツール」

頭の中の設計を文章にすると、矛盾や見落としが見えてきます。Claude の Plan mode は承認前に「一緒に詰める」ことができるので、実装に入る頃には迷いがゼロでした。

### 2. Phase 分けすると迷いが消える

Phase 1 で絶対必要なものだけ、Phase 2 は補完、Phase 3 は収益化。この線引きをしておくと「やるかやらないか」の議論が不要になります。

### 3. 確認を挟まない開発リズム

「この実装進めてもいい？」と聞かずに、確認せず進めてコミット → ビルド → push のサイクルを止めない。テンポが命です。

### 4. DEVLOG にリアルタイムで書く

まとめて書こうとすると書けなくなる。実装のたびに 3 行書く。これで「2 日間の記録」が残せました。

## 今後

- **ユーザーテスト**: 初回の生のフィードバックが最重要
- **Show HN / Product Hunt ローンチ**: 英語圏への展開
- **tree-sitter + Semgrep**: 学びの種の自動検出（Phase 2 延長）
- **ブログ記事の量産**: SEO 流入強化
- **リファラル機能**: 既存ユーザーからのバイラル導線

## まとめ

個人開発で AI SaaS を作るなら、以下の 3 つを意識するといいかもしれません。

1. **知ってる技術だけで組む**: 新しい何かを学ぶ時間は 0
2. **Plan mode で先に全部決める**: 実装中の迷いを最小化
3. **小さく速く出す**: 「動いて本番に出ている」状態を最初に作る

CodeSensei は現在アーリーアクセス中で、Free プランで月 200 回 AI 会話が使えます。試してみて、感想をリプライや DM でいただけると嬉しいです。

https://codesensei-iota.vercel.app

読んでいただきありがとうございました 🙏

---

**関連リンク**
- [CodeSensei 本体](https://codesensei-iota.vercel.app)
- [GitHub (private)](https://github.com/sanyoex8/codesensei)
- [ブログ（CodeSensei 内）](https://codesensei-iota.vercel.app/blog)
