---
title: "Claude Skills と Cursor rules を作っているうちに、AI学習SaaSになった話（OSSとして公開しました）"
emoji: "🪄"
type: "tech"
topics: ["claudecode", "cursor", "ai", "個人開発", "saas"]
published: true
---

:::message
この記事は CodeSensei というAI学習SaaSの「設計の起源」と、それを **OSS の Claude Skill / Cursor rules として公開した話** です。母艦記事は [プログラミング未経験の事務員だった僕が、AI学習SaaSを作るまでの話](https://zenn.dev/ze1ny/articles/story-zero-to-codesensei) にあります。
:::

## TL;DR

- 自分の学習用に作っていた **「技術書を読むための Claude Skill ライブラリ」** が、ある日 SaaS化のヒントになった
- 6日で実装、ローンチして 12日後に GitHub OAuth が壊れていることに気づいた（[別記事](https://zenn.dev/ze1ny/articles/ai-overtrust-12days-broken)）
- そして今日、その**核となる Skill / Cursor rules を OSS（CC BY 4.0）で配布**しました
- ダウンロードURL: https://codesensei-iota.vercel.app/ja/skill

Claude Code / Cursor を使っている個人開発者が、自分のプロジェクトに置くだけで **AIが「コードレビュアー」から「学習パートナー」に変わる** 設定ファイル一式です。

---

## なぜこれを作ったか

僕は2026年4月15日に [CodeSensei](https://codesensei-iota.vercel.app) という AI学習SaaSをローンチしました。
ただ、このプロダクトは「最初から SaaSを作ろう」と思って作ったものではありません。

元になっているのは、**自分が技術書を学ぶために週末に作っていた、小さな Claude Skill ライブラリ** です。

### 最初の Skill — 挫折した『リーダブルコード』を学び直す

技術書って、一度読んだだけでは身につきません。
僕も例に漏れず『リーダブルコード』を読み終えて「なるほど」と本棚にしまい、3ヶ月後にはほぼ忘れていました。

そこで、こんな Skill を Claude Code 用に書きました（雰囲気を伝えるための擬似コード）:

```markdown
# `.claude/CLAUDE.md`

## あなたの役割

あなたは『リーダブルコード』の解説者です。
ユーザーがコードを貼り付けたら、その本の知識を踏まえて
「このコード、リーダブルコード的にはどう読めるか」を解説してください。

## 注目してほしい原則

- 名前から情報が伝わるか
- ループや if 文の構造が読みやすいか
- 変数のスコープは適切か
- コメントは「なぜ」を語っているか

## トーン

「ダメ出し」ではなく「学びのポイント」として伝える。
```

そして、**自分で書いたコード**（業務アプリの一部とか、過去の練習コード）を Claude に渡して解説してもらう。

これが、効きました。

「あ、自分の `tmp` という変数名は、こういう理由で問題なんだ」
「ループの中で同じ計算が繰り返されてる、って指摘されてる」

**本を読み直すより、ずっと頭に入った** 理由はシンプルで、**自分のコードが題材** だから。
他人事じゃないから、忘れません。

### 次々と作っていった

『リーダブルコード』で効果を実感してから、僕は本棚にあった技術書を片っ端から Skill 化していきました。

- Clean Code（第3章「関数」、第6章「オブジェクトとデータ構造」）
- Refactoring（Fowler のカタログ: Extract Method、Replace Conditional 等）
- SQL Antipatterns（N+1問題、ナイーブな木構造、暗黙のカラム）
- Site Reliability Engineering（SLI/SLO、エラーバジェット）
- Designing Data-Intensive Applications（一貫性モデル、レプリケーション）

最終的に **15ドメイン分の Skill ライブラリ** が手元にできていました。

毎週末に１つ Skill を増やして、業務で書いたコードを貼り付けて、AI と一緒に学ぶ。
そのループが、地味にじわじわと、僕のプログラミング理解を底上げしてくれました。

---

## SaaS化のきっかけ

3月のある日、X（Twitter）で駆け出しエンジニアの人がこんな投稿をしていました。

> 『リーダブルコード』読んだけど、結局なにを意識して書けばいいか分からない。
> AIにコード書いてもらってるけど、なんで動いてるかも理解できてない。

それを読んで、ハッとしました。**「これ、3ヶ月前の僕じゃん」**

そして思いました。
僕がやっている「自分用 Skill ライブラリで技術書を学び直す」アプローチ、もしかして**プロダクトとして他の人にも届けられるんじゃないか**。

そこから6日で、SaaS としての CodeSensei をローンチしました。
技術スタックは **Next.js 16 + Supabase + Claude API + Vercel + Stripe**。
コード生成のほとんどは Claude Code に任せましたが、**「どの Skill を内蔵するか」「本同士をどうつなげるか」「学びの流れをどう設計するか」** という核は、自分で考えました。

ここは、長い時間 Skill を作ってきたからこそできた判断でした。

---

## なぜ OSS で配ることにしたか

ローンチから12日経ち、ふと思いました。

**Skill の中身そのものは、別に独占する価値がない**。

CodeSensei の本当の価値は、SaaS としての:

- 188レッスンの体系的な進行
- 学習履歴の管理
- 複数本の概念をつなぐクロスブック
- AI との対話のオーケストレーション

であって、**「Skill の中身（=学びの観点をどう与えるか）」自体は、配布した方が世の中に貢献できる**。

それに、自分が個人開発で AI 補助を活用してきた立場として、「自分用の Skill / rules を整えると AI 開発体験が劇的に変わる」ということを **Claude Code / Cursor ユーザーにもっと知ってほしい** 気持ちがあります。

そこで今日、**Skill ファイルを OSS 化**しました。

---

## 配布物

https://codesensei-iota.vercel.app/ja/skill

から2種類ダウンロードできます。

### 1. `CLAUDE.md`（Claude Code 用）

Claude Code のプロジェクトルートまたは `.claude/` 配下に置くと、Claude Code が「学びの観点」でコードを読むようになります。

中身は次のような構造です（抜粋）:

```markdown
## 解説の柱（5領域）

ユーザーがコードを貼り付けたとき、以下5つの観点でだけ解説してください。
すべて触れる必要はなく、そのコードに最も関連する1〜2つだけを深く扱う方が良いです。

### 1. コード品質（Clean Code 系）
- 名前から意図が伝わるか（変数名・関数名）
- 関数は1つのことだけしているか
- マジックナンバー / マジックストリングはないか
- コメントは「なぜ」を語っているか

### 2. 設計・アーキテクチャ（SOLID / DRY / KISS）
- 単一責任原則
- DRY 違反 — 同じロジックを2回以上書いていないか
- KISS — シンプルにできないか

### 3. データベース・データアクセス
- N+1 問題
- インデックス
- トランザクション
- 正規化

### 4. Web / API設計
- REST 的に一貫したエンドポイント設計か
- 環境変数の取り扱い（コードに直書きしていないか、末尾改行に注意）
- レート制限・リトライポリシー

### 5. テスト容易性
- 副作用を関数に閉じ込めているか
- 外部依存を注入できるようになっているか
```

トーンの指定も明記しています:

```markdown
## 守るべきトーン
- ❌ 「このコードはダメです」「悪いコードです」
- ❌ 専門用語を説明なしに連発
- ✅ 「このコード、動きますね。さらに学べる部分があります」
- ✅ 専門用語を使ったら必ず1行で言い換える
```

このトーンルールが、地味だけど大事な部分です。
**「ダメ出し」と「学びの提示」は出力構造が全然違う** ので、Skill レベルで仕切らないと、Claude は容赦ないレビュアーモードに振れがちです。

### 2. `.cursorrules`（Cursor 用）

Cursor のプロジェクトルートに `.cursorrules` として置くと、同じ「学習パートナー」モードになります。
内容は英語版で、Claude Code 用と同じ5領域・同じトーンルール。

```text
You are a learning partner, not a code reviewer.
When the user asks for help with their code, your role is to help them
understand WHY good code is good, not just point out what's wrong.

## Tone
- Casual, friendly. Like a senior who shares the same struggles.
- Avoid "this is bad code" / "this is wrong" framings.
- Use "this works, and here's a learning opportunity" framings.
- Define jargon in plain language the first time you use it.
```

英語にしたのは、Cursor のグローバル開発者層に届けるためです。

---

## なぜ「OSSとして配る」が悪手じゃないか

「SaaS の中核を OSS で配ったら、ユーザーが本体に来なくなるのでは？」と思うかもしれません。
実は逆で、僕は**この配布が SaaS の到達点を増やす**と読んでいます。

理由は3つ:

1. **Skill / rules は静的なテキスト** — 一度書けば差別化ポイントにならない（誰でも作れる）
2. **SaaS の本当の価値は動的な部分** — 188レッスンのオーケストレーション、進捗管理、複数本の概念のクロスリンク。これは Skill だけでは再現不可
3. **Claude Code / Cursor ユーザーに「あ、これ良い」と思ってもらえれば、CodeSensei 本体への入り口になる** — Free プラン（月 50 回 AI 会話）から試せます

OSS の Skill が「サンプル」、SaaS が「フルセット」、という関係です。

---

## Claude Code / Cursor ユーザーへ

もしあなたが Claude Code や Cursor で開発していて、まだ Skill / rules を整えていないなら、まず上記をダウンロードして自分のプロジェクトに置いてみてください。**`.claude/CLAUDE.md`** を1つ置くだけで、Claude Code との対話がガラっと変わります。

そして、Skill のテンプレを編集して**自分の業務領域に合わせる** のもおすすめです。例えば:

- 自社サービスのドメイン用語を覚えさせる
- 特定ライブラリ（自社製や OSS）の利用規約を埋め込む
- コードレビューのチェックリストを共有する

Skill は **「AI に対する社内ドキュメント」** のような立ち位置です。一度書けば、チーム全員の AI 体験が揃います。

---

## まとめ

- 自分用に作った Claude Skill ライブラリが、結果として SaaS の核になった
- ローンチ後、その核を OSS で配布することにした
- Claude Code 用 `CLAUDE.md` と Cursor 用 `.cursorrules` を CC BY 4.0 で公開
- ダウンロード: https://codesensei-iota.vercel.app/ja/skill

「自分のために作ったツール」と「他人にも届けるプロダクト」の境界線は、思ったより薄いです。
あなたが今、Claude Code や Cursor のために整えている Skill / rules も、もしかしたら誰かのプロダクトの種かもしれません。

---

### 関連記事
- 母艦: [プログラミング未経験の事務員だった僕が、AI学習SaaSを作るまでの話](https://zenn.dev/ze1ny/articles/story-zero-to-codesensei)
- ローンチ後の事故: [AIに任せきりで作ったSaaSが、ローンチ12日間壊れていた話](https://zenn.dev/ze1ny/articles/ai-overtrust-12days-broken)
- [リファクタリング前に知るべき5つの原則](https://zenn.dev/ze1ny/articles/refactoring-5-principles)
- [個人開発SaaSを¥0で海外ローンチする完全ガイド](https://zenn.dev/ze1ny/articles/zero-yen-saas-launch)
- 🔗 CodeSensei: https://codesensei-iota.vercel.app
