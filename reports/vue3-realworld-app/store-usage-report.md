# Pinia ストア使用箇所調査

- 対象: [gpingfeng/vue3-realworld-app](https://github.com/gpingfeng/vue3-realworld-app)
- 対象範囲: `src/` 全体
- 調査日: 2026-11-04

## 1. 概要

| 項目 | 件数 |
| --- | ---: |
| Pinia ストア | 1 |
| ストア使用ファイル | 11 |
| 画面（ストアを利用するルート） | 6 |
| CSV 操作・ストア組み合わせ | 15 |
| 違反候補 | 2（ストア外からの直接更新） |

`vue` 3.3.4、`pinia` 2.1.6、`vue-router` 4.2.4。ストアは Setup Store。永続化プラグインは依存にない。JWT は `localStorage` に保存されるが、`userInfo` はメモリ上の state。

ルーターはファイルベースの自動生成。該当画面は `/`、`/login`、`/register`、`/article/:id`、`/profile/:id`、`/settings`。共通ヘッダーはこれらの画面で使用される。

## 2. ストア一覧

| ID / export | 種別 | state | getter | action | 依存 / 永続化 |
| --- | --- | --- | --- | --- | --- |
| `user` / `useUserStore` | Setup | `userInfo` | `isLoggedIn` | `handleAuthAction`, `getUserInfo` | API。永続化プラグインなし。トークンのみ localStorage |

定義: [`useUserStore.ts`](../../vue3-realworld-app/src/stores/useUserStore.ts)

## 3. 画面別・操作単位の一覧

CSV は「画面・操作・ストア・アクセス種別」の組み合わせを 1 行としている。同じ画面内で同じイベントから生じる複数の表示参照は、対象をまとめている。

| 操作 | トリガー | ハンドラ / 経路 | ストア | R/W | 対象 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 初回遷移・ログイン状態確認 | 画面遷移 | `router.beforeEach` | user | R | `isLoggedIn` | `src/router/permissions.ts:5,9` | 確定 | - |
| トークンからユーザー情報復元 | 画面遷移 | `router.beforeEach` → `getUserInfo` | user | W | `getUserInfo`, `userInfo` | `src/router/permissions.ts:14` → `src/stores/useUserStore.ts:18-21` | 確定 | - |
| ログイン送信 | submit | `AuthPage` → `handleAuthAction` | user | W | `handleAuthAction`, `userInfo` | `src/components/AuthPage.vue:12,19` → `src/stores/useUserStore.ts:13-16` | 確定 | - |
| 新規登録送信 | submit | `AuthPage` → `handleAuthAction` | user | W | `handleAuthAction`, `userInfo` | `src/components/AuthPage.vue:12,19` → `src/stores/useUserStore.ts:13-16` | 確定 | - |
| 共通ナビゲーション表示 | 描画 | `TheHeader` | user | R | `isLoggedIn`, `userInfo` | `src/layouts/components/TheHeader.vue:4-5,35-38` | 確定 | - |
| フィード表示・切替 | 描画 / 入力変更 | `index.vue` | user | R | `userInfo`, `isLoggedIn` | `src/pages/index/index.vue:6-16,45` | 確定 | - |
| 記事表示・コメント導線 | 描画 | 記事ページ | user | R | `userInfo`, `isLoggedIn` | `src/pages/article/[[id]].vue:10,29,124` | 確定 | - |
| プロフィール表示 | 描画 | プロフィールページ | user | R | `userInfo` | `src/pages/profile/[[id]].vue:7,16,45-46` | 確定 | - |
| 設定画面の初期値設定 | mounted | `onMounted` | user | R | `userInfo` | `src/pages/settings/index.vue:32-34` | 確定 | - |
| プロフィール設定保存 | submit | `onSubmit` → API → 直接代入 | user | W | `userInfo` | `src/pages/settings/index.vue:19` | 確定 | ストア外から直接更新 |
| ログアウト | click | `onLogout` → 直接代入 | user | W | `userInfo` | `src/pages/settings/index.vue:25-29` | 確定 | ストア外から直接更新 |
| 記事カードのお気に入り | click | `onFavorited` | user | R | `isLoggedIn` | `src/components/ArticlePreview.vue:11,14,45` | 確定 | - |
| 記事のお気に入り | click | `onFavorited` | user | R | `isLoggedIn` | `src/components/FavoriteButton.vue:10,16,34` | 確定 | - |
| 著者フォロー | click | `onFollow` | user | R | `isLoggedIn` | `src/components/FollowButton.vue:9,14,32` | 確定 | - |
| コメント表示・投稿者判定 | 描画 / submit | `isAuthor` / `onSubmit` | user | R | `userInfo` | `src/pages/article/components/CommentList.vue:11,15,55` | 確定 | - |

## 4. ストア別マトリクス

| ストア | 共通ナビ | `/` | `/login`, `/register` | `/article/:id` | `/profile/:id` | `/settings` | ルート遷移 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| user | R | R | W | R | R | RW | RW |

## 5. 違反候補

- **直接更新** — `src/pages/settings/index.vue:19` が API 応答を `store.userInfo` に代入し、`src/pages/settings/index.vue:27` がログアウト時に `null` を直接代入する。Store action を経由しない更新。
- いずれも `src/stores/` 外からの更新として候補化した。設定保存の更新では `updateUserInfo` が持つ処理（JWT 保存）を通らない点に注意。

## 6. 調査の限界

- Pinia 利用は単一ストアで、列挙したコンポーネントからユーザー操作・描画・ルートガードまで追跡できた。
- ルート名はファイルベースの自動生成に依存する。記事詳細の `ArticleId?` はルーターガード中の route name 表記を参照。
- CSV は同一イベント内の同じストアアクセスを対象単位でまとめている。細かなテンプレート式ごとの重複参照は分割していない。
