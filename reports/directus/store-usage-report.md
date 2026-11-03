# Pinia ストア使用箇所調査

- 対象: [directus/directus](https://github.com/directus/directus) の `app/`
- 調査対象: 指定された `app/src/stores/` の 16 ストア
- 調査日: 2026-11-04

## 1. 概要

| 項目 | 件数 |
| --- | ---: |
| 対象ストア | 16 |
| app 全体で検出したストア定義 | 22（AI / private-view / field-detail の局所ストアを含む） |
| 画面・共通 UI / lifecycle の操作グループ | 22 |
| CSV 操作・ストア組み合わせ | 37 |
| 明確な Pinia 利用違反候補 | 0 |
| 設計確認候補 | 1（store 間循環依存） |

`vue` 3.5.43、`pinia` 2.3.1、`vue-router` 4.6.3。バージョンは workspace catalog を参照。persisted-state plugin は確認できず、主なストアは API から hydrate / dehydrate される。アプリ全体の `useAppStore` は `@directus/stores` 由来であり、指定された `app/src/stores/` の 16 件には含めていない。

ルーターは `/setup`、`/login` などの固定 route と、権限に応じて登録される module route を併用する。機能画面は Content、Files、Users、Settings、Flows、Insights、Activity、Deployment、Visual を中心に整理した。

CSV は画面・操作・ストア単位で同じイベントの同じストア利用を集約したイベント表である。テンプレートの各プロパティ参照を全件ばらした行数ではない。Vue ファイルでの利用はストアごとに以下の数のファイルで検出した（各ストア間で重複あり。`.vue` 限定）。

| Store | Vue 利用ファイル数 | Store | Vue 利用ファイル数 |
| --- | ---: | --- | ---: |
| user | 52 | settings | 21 |
| server | 44 | collections | 31 |
| fields | 43 | relations | 20 |
| license | 30 | permissions | 10 |
| insights | 8 | flows | 5 |
| presets | 4 | extensions | 4 |
| files | 3 | requests | 1 |
| notifications | 11 | translations | 1 |

## 2. ストア一覧

| ID / export | 種別 | 主な state / getter | 主な action | 依存ストア | 永続化 |
| --- | --- | --- | --- | --- | --- |
| `userStore` / `useUserStore` | Setup | `currentUser`, `loading`, `error`; `fullName`, `isAdmin`, `language`, `textDirection` | `hydrate`, `dehydrate`, `hydrateAdditionalFields`, `trackPage` | server | Pinia 永続化なし |
| `translations` / `useTranslationsStore` | Setup | `loading`, `translations`, `lang` | `loadTranslations`, `create` | - | Pinia 永続化なし |
| `settingsStore` / `useSettingsStore` | Option | `settings` | `hydrate`, `dehydrate`, `setOwner`, `updateSettings` | user | Pinia 永続化なし |
| `serverStore` / `useServerStore` | Setup | `info`, `auth`; `providerOptions` | `hydrate`, `dehydrate` | - | Pinia 永続化なし |
| `requestsStore` / `useRequestsStore` | Option | `queue`; `queueHasItems` | `startRequest`, `endRequest` | - | transient |
| `relations` / `useRelationsStore` | Setup | `relations` | `hydrate`, `dehydrate`, relation lookup/upsert | fields, AI tools | Pinia 永続化なし |
| `presetsStore` / `usePresetsStore` | Option | `collectionPresets`; `bookmarks` | `hydrate`, preset CRUD/cache actions | user | Pinia 永続化なし |
| `permissionsStore` / `usePermissionsStore` | Option | `permissions` | `hydrate`, `dehydrate`, `getPermission`, `hasPermission` | user | Pinia 永続化なし |
| `notificationsStore` / `useNotificationsStore` | Option | dialogs, queue, previous, notifications, unread, overlay; `lastFour` | `hydrate`, unread refresh, add/hide/remove/update | user | Pinia 永続化なし |
| `licenseStore` / `useLicenseStore` | Setup | license info/addons/pending resolution/loading/error; entitlement and license computed values | hydrate/refresh and license resolution actions | user | Pinia 永続化なし |
| `insightsStore` / `useInsightsStore` | Setup | dashboards/panels/variables and related loading state | insight/dashboard/panel load and mutation actions | user (API context) | Pinia 永続化なし |
| `flowsStore` / `useFlowsStore` | Setup | `flows` | `hydrate`, `dehydrate`, `getManualFlowsForCollection` | permissions, AI tools | Pinia 永続化なし |
| `filesStore` / `useFilesStore` | Setup | reactive upload map; `isAnyUploadActive` | `upload` → upload-local `start`, `finish` | - | transient |
| `fieldsStore` / `useFieldsStore` | Setup | `fields` | hydrate/dehydrate, field CRUD, field queries, translation | collections, relations, AI tools | Pinia 永続化なし |
| `extensions` / `useExtensionsStore` | Setup | `loading`, `error`, `extensions`, reload notice; `extensionIds` | `refresh`, `toggleState`, install/uninstall/reinstall/remove | notifications | Pinia 永続化なし |
| `collectionsStore` / `useCollectionsStore` | Setup | collections and sorted/configured/system/visible computed lists | hydrate/dehydrate, collection CRUD and translation | relations, license, AI tools | Pinia 永続化なし |

ストア定義: [`app/src/stores/`](../../directus/app/src/stores/)

## 3. 画面別・操作単位の一覧

| 操作 | トリガー種別 | ハンドラ / 呼び出し | ストア | R/W | 対象 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 初回ルート遷移・setup 状態判定 | navigation | router guard | server | RW | `info`, `hydrate` | `app/src/router.ts:31-44,134-145` | 確定 | - |
| 認証後の初期化 | lifecycle / navigation | guard → `hydrate()` | user | W | `hydrate`, `currentUser`, `language` | `app/src/router.ts:148-167` → `app/src/hydrate.ts:69` | 確定 | - |
| 認証後の初期化 | lifecycle | `hydrate()` | permissions, fields | W | permissions / fields hydrate | `app/src/hydrate.ts:74` | 確定 | - |
| 認証後の初期化 | lifecycle | `hydrate()` | collections, relations, presets, settings, server, insights, flows, notifications, license | W | 各ストアの `hydrate` | `app/src/hydrate.ts:30-45,76` | 確定 | - |
| ログアウト・認証解除 | auth lifecycle | `dehydrate()` | 各 hydration store | W | 各ストアの `dehydrate` | `app/src/hydrate.ts:94-103` | 確定 | - |
| 認証後のページ遷移 | navigation → timeout | `onAfterEach` → `trackPage` | user | W | `trackPage`, `currentUser.last_page` | `app/src/router.ts:199-215` | 確定 | - |
| 共通 private layout | 描画 / state change | `private-view` | user, server, settings, license | R | user / project / settings / lock state | `app/src/views/private/private-view/components/private-view.vue:57,70-72` | 確定 | - |
| Content collection の選択・一覧 | 描画 / selection | collection route/navigation | collections, presets, user, permissions | R | collection metadata, saved presets, access | `app/src/modules/content/routes/collection.vue:66,168,350`; `app/src/modules/content/components/NavigationItem.vue:31-33` | 確定 | - |
| Content item 表示・編集 | mounted / input / save | item route / interfaces | fields, collections, settings, notifications, user | RW | field metadata, project settings, item notifications | `app/src/modules/content/routes/item.vue:105-107,982` | 確定 | - |
| Data Model: collection / field 編集 | click / submit | settings routes | collections, fields, relations, license | RW | collection/field/relationship CRUD, entitlement checks | `app/src/modules/settings/routes/data-model/collections/collections.vue:42-45`; `app/src/modules/settings/routes/data-model/field-detail/field-detail.vue:50-51` | 確定 | store dependency cycle |
| Permissions / roles 編集 | input / save | permissions interfaces / role route | permissions, collections, fields, user, license | RW | permission data, collection/field options, access checks | `app/src/interfaces/_system/system-permissions/system-permissions.vue:54`; `app/src/modules/settings/routes/roles/item.vue:33-35` | 確定 | - |
| Content navigation bookmark / preset | click / save | navigation / preset route | presets, collections | RW | `collectionPresets`, `bookmarks`, preset CRUD | `app/src/modules/content/components/navigation-bookmark.vue:36`; `app/src/modules/settings/routes/presets/item.vue:53-54` | 確定 | - |
| Files list・upload | click / file event / progress | file collection / `v-upload` | files, user, server, notifications, presets | RW | upload progress, file metadata, notification | `app/src/modules/files/routes/collection.vue:51-60,239`; `app/src/components/v-upload.vue:56-57,140,317` | 確定 | - |
| Flows 一覧・編集・手動実行 | navigation / click / save | flow module | flows, permissions, relations, license, user | RW | flow list, manual-flow options, access / entitlement | `app/src/modules/flows/overview.vue:68,170,180`; `app/src/modules/flows/flow.vue:65-66` | 確定 | - |
| Insights dashboard / panel 設定 | navigation / save | insights module and panels | insights, fields | RW | dashboard / panel / variable data and field metadata | `app/src/modules/insights/routes/dashboard.vue:44`; `app/src/modules/insights/routes/panel-configuration.vue:45`; `app/src/panels/list/panel-list.vue:34-35` | 確定 | - |
| Settings / extension 操作 | navigation / click | Settings routes | settings, server, extensions, notifications | RW | project settings, extension inventory and state | `app/src/modules/settings/routes/project/project.vue:23-24`; `app/src/modules/settings/routes/extensions/extensions.vue:17` | 確定 | - |
| License status / recovery | route / reactive watch / click | guard + `app.vue` + license UI | license, user, server | RW | lock/entitlement status, recovery actions | `app/src/app.vue:26-27`; `app/src/router.ts:181-196`; `app/src/routes/license-recovery.vue:11-12` | 確定 | - |
| Global notifications | click / API result / timer | notification drawer / producers | notifications, user, collections | RW | unread count, queues, dialogs, display | `app/src/views/private/components/notifications-drawer.vue:47-49`; `app/src/views/private/components/notification-dialogs.vue:19-21` | 確定 | - |
| API request activity indicator | API request lifecycle | API client / logo | requests | RW | request queue / `queueHasItems` | `app/src/views/private/components/module-bar-logo.vue:11` | 確定 | - |
| Translated field editing | input / save | translated string interface | translations, user, settings, permissions | RW | current locale translations / create | `app/src/interfaces/_system/system-input-translated-string/input-translated-string.vue:44,48` | 確定 | - |

## 4. ストア別マトリクス

`RW` は画面イベントで state 更新と参照の両方があることを示す。全ストアは認証後の全体 hydrate/dehydrate にも参加するものがある。

| ストア | Global lifecycle / auth | Content / item | Files | Data Model / Permissions | Settings / Users | Flows | Insights | 共通 UI |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| user | RW | R | R | R | R | R | R | R |
| translations | W | R | - | R | R | - | - | - |
| settings | W | R | - | R | RW | R | - | R |
| server | RW | R | R | R | R | - | - | R |
| requests | W | - | - | - | - | - | - | R |
| relations | W | R | - | RW | - | R | - | - |
| presets | W | RW | R | - | RW | - | - | - |
| permissions | W | R | - | RW | R | R | - | - |
| notifications | W | RW | RW | - | - | - | - | RW |
| license | W | R | - | R | RW | R | - | R |
| insights | W | - | - | - | - | - | RW | - |
| flows | W | - | - | - | - | RW | - | - |
| files | - | - | RW | - | - | - | - | - |
| fields | W | R | - | RW | R | - | R | - |
| extensions | W | - | - | - | RW | - | - | - |
| collections | W | R | R | RW | R | - | - | - |

## 5. 違反候補・設計所見

- **Store 間循環依存の確認候補** — `collections` が `relations` を利用し、`relations` が `fields` を利用し、`fields` は `collections` と `relations` を利用する。依存方向は `collections → relations → fields → collections` と循環する。実際の呼び出しは action 内など遅延箇所を含むため、初期化時の障害と断定せず、依存グラフの保守性・初期化順序の確認候補とした。
- 画面外の composable / API lifecycle も hydration に結びつけて記録した。共通部品からの store 利用は Directus の共有 UI / interface 設計の一部であり、汎用コンポーネントの利用だけを違反とは判定していない。
- 検索した `.vue` 利用箇所では、`storeToRefs` なしの state 分割代入、store state への直接 `v-model`、store 外からの直接更新は明確には確認できなかった。

## 6. 調査の限界

- 対象を依頼文どおり `app/src/stores/` に限定した。`app/src/ai/stores/`、`views/private/private-view/stores/`、`modules/settings/.../store/` はストア総数には含めたが、詳細調査対象外。
- Directus は大規模なため、UI 内の全プロパティ読み書きを操作行に展開せず、ユーザー操作・画面 lifecycle・連鎖処理ごとに集約した。CSV の行数は個別プロパティアクセスの総件数ではない。
- Vue ファイルの検索結果を中心に追跡した。動的キー、props 経由、外部 extension が追加する route / UI の経路は一部推定を含みうる。
- `app/src/hydrate.ts` で user を先に hydrate し、その後 permissions / fields と残りの stores を並列 hydrate する順序は確認できた。ストア利用の全ての実行時条件までは実行テストで検証していない。
