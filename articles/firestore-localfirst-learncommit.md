---
title: "Firestore を個人アプリに本番投入して学んだ設計判断 ― 1ユーザー1ドキュメント・ローカルファースト・無料枠運用"
emoji: "🔥"
type: "tech"
topics: ["firestore", "firebase", "googlecloud", "個人開発", "react"]
published: true
---

## はじめに

エンジニア資格に特化した勉強記録アプリ「Learn Commit」を個人開発で運用しています。

https://wakufumulearning.com/

このアプリは **ログイン不要のローカルモードで全機能が動き、Google ログインすると Firestore 同期に切り替わる**構成です。アプリ全体のアーキテクチャ（ローカルファースト、Security Rules だけで作る鍵アカウント、初期バンドルの削減）は別記事にまとめました。

この記事はそこから一歩踏み込んで、**Firestore そのものをどう設計したか**だけに絞ります。チュートリアルを一通り終えた人が「で、実運用ではどう設計するの？」とつまずくところを、実際のコードで 5 つ説明します。

1. データモデル：個人の全データを `users/{uid}` の **1 ドキュメント**に収める
2. オフライン永続化：`persistentLocalCache` を入れても **localStorage ミラーが要る**理由
3. 同期と競合解決：`_updatedAt` の last-write-wins と「**不明な相手を上書きしない**」安全弁
4. 認証と SDK 遅延ロード：判定は同期フラグ、SDK は動的 import
5. App Check：公開 read の「**課金暴発**」を止める無料枠の門番

前提スタックは React + Vite + TypeScript / Cloudflare Pages / Firebase（Firestore・Auth・App Check）、運用コストは現状 ¥0（無料枠内）です。

## 1. データモデル：個人の全データを `users/{uid}` 1 ドキュメントに

Firestore のチュートリアルは「コレクションにドキュメントをどんどん追加」しがちですが、Learn Commit は逆に、**ひとりぶんの全データ（資格・教材・学習記録・プロフィール）を 1 ドキュメントに収めています**。

```ts
// 保存単位。_updatedAt は同期の新旧判定に使う内部メタ
type Stored = DataSnapshot & { _updatedAt?: number }
// DataSnapshot = { exams, materials, records, profile }
// → これを丸ごと users/{uid} に setDoc する
```

なぜか。**個人開発で Firestore の無料枠を支配する項は read だから**です。

- 1 ドキュメント設計なら、起動時の読み込みは **1 read**、保存は **1 write** で済む
- 無料枠の read は 50,000 回/日。1 ユーザーが 1 日に何度起動しても、read はせいぜい数回
- 学習記録を 1 件 1 ドキュメントにしていたら、記録一覧を出すたびに件数ぶんの read が飛び、無料枠の消費が読めなくなる

トレードオフは Firestore の **1 ドキュメント上限 1MiB** です。テキストの学習記録なら 1 万件程度まで入る計算で、個人利用なら実質困りません。超えたら `records` だけサブコレクションに切り出す、という分岐を将来に残してあります。

:::message
「1 ドキュメントに全部」が正解なのは、**単一ユーザーが自分のデータを常に丸ごと読む**ケースだからです。複数人での協調編集・部分購読（一部だけ subscribe）・無制限に増えるデータが要るなら、素直にコレクションを分けてください。データモデルは「どう read するか」から逆算するのが Firestore のコツです。
:::

## 2. オフライン永続化：`persistentLocalCache` を入れても localStorage ミラーが要る理由

Firestore SDK にはオフライン永続化があります。Learn Commit も有効化しています。ただし注意点として、`enableIndexedDbPersistence()` は非推奨で、現在は `initializeFirestore` に `persistentLocalCache` を渡すのが正です。

```ts
// db.ts ― Firestore SDK の遅延ロード窓口（後述）。永続化付きで 1 回だけ初期化
const f = await import('firebase/firestore')
const app = await getFirebaseApp()
let db: Firestore
try {
  db = f.initializeFirestore(app, {
    localCache: f.persistentLocalCache({
      tabManager: f.persistentSingleTabManager(undefined),
    }),
  })
} catch {
  db = f.getFirestore(app) // 既に初期化済み（HMR 等）ならフォールバック
}
```

これでオフライン時の `setDoc` はローカルキャッシュに積まれ、オンライン復帰時に自動同期されます。

