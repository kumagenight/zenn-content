---
title: "Studyplusにない資格があったので、勉強記録アプリを自作した ― ローカルファースト設計と運用¥0の話"
emoji: "🎓"
type: "tech"
topics: ["個人開発", "react", "firebase", "firestore", "cloudflare"]
published: false
---

## はじめに

Google Cloud の Associate Cloud Engineer（ACE）の勉強を始めたとき、勉強時間を Studyplus で記録しようとしたら、**Google Cloud の資格が登録されていません**でした。

Studyplus 自体はとても良いサービスです。ただ、大学受験を中心に育ってきたサービスなので、エンジニア資格はカバーが薄い。「ないなら作ればいい」がエンジニアの特権なので、**エンジニア資格に特化した勉強記録アプリ**を作りました。

https://wakufumulearning.com/

この記事では、個人開発で実際に手を動かして得た設計の学びを 3 つに絞って書きます。

1. **ローカルファースト + クラウド同期**（ログイン不要で動き、ログインすると同期される）
2. **鍵アカウントを Firestore Security Rules だけで実装する**
3. **初期バンドルを gz 231KB → 94KB にした話**（manualChunks の罠）

## 作ったもの

- 約 120 のエンジニア資格（Google Cloud / AWS / Azure / 情報処理 / CNCF / Anthropic など 19 ドメイン）に対応した勉強記録アプリ
- 1 分単位の学習記録。ストップウォッチ／タイマー／手動入力の 3 モード
- 教材データベース約 195 件（書籍・公式ドキュメント・ハンズオン）から選んで記録
- 学習タイムライン（公開はオプトイン。フォロー・鍵アカウント・いいね・コメント）
- 週間サマリや合格報告を画像カードにして X にシェア

技術スタックは以下です。

| レイヤ | 採用 | 月額 |
|---|---|---|
| フロント | React + Vite + TypeScript | - |
| ホスティング | Cloudflare Pages | ¥0 |
| データ・認証 | Firebase（Firestore / Auth / App Check） | ¥0（無料枠内） |
| 画像生成（シェアカード） | Canvas API（クライアント側） | ¥0 |

個人の全データを `users/{uid}` の **1 ドキュメント**に収める設計にしたことで、読み書きが「起動時 1 read / 保存 1 write」で済み、Firestore の無料枠（read 50K/日）に余裕で収まります。1 ドキュメントの上限は 1MiB ですが、テキストの学習記録なら 1 万件程度まで入る計算なので、個人利用では実質困りません。

## 設計1: ローカルファースト + クラウド同期

このアプリは**ログインしなくても全機能（記録・集計・教材）が動きます**。データは localStorage に保存され、Google ログインすると Firestore 同期に切り替わる構成です。

```
                  ┌─ ログインしていない ─→ LocalRepo（localStorage）
起動 ─ Repo 選択 ─┤
                  └─ ログイン済み ─────→ FirestoreRepo（Firestore + ミラー）
```

`Repo` インターフェース（`load`/`save`）だけ共有して実装を差し替える、素直な構成です。ポイントは **FirestoreRepo 側も localStorage をミラーとして併用する**ことです。

```ts
async save(data: DataSnapshot): Promise<void> {
  const stored = { ...data, _updatedAt: Date.now() }
  this.writeMirror(stored)        // ① 同期書き込み（localStorage）
  await setDoc(this.ref(), stored) // ② 非同期でクラウドへ
}
```

Firestore の `setDoc` はサーバー反映が非同期なので、「記録した直後にリロードすると消える」事故が起こりえます。先にローカルへ同期的に書いておけば、リロードしてもミラーから復元できます。読み込み時は `_updatedAt` を比較して新しい方を採用し、ローカルが先行していればクラウドへ押し戻します（last-write-wins）。

### 起動を認証判定でブロックしない

もうひとつ、起動時の話。Firebase Auth の認証状態の初回確定は「SDK のロード + IndexedDB の読み込み」を待つため、数百 ms かかります。最初は素直に `onAuthStateChanged` の初回発火を待ってから描画していたのですが、これだと毎回「読み込み中…」が見えてしまう。

そこで**前回の認証状態（uid か未ログインか）を localStorage に記録しておき、次回起動はそれを信じて即描画**するようにしました。auth が確定した時点でズレていれば作り直し、合っていればクラウド差分だけ裏で取り込みます。推測が外れても表示が壊れない（ミラーがあるから）のがローカルファーストの効用です。

## 設計2: 鍵アカウントを Firestore Rules だけで実装する

タイムラインには鍵アカウント（承認したフォロワーだけが投稿を読める）があります。サーバーレス構成なので、この認可を **Firestore Security Rules だけ**で実装しました。

データモデルはこうです。

