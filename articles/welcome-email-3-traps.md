---
title: "Welcomeメール1通を送るために踏んだ3つの罠 — Vercel / Redis / Resend"
emoji: "📬"
type: "tech"
topics: ["nextjs", "vercel", "resend", "redis", "個人開発"]
published: true
---

:::message
この記事は CodeSensei というAI学習SaaSの運用で、サインアップ後の welcome メールを実装する過程で踏んだ3つの罠の記録です。
:::

## TL;DR

サインアップ後のwelcomeメール、たかが1通、と思って組んだら3日で3回ハマりました。

1. **Vercel serverless で fire-and-forget の Promise が完了せず消える** → メール送信処理が走らない
2. **`check-then-set` 方式の dedupe で同時並行リクエストが両方とも貫通** → メールが2通届く
3. **Resend の `onboarding@resend.dev` は Resend アカウント登録メアド宛にしか送れない** → 送信成功なのに届かない

それぞれの罠と、最終的にどう解決したかをまとめます。

---

## 背景: 何を作りたかったか

[CodeSensei](https://codesensei-shunnosuke-uxs-projects.vercel.app) のサインアップ完了後、ユーザーに「最初の5分の歩き方」を案内するwelcomeメールを送りたい。要件は地味:

- Resend で送る
- 1ユーザーにつき1通だけ
- 送信失敗時はある程度 retry したい
- でも Resend のレート上限を圧迫しない

技術スタック: Next.js 16 (App Router) + Supabase Auth + Vercel + Resend + Upstash Redis。

---

## 罠 1: Vercel serverless での fire-and-forget Promise

最初の実装は典型的なクライアント側 fire-and-forget でした。

```ts
// auth/login/page.tsx (signup ハンドラ)
const { error } = await supabase.auth.signUp({ email, password });
if (error) throw error;

// fire-and-forget で welcome メール送信
fetch("/api/email/welcome", { method: "POST" }).catch(() => {});
router.push("/dashboard");
```

`/api/email/welcome` は authenticated user を確認して Resend で送信するルート。

**症状**: サインアップした人にメールが届かない。Resend のログにも履歴が一切ない。

### 原因

実は2段階で問題がありました。

**(a) クライアント側 fetch のタイミング**

`supabase.auth.signUp()` が成功して即座に `fetch()` を発行すると、Supabase の auth cookie がブラウザに完全書き込まれる前に request が飛ぶことがあります。

サーバー側ルートで `supabase.auth.getUser()` を呼ぶと、cookie がまだ無いから user = null になり、ルートは `{ skipped: "no-user" }` で 200 を返して終わる。**Resend は呼ばれない**。

**(b) Vercel serverless で fire-and-forget の Promise が消える**

これが本丸の方。サーバーサイドに移して dashboard server component から呼ぶように変更したものの、

```ts
// dashboard/page.tsx (server component)
maybeSendWelcomeEmail(user.id, user.email).catch(() => {});
return <Dashboard />;
```

これだと **Vercel serverless function は JSX を返した瞬間にレスポンスを送信し、Node のイベントループはまだ Promise が走っている途中でも次のリクエスト処理に移る or 関数インスタンスが終了する**。

結果、`maybeSendWelcomeEmail` の中の `redis.get` や `resend.emails.send` が完了する前に切られる。Resend には1通も届いていなかった。

### 解決

await する。これだけ。

```ts
// dashboard/page.tsx
try {
  await maybeSendWelcomeEmail(user.id, user.email, displayName);
} catch {
  // ダッシュボード描画は止めない
}
return <Dashboard />;
```

「ダッシュボード描画が遅くなるのでは」と思いがちですが、実測で +500ms-1s 程度。Redis dedupe が効いてからは初回のみのコストで、初回は新規ユーザーが welcome を待っているタイミングなので問題なし。

### 学び

**Vercel serverless では「副作用は完了まで await する」が原則**。background promise の生存は保証されない。

Next.js 15 以降は `unstable_after()` という API も用意されていて、これを使うとレスポンス送信後にタスクを continue できるが、現在は preview 機能。await が一番堅実。

---

## 罠 2: check-then-set dedupe で2通届く

await 化して送信が動き始めたら、今度は1人のユーザーに **同時に2通届く** 事態に。

最初の dedupe ロジック:

```ts
const key = `welcome_sent:${userId}`;

const already = await redis.get(key);
if (already) return;

const { error } = await client.emails.send({...});

if (error) {
  // 失敗時は短いTTLで retry を許す
  await redis.set(key, "failed", { ex: 60 * 60 });
  return;
}

await redis.set(key, "sent", { ex: 60 * 60 * 24 * 365 });
```

何が起きるか:

```
時刻 T0:  リクエストA → redis.get(key) → null
時刻 T1:  リクエストB → redis.get(key) → null  ← まだA は set してない
時刻 T2:  リクエストA → resend.emails.send() ← 1通目
時刻 T3:  リクエストB → resend.emails.send() ← 2通目
時刻 T4:  リクエストA → redis.set(key, "sent")
時刻 T5:  リクエストB → redis.set(key, "sent")
```

クラシックな **check-then-act の race condition**。

なぜ並行リクエストが起きるかというと、Next.js App Router の RSC は prefetch + actual render で同じコンポーネントが2回 render されることがあったり、ユーザーが2タブで開いたり、いろいろ。

### 解決: SETNX (atomic claim)

Redis の `SET key value NX` は **「キーが存在しない時だけ set する」** atomic オペレーション。先に "claim" してから副作用に進めば、後発のリクエストは即弾ける。

```ts
// 30秒の "pending" claim を atomic に取る
const claimed = await redis.set(key, "pending", { nx: true, ex: 30 });
if (!claimed) return;  // 別のリクエストが既に claim 済み

// ここから先は1リクエストのみ走る
const { error } = await client.emails.send({...});

if (error) {
  await redis.set(key, "failed", { ex: 60 * 60 });
  return;
}

await redis.set(key, "sent", { ex: 60 * 60 * 24 * 365 });
```

ポイント:

- `pending` の TTL は **30秒**（短い）。万一 send 中にクラッシュしてもキーが自動 expire し、次回訪問で retry できる
- 成功すれば `sent` で 365日に上書き、失敗すれば `failed` で1時間に上書き
- Upstash Redis の SDK では `redis.set(key, value, { nx: true, ex: 30 })` の形

### 学び

dedupe は **check-then-act ではなく claim-then-act で**。Redis の SETNX は原始的だが万能。

これは welcome メールに限らず、ジョブの重複起動防止 / 二重決済防止 など、広く効く考え方です。

---

## 罠 3: Resend の `onboarding@resend.dev` の謎の制約

ここまで来て、自分のメアド (`shunnosukenishimura@gmail.com`) に届かない問題。

実装は正しい。Vercel logs にも welcome 送信の log は出ている。なのにメールが来ない。Resend ログにも履歴が残らない。

### 原因

Resend のデフォルト送信元 `onboarding@resend.dev` は、**Resend アカウント登録メアドにしか送信できない** という仕様の制限。

> When using the @resend.dev domain, you can only send emails to the email address associated with your Resend account.

つまり Resend を `sanyoex8@gmail.com` で登録した場合、`onboarding@resend.dev` から送信できる宛先は `sanyoex8@gmail.com` のみ。それ以外（テスト時の `shunnosukenishimura@gmail.com` 含む）は **silently rejected** されます。

正確には silently じゃなくて、Resend の Logs ページに「Validation Error」が出ているはずなんですが、当時は気づかず。

### 解決: ドメイン verify

別プロジェクトで verify 済みのドメイン (`yousha.sanyo-transportation.com`) を再利用。

```bash
EMAIL_FROM_ADDRESS="CodeSensei <noreply@yousha.sanyo-transportation.com>"
```

display name に `CodeSensei` を入れることで、受信箱では「CodeSensei」と見える。@ ドメインが別プロジェクトのものでも実害なし。

Resend Free tier は 1ドメイン制限なので、追加で verify するなら Pro ($20/月)。新規ユーザー数が伸びてから判断。

### 学び

**Resend の `onboarding@resend.dev` はテスト用**。本番運用するなら最初から自前ドメインを verify すべき。

似た仕様は SendGrid, Mailgun, AWS SES (sandbox mode) にもあって、初期は「自分のメアドにしか送れない」がデフォルト。新規プロバイダ採用時は最初に確認推奨。

---

## まとめ — 1通送るのに必要だったこと

| 罠 | 解決 |
|---|---|
| fire-and-forget の Promise が消える | `await` する |
| check-then-set で並行2通送られる | Redis SETNX で atomic claim |
| `onboarding@resend.dev` が届かない | 自前ドメイン verify |

「welcome メール 1通を実装する」というのは、書き方によっては小さな副作用処理に見えますが、

- サーバーレスの実行モデル
- 並行処理の dedupe
- メール配信の認証/制限

を全部押さえる必要がある、案外重い仕事でした。

個人開発でこういう副作用処理をする時は、**「最初の1ユーザーで届くか実測する」 → 「2タブで開いて2通届かないか実測する」 → 「他人のメアドに届くか実測する」** の3ステップで踏んでおくと、本記事の罠は避けられます。

---

## 関連記事

- [AIに任せきりで作ったSaaSが、ローンチ12日間壊れていた話](https://zenn.dev/ze1ny/articles/ai-overtrust-12days-broken)
- [自分用に作ったClaude Skillsが、AI学習SaaSになった話](https://zenn.dev/ze1ny/articles/skills-to-saas)
- [個人開発SaaSを¥0で海外ローンチする完全ガイド](https://zenn.dev/ze1ny/articles/zero-yen-saas-launch)
- 🔗 CodeSensei: https://codesensei-shunnosuke-uxs-projects.vercel.app
