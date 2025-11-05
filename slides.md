---
theme: default
title: account-apiのリファクタリング
class: text-center
transition: slide-left
---

# account-apiのリファクタリング

---
transition: slide-left
---

# account-apiのつらいところ

- **巨大なAPI** - 一つのAPIで更新処理を全て担当している
- **any型** - 多くの場所で型がanyになっている
- **破壊的な関数** - 渡した値の中身が書き換えられる
- **テスタビリティ** - サービス、コントローラーなどの境界が曖昧で密結合
- **古いバージョン** - Nodeがv18、Expressがv4
- **Express機能の活用** - ルーター、ミドルウェア

---
transition: slide-left
---

# 巨大なAPI

各種申込などで使われるupdateAccountがコントローラーの記述が膨大(1000行)で全ての申し込みが関わるので一つの申込に関して修正が入ったとき全ての申込をテストしなければならなくなる

![](./images/updateAccount.png)

---
transition: slide-left
---

# any型

引数に型がつけられていないのでany型になっており、どんな値が入ってくるか実行して確かめるまでわからない

![](./images/any.png)

---
transition: slide-left
---

# 破壊的な関数

関数内で`param`の値を書き換えており、事故が起こりやすい

![](./images/destructive.png)

---
transition: slide-left
---

# 古いバージョン - node -

v18はサポート外 v24かv22に変更した方が良さそう

![](./images/nodev18.png)

<img src="./images/schedule.svg" style="height: 250px;">

---
transition: slide-left
---

# テスタビリティが低い

例えば画像のコントローラだと、コントローラーを初期化した時にはもうデータソースが初期化されているのでコントローラーのテストをやろうとした時にDBに繋いでしまう

![](./images/testability.png)

---
transition: slide-left
---

# 古いバージョン - express -

v5だとasync/awaitに対応しているので使いやすい
セキュリティ関連(DoS)のアップデートもあったので上げた方が良さそう
v4のサポート期間的には1年ほど余裕がある

![](./images/expressv4.png)

![](./images/expressschedule.png)

---
transition: slide-left
---

# Express 機能の活用 - ルーター -

ルーティングを機能ごとにまとめることができる

##### router未使用

```typescript
const app = Express();
app.get('/calendar/events', (req, res, next) => {
  // ..
})
```

##### router使用

```typescript
const app = Express();
const router = Router();
router.get('/events', (req, res, next) => {
  // ..
})
app.use('/calendar', router)
```

---
transition: slide-left
---

# Express 機能の活用 - ミドルウェア -

処理の前後にミドルウェアを入れることができる

##### middleware未使用

```typescript
const app = Express()

app.get("/", (req, res, next) => {
  console.log('Time:', Date.now())
  // ..
})
```

##### middleware使用

```typescript
const app = Express()

app.use((req, res, next) => {
  console.log('Time:', Date.now())
  next()
})

app.get('/', (req, res, next) => {
  // ..
})
```

---
transition: slide-left
---

# 改善案

1. typescript -> Golang
2. express -> hono
3. express -> nestjs
4. express

```mermaid {theme: 'neutral', scale: 0.8}
graph LR
A{言語を変える} 
A -- ◯ --> B[改善案1]
A -- ✖︎ --> C{フレームワークを変える}
C -- ◯ --> D{hono or nestjs}
D -- hono --> E[改善案2]
D -- nestjs --> F[改善案3]
C -- ✖︎ --> G[改善案4]
```

---
transition: slide-left
---

# 改善案1 typescript -> Golang

typescriptからGoに置き換える

コスト <span style="color: red;">大</span>

- Google主導のGoは言語仕様が意図的にシンプルに保たれており誰が書いてもある程度同じ実装になる
- 言語機能の追加にとても慎重で後方置換性が高くGoのバージョンアップで破壊的変更が少ないため長期的に使える(昔のコードが今でも動く)
- 言語仕様が少ないので学習コストが低い
- 単一のバイナリにコンパイルされるためDockerのイメージが小さくメモリ使用量も少ない

---
transition: slide-left
---

# 改善案2 Express -> Hono

フレームワークを変える

コスト <span style="color: orange;">中</span>

- expressに似ているAPIなので学習コストが低い
- リクエストに型をつけたりするのが簡単
- 既存コードをある程度流用できる

---
transition: slide-left
---

# 改善案3 Express -> nestjs

フレームワークを変える

コスト <span style="color: red;">大</span>

- ガチガチのフレームワークなので既存コードはあまり流用できない
- フレームワークとしてDI(依存性注入)がサポートされているのでサービス間の密結合を解消し、テストしやすいコードになる
- 必要な機能が全て揃っている（Djangoみたいな）
- 明確に構造が決まっているため、保守性が高い

---
transition: slide-left
---

# 改善案4 Express

APIのリファクタリング

コスト <span style="color: blue;">小</span>

- accountUpdateを適切なサイズに分ける
  - 各種申込毎、変更届毎、口座開設時の処理
  - フロントからはapplication_typeを渡さないようにする
  - 一つのエンドポイントが単一の動作をするようにする
- バージョンをv4からv5にあげる
  - 破壊的変更もあるが使用している箇所は少なそうだったのですぐに動かせそう
- nodeのバージョンを上げる
  - 使用中の主要なライブラリは対応しているのですぐ動かせそう