```
users/{uid}        … 個人の全データ。本人しか読めない（公開しない）
profiles/{uid}     … 公開プロフィール。「公開する」を選んだ人だけ作られる
activities/{id}    … 公開した学習記録。locked フラグを非正規化して持つ
follows/{owner}/followers/{uid} … 承認済みフォロワー（認可レコード）
```

大事なのは「**公開はオプトイン**」の徹底です。`users/{uid}` の生データは絶対に他人へ出さず、公開用コレクションへ**意図的に書き出した分だけ**が見えます。

鍵アカの read 制御は rules でこう書けます。

```js
function isApprovedFollower(ownerUid) {
  return exists(/databases/$(database)/documents/follows/$(ownerUid)/followers/$(request.auth.uid));
}

match /activities/{id} {
  allow read: if resource.data.get('locked', false) != true
              || request.auth.uid == resource.data.uid
              || isApprovedFollower(resource.data.uid);
}
```

- 公開投稿（`locked == false`）は条件の最初で短絡するので、`exists()` の追加 read コストがかかりません
- 鍵投稿だけ `follows/{owner}/followers/{me}` の存在チェックが走ります
- グローバルフィードのクエリは `where('locked', '==', false)` を付け、rules と整合させます

この「見せかけではなく rules レベルで遮断される鍵アカ」は、エミュレータでテストを書いて担保しています。`@firebase/rules-unit-testing` + vitest で「非フォロワーは鍵投稿を読めない」「なりすまし投稿は拒否される」「いいねカウントは ±1 しか動かせない」など 49 ケースを自動化しました。rules はデプロイしたら UI からは壊れたことに気づきにくいので、テストがあると安心感がまるで違います。

## 設計3: 初期バンドル gz 231KB → 94KB（manualChunks の罠）

リリース後、起動が重いのが気になって計測したところ、初期ロードが gzip 後 231KB あり、**そのうち 137KB が firebase チャンク**でした。

原因を調べると、二段構えの問題でした。

**① データ層が `firebase/firestore` を静的 import していた。** 認証（`firebase/auth`）は最初から `await import()` で遅延させていたのに、Firestore を使うリポジトリ層が普通に import していたので、結局初期バンドルに firebase が全部入っていました。

**② vite の manualChunks が遅延を無効化していた。** バンドルを整理するつもりで書いたこの設定が逆効果でした。

```ts
// やってはいけない例
manualChunks(id) {
  if (id.includes('node_modules/firebase')) return 'firebase' // ←これ
}
```

`firebase/*` を 1 チャンクに固めると、静的に必要な `firebase/app` と動的に遅延したい `firebase/firestore` が**同じチャンクに同居**してしまい、初期ロードで丸ごと引きずり込まれます。dynamic import でコード分割する場合、**manualChunks でその境界をまたぐグルーピングをしてはいけない**。Rollup の自動分割に任せれば、dynamic import の境界に沿ってチャンクが分かれます。

対処は次の 3 つです。

1. Firestore SDK の遅延ロード窓口を 1 モジュール（`db.ts`）に集約し、データ層の全関数を `const { f, db } = await firestore()` 経由に書き換え
2. `getFirebaseApp()` を Promise 化して `firebase/app` ごと遅延（判定に SDK が要らないよう、設定有無のフラグだけ同期で持つ）
3. manualChunks から firebase の指定を削除（react-vendor だけ残す）

結果、初期ロードは **gz 94KB（main 43KB + react 46KB + CSS 5KB）**になりました。ログインしていないユーザーは Firestore SDK（gz 112KB）を**1 バイトもダウンロードしません**。タイムラインを開くなど、実際に使う瞬間に初めて取りに行きます。

## 運用コストと収益の話

- ランニングコストは現状 **¥0**（Cloudflare Pages 無料枠 + Firestore 無料枠）
- コストの支配項は Firestore の read です。雑な試算で 1,000 ユーザー ≈ 月数百円、1 万ユーザーで月数千円のオーダー。そうなったら嬉しい悲鳴です
- 収益化は教材（書籍）紹介の楽天アフィリエイトリンクのみ。**報酬の有無で教材の並び順は変えない**方針です（並び替えた瞬間にレコメンドの信頼が死ぬので）

## まとめ

- ニッチでも「自分が毎日使うもの」を作ると、ドッグフーディングで改善が回り続けます
- ローカルファースト + クラウド同期は、サーバーレス個人開発と相性が良い（オフライン耐性・起動速度・無料枠の節約が同時に手に入る）
- Firestore の鍵アカ実装は rules + エミュレータテストまでやって初めて安心できる
- **dynamic import と manualChunks の組み合わせには注意**。固めた瞬間に遅延が死にます

同じ資格を勉強している方、よければ使ってみてください。タイムラインでお会いしましょう。

https://wakufumulearning.com/