**では、なぜそれとは別に localStorage のミラーも持つのか？** Firestore が永続化してくれるのに二重では？ ――ここが個人開発でローカルファーストをやるときの肝でした。理由は 2 つです。

```ts
async save(data: DataSnapshot): Promise<void> {
  const stored: Stored = { ...data, _updatedAt: Date.now() }
  this.writeMirror(stored)                        // ① 同期書き込み（localStorage）
  const { f, db } = await firestore()
  await f.setDoc(f.doc(db, 'users', this.uid), stored) // ② 非同期でクラウドへ
}
```

**理由1：即描画できる（同期・SDK 非依存）。** Firestore SDK は gzip 後 ~112KB あり、Learn Commit では遅延ロードしています（次節）。起動時に localStorage を**同期的に**読めば、SDK のダウンロードも IndexedDB の読み込みも待たずに、いきなり前回の状態を描画できます。これがローカルファーストの「待ち時間ゼロ起動」です。

**理由2：非同期の隙間で取りこぼさない。** `setDoc` はサーバー反映が非同期で、しかも SDK 自体が遅延ロード。つまり「記録した直後にリロード」したとき、SDK がまだ書き込みをコミットし切る前の一瞬がありえます。先に同期で localStorage へ書いておけば、その隙間でリロードされても**ミラーから確実に復元**できます。

要するに、Firestore の永続化は「オフライン耐性」を、localStorage ミラーは「**同期的な即時描画と取りこぼし防止**」を担当していて、役割が違います。

## 3. 同期と競合解決：last-write-wins と「不明な相手を上書きしない」安全弁

複数端末で使うと、ローカルとクラウドのどちらが新しいかを決める必要があります。Learn Commit は内部メタ `_updatedAt`（保存時刻）で **last-write-wins** にしています。

ただ、ナイーブに「クラウドを読んで、ローカルが新しければ push し返す」とやると事故ります。**`getDoc` が失敗したとき（オフライン／認証未確定／permission-denied）に、空とみなして相手を上書きしてしまう**からです。そこで「クラウドの状態を本当に確認できたか」を `cloudKnown` フラグで持ち、確認できたときだけ書き戻します。

```ts
async load(): Promise<DataSnapshot> {
  const local = this.readMirror()
  let cloud: Stored | null = null
  let cloudKnown = false // getDoc が成功して初めて true
  let fs = null
  try {
    fs = await firestore()
    const snap = await fs.f.getDoc(fs.f.doc(fs.db, 'users', this.uid))
    cloudKnown = true
    if (snap.exists()) cloud = snap.data() as Stored
  } catch {
    // オフライン・認証未確定など。ローカルミラーで継続（上書きしない）
  }

  const localT = local?._updatedAt ?? -1
  const cloudT = cloud?._updatedAt ?? -1

  // ローカルが新しい → ローカル採用。ただし push し返すのは cloudKnown のときだけ
  if (local && localT >= cloudT) {
    if (localT > cloudT && cloudKnown && fs) {
      const { f, db } = fs
      f.setDoc(f.doc(db, 'users', this.uid), local).catch(/* ... */)
    }
    return toSnapshot(local)
  }
  // クラウドが新しい（他端末で更新） → クラウド採用＋ミラー更新
  if (cloud) {
    this.writeMirror(cloud)
    return toSnapshot(cloud)
  }
  // 初回ログイン: クラウドが「空だと確認できた」ときだけ、ローカルモードのデータを移行
  // （状態不明のまま空で上書きしない）
  // ...
}
```

ポイントは **「確認できていない相手は上書きしない」**という一点に尽きます。認証が確定する前の起動、ネットワーク断、Rules による read 拒否――どれも「クラウドが空」とは違います。`cloudKnown` がないと、これらを空と誤認して相手のデータを消しかねません。auth 確定後にアプリ側で reload して、改めて整合させています。

:::message alert
この方式は **last-write-wins（端末時計依存）かつ単一ユーザー想定**の割り切りです。複数ユーザーが同じドキュメントを編集する／フィールド単位でマージしたいなら、`runTransaction`・サーバータイムスタンプ・フィールド単位の更新が必要になります。「自分のデータを自分の端末から触る」用途だからこの単純さが許されています。
:::

## 4. 認証と SDK 遅延ロード：判定は同期フラグ、SDK は動的 import

ローカルモードのユーザーに Firebase SDK を一切ダウンロードさせないために、**「クラウドモードかどうかの判定」と「SDK のロード」を分離**しています。

