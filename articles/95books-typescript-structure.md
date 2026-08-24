---
title: "95冊の技術書をTypeScriptの構造化データにした方法"
emoji: "📖"
type: "tech"
topics: ["typescript", "設計", "技術書", "AI", "個人開発"]
published: true
---

:::message
この記事は [CodeSensei](https://codesensei-shunnosuke-uxs-projects.vercel.app) の知識ベース構築過程の記録です。
:::

## 問題: 技術書の知識は「散らばっている」

エンジニアなら本棚に数十冊の技術書があるはず。でも、こんな経験はないですか？

- 「この概念、どの本に書いてあったっけ？」
- 「Clean Code と Refactoring で同じこと言ってるけど、微妙に切り口が違う」
- 「SOLID原則の説明、3冊の本にバラバラに載ってる」

技術書の知識は **孤立している**。書籍Aで学んだことと書籍Bで学んだことが、頭の中で繋がっていない。

CodeSensei を作る時に最初にやったのは、95冊の技術書の知識を **構造化データ** にすることでした。

## 構造の設計

### 4層のデータモデル

```
Books（95冊）
  ↓ 多対多
Courses（15コース）
  ↓ 一対多
Lessons（188レッスン）
  ↓ 多対多
CrossbookConnections（概念間の接続）
```

TypeScript の型定義：

```typescript
// books.ts
export type Book = {
  id: string;           // "clean-code"
  title: string;        // "Clean Code"
  author: string;       // "Robert C. Martin"
  essence_ja: string;   // 一言で表す本の本質
  essence_en: string;
  field_ja: string;     // 分野（コード品質、設計原則...）
  field_en: string;
};

// courses.ts
export type Course = {
  id: string;           // "code-quality"
  name_ja: string;      // "コード品質コース"
  name_en: string;
  description_ja: string;
  description_en: string;
  icon: string;         // lucide-react のアイコン名
  color: string;        // テーマカラー
  bookIds: string[];    // このコースの基盤書籍
};

// lessons.ts
export type Lesson = {
  id: string;           // "naming-things"
  courseId: string;      // "code-quality"
  level: number;        // 1-4（基礎→発展）
  lessonNumber: number;
  title_ja: string;     // "名前の付け方"
  title_en: string;     // "Naming Things"
  coreConcept_ja: string;
  coreConcept_en: string;
  sourceReference: string;  // "Clean Code Ch.2"
  importance: "required" | "important" | "culture";
  estimatedMinutes: number;
};
```

### なぜ DB ではなく TypeScript か

Supabase にマスタデータとしてテーブルを作ることもできました。しかし TypeScript にした理由は3つ：

**1. 型安全**: 存在しない書籍IDを参照したら **コンパイルエラー** になる

```typescript
// ❌ DBだとランタイムまで気づかない
const book = await supabase.from("books").select().eq("id", "cleean-code"); // タイポ

// ✅ TypeScriptだとエディタが即座に教えてくれる
const book = BOOKS["cleean-code"]; // Property does not exist
```

**2. バンドルサイズ**: 95冊 × 数フィールド = 数十KB。DB クエリのラウンドトリップよりインライン展開の方が速い

**3. プロンプト構築**: AI のプロンプトに書籍情報を含める時、TypeScript オブジェクトからそのまま文字列化できる

```typescript
const courseBooks = course.bookIds
  .map(id => BOOKS[id])
  .map(b => `『${b.title}』by ${b.author}: ${b.essence_ja}`)
  .join("\n");

const prompt = `以下の書籍の知見をベースに教えてください:\n${courseBooks}`;
```

## 15コースの設計基準

95冊を15の分野に分類しました：

| コース | 代表書籍 | レッスン数 |
|--------|---------|-----------|
| Code Quality | Clean Code, Readable Code | 18 |
| Web & API | Web API Design, HTTP Definitive Guide | 14 |
| Security | OWASP, Secure Coding | 10 |
| Database | SQL Antipatterns, DDIA | 14 |
| Frontend | DOM Enlightenment, CSS Secrets | 11 |
| Testing | TDD, Growing OO Software | 12 |
| Architecture | Clean Architecture, DDD | 15 |
| Infrastructure | Docker Deep Dive, Kubernetes | 10 |
| DevOps | SRE Book, Continuous Delivery | 10 |
| Algorithms | CLRS, Grokking Algorithms | 14 |
| OS & Low-Level | OSTEP, CSAPP | 12 |
| Performance | High Performance Browser | 10 |
| Agile & PM | Mythical Man-Month, Lean Startup | 12 |
| Soft Skills | Pragmatic Programmer, Staff Engineer | 12 |
| Git | Pro Git | 14 |

### 分類の判断基準

1. **1コースに2冊以上の書籍が紐づくこと** — 1冊だけなら独立コースにする意味がない
2. **レッスンが8つ以上になること** — 少なすぎるとコースとして成立しない
3. **レベル1（基礎）から始められること** — 初心者が入り口を見つけられるように

## クロスブック接続

一番面白かったのがこの部分です。

「関心の分離（Separation of Concerns）」という概念は、以下の書籍で **異なる角度から** 語られています：

- **Clean Code**: 関数レベルの責任分離
- **Clean Architecture**: レイヤーレベルの依存性逆転
- **Domain-Driven Design**: ドメインとインフラの分離
- **SICP**: 抽象化の壁
- **Designing Data-Intensive Applications**: ストレージとクエリの分離

```typescript
export type CrossbookConnection = {
  concept: string;      // "Separation of Concerns"
  links: {
    fromBookId: string; // "clean-code"
    toBookId: string;   // "clean-architecture"
    description_ja: string;
    description_en: string;
  }[];
};
```

8つの中核概念について、書籍間の接続を定義しました。これにより「Clean Code を学んだ人が、次に Clean Architecture を学ぶと何が繋がるか」が可視化できます。

## 188レッスンの粒度

各レッスンは **15-20分で完結** する粒度に設計しました。

```
1レッスン = 1つの核心概念 + 1つの書籍参照 + AI会話1セッション
```

例: Code Quality コースのレベル1

| # | タイトル | 核心概念 | 参照 |
|---|---------|---------|------|
| 1 | 名前の付け方 | 明確で誤解されない名前 | Clean Code Ch.2 |
| 2 | コメントの技術 | コードで表現できないことだけ書く | Clean Code Ch.4 |
| 3 | ネストを浅く | 早期リターン、ガード節 | Readable Code Ch.7 |
| 4 | 巨大式の分解 | 説明変数、要約変数 | Readable Code Ch.8 |
| 5 | 関数設計 | 小さく、1つのことだけ | Clean Code Ch.3 |

### importance フィールド

全188レッスンに重要度を設定しました：

- **required**: 全員が学ぶべき（40%）
- **important**: 実務で頻出（40%）
- **culture**: 知っていると会話が広がる（20%）

カスタムカリキュラム生成時に、`required` を優先的に組み込む判断材料になります。

## 学んだこと

### 構造化は「理解の強制装置」

95冊を TypeScript に落とす作業は、実質的に **全冊の再読** でした。「この本の本質を1文で表すと？」「この概念は他のどの本と繋がる？」と自問しながら構造化すると、読んだ時より遥かに深い理解が得られます。

### 完璧を目指さない

最初のバージョンは `importance` フィールドも `crossbookConnections` もありませんでした。まず最小限の構造で動くものを作り、使いながら「ここが足りない」と気づいたら追加する。データ設計も MVP と同じです。

### 型は最高のドキュメント

`Book` 型を見れば、書籍データに何が含まれるか一目瞭然。JSDoc も README も不要。TypeScript の型定義が、そのまま仕様書になります。

---

CodeSensei では、この構造化データを使って AI がレッスンを組み立てます。あなたのコードを貼り付けると、95冊の書籍の中から関連する概念を引き出して解説してくれます。

https://codesensei-shunnosuke-uxs-projects.vercel.app

