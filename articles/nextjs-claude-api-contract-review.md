---
title: "Next.js + Claude API で契約書AIレビュー機能を実装した話"
emoji: "📝"
type: "tech"
topics: ["nextjs", "claude", "anthropic", "typescript", "ai"]
published: true
---

> 運送業向けの契約書管理SaaSに、Claude Sonnet 4.6 を使った「契約書AIレビュー」機能を1日で実装した記録です。法務の専門家がいない中小企業が、受け取った契約書の問題点を即座に把握できるようになります。

## この記事で分かること

- Next.js 16 の API Route で Claude API を呼ぶ方法
- 契約書レビューに特化したプロンプトの設計
- 構造化されたJSON出力を安定して得るコツ
- 有料プラン限定機能のゲーティング実装
- Adaptive Thinking の使い方

## 背景：なぜ契約書AIレビューが必要だったか

僕は個人開発で、中小運送会社向けの「傭車契約電子化SaaS」を作っています。

2025年4月に改正貨物自動車運送事業法が施行されて、傭車先との契約で書面交付が義務化されました。このSaaSでは契約書の一括生成→送付→承諾追跡まで一気通貫でできるんですが、実際に運用してみて気づいたことがあります。

**取引先から送られてくる契約書のレビューで、社内稟議が止まる。**

中小運送会社には法務部がない。社長か配車担当が「これ、ハンコ押して大丈夫？」と悩んで、結局そのまま放置。29社に送った契約書のうち、返信が遅い会社の多くが「社内精査中」と言っていました。

これ、AIで解決できるんじゃないか？と。

## 技術スタック

```
Next.js 16.2.1 + React 19
TypeScript 5
@anthropic-ai/sdk (Anthropic公式SDK)
Claude Sonnet 4.6（Adaptive Thinking有効）
Supabase (認証・DB)
Upstash Redis (レート制限)
Vercel (ホスティング)
```

## Step 1：Anthropic SDK のインストール

```bash
npm install @anthropic-ai/sdk
```

環境変数に `ANTHROPIC_API_KEY` を追加。以上。驚くほどシンプル。

## Step 2：プロンプト設計 — ここが一番大事

契約書レビューのプロンプトで意識したのは3点です。

### ① 「誰が読むか」を明確にする

```text
あなたは運送業界に精通した契約書レビューの専門家です。
中小運送会社の担当者（法務の専門知識がない方）に向けて、
分かりやすい日本語でレビューを行います。
```

「法務の専門知識がない方」がキー。これがないとClaudeは法律用語だらけの回答をしてきます。

### ② 業界特有のチェック項目を具体的に指定する

汎用的な「契約書をレビューして」ではなく、法定6項目を明示します。

```text
## 法定記載事項チェック（改正貨物自動車運送事業法 第12条・第24条）
必須6項目の有無を確認：
- 運送役務の内容及び対価（運賃）
- 荷役・附帯業務の内容及び対価
- 特別費用（高速代・燃料サーチャージ等）
- 契約当事者の名称及び住所
- 運賃・料金の支払方法
- 書面の交付年月日
```

さらに「不利条項の検出」「欠落条項の指摘」も具体的に列挙します。

```text
## 不利条項の検出
- 損害賠償が一方的に過大
- 遅延損害金の利率が高すぎる（年14.6%超）
- 解約条件が一方的（相手だけ即時解約可等）
- 免責範囲が広すぎる
- 運賃改定の拒否権がない
- 荷待ち時間の料金規定がない
```

業界固有の法律・基準を入れることで、汎用AIレビューとの差別化になります。

### ③ JSON出力フォーマットを厳密に指定する

```text
## 出力フォーマット
以下のJSON形式で出力してください。JSON以外のテキストは出力しないでください。

{
  "grade": "A" | "B" | "C" | "D",
  "grade_label": "問題なし" | "軽微な修正推奨" | "修正必須" | "重大な問題あり",
  "summary": "2-3文の総合所見",
  "legal_items": [
    { "name": "項目名", "found": true/false, "detail": "説明" }
  ],
  "risks": [
    { "severity": "高" | "中" | "低", "title": "タイトル", "detail": "説明", "article": "該当条項" }
  ],
  "missing": [
    { "name": "欠落条項名", "importance": "必須" | "推奨" | "任意", "detail": "理由" }
  ],
  "suggestions": ["修正提案1", "修正提案2"]
}
```

**`JSON以外のテキストは出力しないでください`** が重要。これがないとClaudeは親切に前置きを入れてきて、パースに失敗します。