判定は環境変数の有無を見るだけ。SDK は不要です。

```ts
// firebase.ts
export const firebaseEnabled = Boolean(cfg.apiKey && cfg.projectId)
```

そのうえで `firebase/app`・`firebase/firestore`・`firebase/auth`・`firebase/app-check` を**すべて動的 import**にしています。`firebase/app` すら初期バンドルに入れません。

```ts
let appPromise: Promise<FirebaseApp> | null = null
export function getFirebaseApp(): Promise<FirebaseApp> {
  if (!appPromise) {
    appPromise = import('firebase/app').then(({ initializeApp }) => {
      const app = initializeApp(cfg)
      ensureAppCheck(app)
      return app
    })
  }
  return appPromise
}
```

認証も同じで、設定があるときだけ `firebase/auth` を動的 import し、Google ログイン（`signInWithPopup`）と `onAuthStateChanged` を繋ぎます。永続化は `browserLocalPersistence` です。

結果として、**ログインしていないユーザーは Firestore SDK（gz ~112KB）を 1 バイトも落としません。** タイムラインを開く・ログインするなど、実際に使う瞬間に初めて取りに行きます。

:::message
ただし「動的 import したのにバンドラが 1 チャンクに固めてしまい遅延が無効化される」という罠があります（vite の `manualChunks`）。この詳細はアーキテクチャ記事の方に書きました。dynamic import と手動チャンク分割は喧嘩します。
:::

## 5. App Check：公開 read の「課金暴発」を止める門番

Learn Commit には公開学習タイムライン（オプトイン）があり、ここは**誰でも read 可**です。サーバーレスで「誰でも読める公開データ」を置くと、無料枠に対するリスクが 1 つ増えます。**スクリプトで read を叩かれて課金が暴発する**ことです。

これを止めるのが Firebase App Check（reCAPTCHA v3）です。「**本物の自分のアプリからのリクエストか**」を検証し、bot やスクレイピングからの read を弾きます。

```ts
export function ensureAppCheck(app: FirebaseApp): void {
  const siteKey = import.meta.env.VITE_FIREBASE_APPCHECK_SITE_KEY
  if (!siteKey) return // 未設定なら何もしない（個人版・ローカルモードに影響なし）
  import('firebase/app-check').then(({ initializeAppCheck, ReCaptchaV3Provider }) => {
    initializeAppCheck(app, {
      provider: new ReCaptchaV3Provider(siteKey),
      isTokenAutoRefreshEnabled: true,
    })
  })
}
```

設計上のポイント：

- **サイトキーがあるときだけ有効化**（条件付き）。個人で動かすローカルモードには影響しません
- localhost 検証用に **debug token** を入れる口を用意（`FIREBASE_APPCHECK_DEBUG_TOKEN`）。本番ビルドでは未設定にします
- App Check は「認証」とは別物です。認証は「誰か」、App Check は「どのアプリから」。公開 read のように**認証で守れない経路の前段**として効きます

「個人データは Security Rules で本人しか読めない」ようにしていても、公開コレクションは別の守りが要る、という整理です。

## まとめ

Learn Commit で Firestore を本番運用して得た設計判断はこの 5 つでした。

- **データモデルは read 課金から逆算する。** 単一ユーザーが丸ごと読むなら `users/{uid}` 1 ドキュメントが強い（起動 1 read / 保存 1 write）。上限 1MiB だけ意識する
- **オフライン永続化（`persistentLocalCache`）と localStorage ミラーは役割が違う。** 前者はオフライン耐性、後者は同期的な即時描画と取りこぼし防止
- **同期は last-write-wins でも、`cloudKnown` のような「不明な相手を上書きしない」安全弁が要る**
- **判定は同期フラグ、SDK は全部動的 import。** 非ログインユーザーに Firestore SDK を落とさせない
- **公開 read には App Check。** 認証で守れない経路の課金暴発を止める

「サーバーレス × ローカルファースト × 無料枠」は個人開発と相性が良く、Firestore はその中心にちょうど収まります。同じ資格を勉強している方、よければ使ってみてください。

https://wakufumulearning.com/

> アプリ全体のアーキテクチャ（ローカルファースト全体像・Security Rules だけで作る鍵アカウント・初期バンドル gz 231KB→94KB）は別記事に書いています。あわせてどうぞ。
