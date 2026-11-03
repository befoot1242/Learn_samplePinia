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
| 画面別ストアアクセス明細 | 15 |
| 違反候補 | 2（ストア外からの直接更新） |

`vue` 3.3.4、`pinia` 2.1.6、`vue-router` 4.2.4。ストアは Setup Store。永続化プラグインは依存にない。JWT は `localStorage` に保存されるが、`userInfo` はメモリ上の state。

ルーターはファイルベースの自動生成。該当画面は `/`、`/login`、`/register`、`/article/:id`、`/profile/:id`、`/settings`。共通ヘッダーはこれらの画面で使用される。

## 2. ストア一覧

| ID / export | 種別 | state | getter | action | 依存 / 永続化 |
| --- | --- | --- | --- | --- | --- |
| `user` / `useUserStore` | Setup | `userInfo` | `isLoggedIn` | `handleAuthAction`, `getUserInfo` | API。永続化プラグインなし。トークンのみ localStorage |

定義: [`useUserStore.ts`](../../vue3-realworld-app/src/stores/useUserStore.ts)

## 3. 画面別・操作単位の一覧

CSV は1行を1ストアアクセスとして記録する。同一画面・操作でも利用ストアごとに行を分け、画面別にまとめた。

### Route guard (`all protected routes; token restoration`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Check login state | navigation | router.beforeEach | user | R | isLoggedIn | read | src/router/permissions.ts:5,9 | 確定 | - |
| Fetch current user | navigation | router.beforeEach | user | W | getUserInfo; userInfo | action | src/router/permissions.ts:14 -> src/stores/useUserStore.ts:18-21 | 確定 | - |

### Login (`/login`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Submit credentials | submit | AuthPage onSubmit | user | W | handleAuthAction; userInfo | action | src/components/AuthPage.vue:12,19 -> src/stores/useUserStore.ts:13-16 | 確定 | - |

### Register (`/register`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Submit registration | submit | AuthPage onSubmit | user | W | handleAuthAction; userInfo | action | src/components/AuthPage.vue:12,19 -> src/stores/useUserStore.ts:13-16 | 確定 | - |

### Shared header (`all routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Render auth links and user identity | render | navList computed | user | R | isLoggedIn; userInfo | storeToRefs + property read | src/layouts/components/TheHeader.vue:4-5,35-38 | 確定 | - |

### Home feed (`/`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Choose personalized feed | render | setup and template | user | R | userInfo; isLoggedIn | property read | src/pages/index/index.vue:6-16,45 | 確定 | - |

### Article detail (`/article/:id`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Determine article owner and comment prompt | render | isAuthor / template | user | R | userInfo; isLoggedIn | property read | src/pages/article/[[id]].vue:10,29,124 | 確定 | - |
| Favorite article | click | onFavorited | user | R | isLoggedIn | property read | src/components/FavoriteButton.vue:10,16,34 | 確定 | - |
| Show commenter identity and authorize comment controls | render | isAuthor / template | user | R | userInfo | property read | src/pages/article/components/CommentList.vue:11,15,55 | 確定 | - |

### Profile (`/profile/:id`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Determine whether profile is current user | render | isSelf | user | R | userInfo | property read | src/pages/profile/[[id]].vue:7,16,45-46 | 確定 | - |

### Settings (`/settings`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Initialize edit form | mounted | onMounted | user | R | userInfo | property read | src/pages/settings/index.vue:32-34 | 確定 | - |
| Save profile | submit | onSubmit | user | W | userInfo | direct assignment | src/pages/settings/index.vue:19 | 確定 | 直接更新 |
| Log out | click | onLogout | user | W | userInfo | direct assignment | src/pages/settings/index.vue:25-29 | 確定 | 直接更新 |

### Article preview (`/ and feed pages`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Favorite article | click | onFavorited | user | R | isLoggedIn | property read | src/components/ArticlePreview.vue:11,14,45 | 確定 | - |

### Profile/article detail (`/profile/:id; /article/:id`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Follow author | click | onFollow | user | R | isLoggedIn | property read | src/components/FollowButton.vue:9,14,32 | 確定 | - |

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