## Step 3：API Route 実装

```typescript
// src/app/api/review/route.ts
import { NextRequest, NextResponse } from 'next/server'
import Anthropic from '@anthropic-ai/sdk'

export const maxDuration = 60 // Vercelの実行時間上限

export async function POST(request: NextRequest) {
  // 認証チェック（Supabase）
  // レート制限チェック（Upstash Redis）
  // プランチェック（有料プランのみ）
  // ... 省略 ...

  const { text } = await request.json()

  const client = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY! })

  const response = await client.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 16000,
    thinking: { type: 'adaptive' },
    system: SYSTEM_PROMPT,
    messages: [
      {
        role: 'user',
        content: `以下の契約書をレビューしてください。\n\n${text}`,
      },
    ],
  })

  // テキストブロックだけ取得（thinkingブロックを除外）
  let resultText = ''
  for (const block of response.content) {
    if (block.type === 'text') {
      resultText = block.text
    }
  }

  // JSONを抽出（```json ... ``` で囲まれることがあるため）
  const jsonMatch = resultText.match(/\{[\s\S]*\}/)
  const result = JSON.parse(jsonMatch ? jsonMatch[0] : resultText)

  return NextResponse.json({ result })
}
```

### ポイント解説

**`thinking: { type: 'adaptive' }`**

Claude Sonnet 4.6 / Opus 4.6 の新機能です。Claudeが自分で「どれくらい深く考えるか」を判断します。契約書のような複雑な文書は深く考えてくれるし、単純なものは素早く返す。

以前の `budget_tokens` は非推奨になったので、新規コードでは `adaptive` を使いましょう。

**`max_tokens: 16000`**

レビュー出力は2000-4000トークン程度ですが、余裕を持たせています。ここをケチると出力が途中で切れます。

**`response.content` のフィルタリング**

`thinking` が有効だと `response.content` に `thinking` ブロックと `text` ブロックが混在します。`text` だけ取得するのがポイント。

### エラーハンドリング

SDKの型付き例外クラスを使います。文字列マッチングは❌。

```typescript
try {
  const response = await client.messages.create({...})
} catch (err) {
  if (err instanceof Anthropic.RateLimitError) {
    return NextResponse.json({ error: 'レート制限' }, { status: 429 })
  }
  if (err instanceof Anthropic.APIError) {
    return NextResponse.json({ error: err.message }, { status: 500 })
  }
  if (err instanceof SyntaxError) {
    // 稀にClaudeが壊れたJSONを返す → リトライを促す
    return NextResponse.json({ error: 'JSON解析失敗' }, { status: 500 })
  }
}
```

## Step 4：有料プラン限定にする

Claude API はトークン課金なので、無料プランで無制限に使われるとコストが爆発します。

**API側（サーバー）:**
```typescript
const plan = (org?.plan as string) || 'free'
if (plan === 'free') {
  return NextResponse.json(
    { error: 'AIレビューは有料プランで利用できます。' },
    { status: 403 }
  )
}
```

**UI側（クライアント）:**
Freeプランの場合はロック画面を表示し、「プランをアップグレード」ボタンで設定ページに誘導。

**API・UI両方でチェック**するのが大事。UI側だけだとcurl等で直接叩かれます。

## コスト感

Claude Sonnet 4.6 の場合：
- Input: $3/1M tokens
- Output: $15/1M tokens

契約書1通のレビュー：
- 入力: 約2,000-5,000トークン（契約書テキスト + システムプロンプト）
- 出力: 約1,000-3,000トークン（JSONレビュー結果）
- **1回あたり約 ¥3-8**

月100件レビューしても ¥300-800。有料プラン（¥3,980/月）の原価として余裕で回収できます。

## まとめ

- Next.js API Route + Anthropic SDK で、**1日で契約書AIレビューを実装**
- プロンプトに「業界知識 + 誰向けか + JSON出力形式」を明記するのが品質の鍵
- Adaptive Thinking で複雑な契約書は深く、単純なものは素早くレビュー
- 有料プラン限定にしてコスト管理（API + UI の二重チェック）

法務の専門家がいない中小企業にとって、「とりあえずAIに聞く」というステップがあるだけで意思決定が早くなります。完璧な法的判断は求めていなくて、**「ここ確認した方がいいですよ」を教えてくれるだけで十分**なんです。

:::message alert
AIによるレビューは参考情報です。法的な判断には専門家にご相談ください。
:::
