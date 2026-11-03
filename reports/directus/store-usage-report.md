# Pinia ストア使用箇所調査

- 対象: [directus/directus](https://github.com/directus/directus) の `app/`
- 調査対象: 指定された `app/src/stores/` の 16 ストア
- 調査日: 2026-11-04

## 1. 概要

| 項目 | 件数 |
| --- | ---: |
| 対象ストア | 16 |
| app 全体で検出したストア定義 | 22（AI / private-view / field-detail の局所ストアを含む） |
| 操作を確認した画面グループ | 19 |
| 画面別ストアアクセス明細 | 97 |
| 違反候補 | 5（リアクティビティ喪失候補） |
| 設計確認候補 | 1（store 間循環依存） |

`vue` 3.5.43、`pinia` 2.3.1、`vue-router` 4.6.3。バージョンは workspace catalog を参照。persisted-state plugin は確認できず、主なストアは API から hydrate / dehydrate される。アプリ全体の `useAppStore` は `@directus/stores` 由来であり、指定された `app/src/stores/` の 16 件には含めていない。

ルーターは `/setup`、`/login` などの固定 route と、権限に応じて登録される module route を併用する。機能画面は Content、Files、Users、Settings、Flows、Insights、Activity、Deployment、Visual を中心に整理した。

CSV は1行を1ストアアクセスとして記録する。同じ操作で異なるストアに触れる場合は行を分け、同じストア内で同一操作から参照する複数の関連対象は対象欄にまとめている。Vue ファイルでの利用はストアごとに以下の数のファイルで検出した（各ストア間で重複あり。`.vue` 限定）。

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

CSV は1行を1ストアアクセスとして記録する。同一画面・操作でも利用ストアごとに行を分け、画面別にまとめた。

### App navigation (`/ and setup routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Read setup state and hydrate server | navigation | router redirect / beforeEnter | server | RW | info; hydrate | action + state read | app/src/router.ts:31-44 | 確定 | - |

### Authenticated app (`private routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Load current user before dependent stores | lifecycle | hydrate | user | W | hydrate; currentUser; language | action | app/src/router.ts:148-167 -> app/src/hydrate.ts:69 | 確定 | - |
| Load access and field metadata | lifecycle | hydrate | permissions | W | hydrate; permissions | action | app/src/hydrate.ts:74 | 確定 | - |
| Load schema field metadata | lifecycle | hydrate | fields | W | hydrate; fields | action | app/src/hydrate.ts:74 | 確定 | - |
| Load remaining application data | lifecycle | hydrate | collections | W | hydrate; collections | action | app/src/hydrate.ts:30-45,76 | 確定 | - |
| Load remaining application data | lifecycle | hydrate | relations | W | hydrate; relations | action | app/src/hydrate.ts:30-45,76 | 確定 | - |
| Load saved views | lifecycle | hydrate | presets | W | hydrate; collectionPresets | action | app/src/hydrate.ts:30-45,76 | 確定 | - |
| Load settings | lifecycle | hydrate | settings | W | hydrate; settings | action | app/src/hydrate.ts:30-45,76 | 確定 | - |
| Load server configuration | lifecycle | hydrate | server | W | hydrate; info; auth | action | app/src/hydrate.ts:30-45,76 | 確定 | - |
| Load dashboards and flows | lifecycle | hydrate | insights | W | hydrate; dashboards | action | app/src/hydrate.ts:30-45,76 | 確定 | - |
| Load dashboards and flows | lifecycle | hydrate | flows | W | hydrate; flows | action | app/src/hydrate.ts:30-45,76 | 確定 | - |
| Load notifications and license | lifecycle | hydrate | notifications | W | hydrate; unread count | action | app/src/hydrate.ts:30-45,76 | 確定 | - |
| Load notifications and license | lifecycle | hydrate | license | W | hydrate; license info | action | app/src/hydrate.ts:30-45,76 | 確定 | - |

### Logout / auth loss (`private routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Clear hydrated state | auth lifecycle | dehydrate | user | W | dehydrate; currentUser; loading; error | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |
| Clear hydrated state | auth lifecycle | dehydrate | settings | W | dehydrate; settings | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |
| Clear hydrated state | auth lifecycle | dehydrate | server | W | dehydrate; info; auth | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |
| Clear hydrated state | auth lifecycle | dehydrate | permissions | W | dehydrate; permissions | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |
| Clear hydrated state | auth lifecycle | dehydrate | fields | W | dehydrate; fields | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |
| Clear hydrated state | auth lifecycle | dehydrate | collections | W | dehydrate; collections | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |
| Clear hydrated state | auth lifecycle | dehydrate | relations | W | dehydrate; relations | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |
| Clear hydrated state | auth lifecycle | dehydrate | presets | W | dehydrate; collectionPresets | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |
| Clear hydrated state | auth lifecycle | dehydrate | insights | W | dehydrate; dashboards | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |
| Clear hydrated state | auth lifecycle | dehydrate | flows | W | dehydrate; flows | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |
| Clear hydrated state | auth lifecycle | dehydrate | notifications | W | dehydrate; queues; unread | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |
| Clear hydrated state | auth lifecycle | dehydrate | license | W | dehydrate; license state | dehydrate action | app/src/hydrate.ts:94-103 | 確定 | - |

### Private app shell (`private routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Render user/project/theme settings and license state | render / state change | private-view | user | R | currentUser; language; textDirection | property read | app/src/views/private/private-view/components/private-view.vue:57,70-72 | 確定 | - |
| Render user/project/theme settings and license state | render / state change | private-view | server | R | info.project | property read | app/src/views/private/private-view/components/private-view.vue:57,70-72 | 確定 | - |
| Render user/project/theme settings and license state | render / state change | private-view | settings | R | settings | property read | app/src/views/private/private-view/components/private-view.vue:57,70-72 | 確定 | - |
| Render user/project/theme settings and license state | render / state change | private-view | license | R | isLocked | property read | app/src/views/private/private-view/components/private-view.vue:57,70-72 | 確定 | - |
| Track visited page | navigation + timeout | onAfterEach -> trackPage | user | W | trackPage; currentUser.last_page | action + in-store mutation | app/src/router.ts:199-215 | 確定 | - |

### Content navigation (`/content`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Show available collections and bookmarks | render / click | navigation and collection route | collections | R | visibleCollections | property/action read | app/src/modules/content/routes/collection.vue:66,168,350; app/src/modules/content/components/NavigationItem.vue:31-33 | 確定 | - |
| Show available collections and bookmarks | render / click | navigation and collection route | presets | R | collectionPresets; bookmarks | property/action read | app/src/modules/content/routes/collection.vue:66,168,350; app/src/modules/content/components/NavigationItem.vue:31-33 | 確定 | - |
| Show available collections and bookmarks | render / click | navigation and collection route | user | R | isAdmin; currentUser | property/action read | app/src/modules/content/routes/collection.vue:66,168,350; app/src/modules/content/components/NavigationItem.vue:31 | 確定 | リアクティビティ喪失候補: storeToRefs を使わず isAdmin を分割代入 |
| Show available collections and bookmarks | render / click | navigation and collection route | permissions | R | hasPermission; permissions | property/action read | app/src/modules/content/routes/collection.vue:66,168,350; app/src/modules/content/components/NavigationItem.vue:31-33 | 確定 | - |

### Content item (`/content/:collection/:primaryKey`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Render and edit item metadata | render / input / save | item route | fields | R | fields; getField | property read | app/src/modules/content/routes/item.vue:105-107 | 確定 | - |
| Render and edit item metadata | render / input / save | item route | collections | R | collections; getCollection | property read | app/src/modules/content/routes/item.vue:105-107 | 確定 | - |
| Render and edit item metadata | render / input / save | item route | settings | R | settings | property read | app/src/modules/content/routes/item.vue:105-107 | 確定 | - |
| Render and edit item metadata | render / input / save | item route | user | R | currentUser | property read | app/src/modules/content/routes/item.vue:105-107 | 確定 | - |
| Show mutation notifications | API result | item route | notifications | W | add/update notification | action | app/src/modules/content/routes/item.vue:106,982 | 確定 | - |

### Data Model (`/settings/data-model/collections; /settings/data-model/fields; /settings/data-model/field-detail`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Create/update/delete collection | click / submit | collections routes | collections | W | upsertCollection; updateCollection; deleteCollection | action | app/src/modules/settings/routes/data-model/collections/collections.vue:42-45 | 確定 | - |
| Check collection limits and metadata | render / dialog | collection dialog | license | R | entitlements; limits | property/action read | app/src/modules/settings/routes/data-model/collections/collections.vue:43-45 | 確定 | - |
| Check collection limits and metadata | render / dialog | collection dialog | fields | R | fields | property/action read | app/src/modules/settings/routes/data-model/collections/collections.vue:43-45 | 確定 | - |
| Check collection limits and metadata | render / dialog | collection dialog | relations | R | relations | property/action read | app/src/modules/settings/routes/data-model/collections/collections.vue:43-45 | 確定 | - |
| Create/update/delete fields | click / submit | fields management | fields | W | createField; updateField; updateFields; deleteField | action | app/src/modules/settings/routes/data-model/fields/fields.vue:38 | 確定 | - |
| Configure relationship | input / save | field-detail components | fields | RW | fields; getField; updateField | action + property read | app/src/modules/settings/routes/data-model/field-detail/field-detail.vue:50-51; app/src/modules/settings/routes/data-model/field-detail/field-detail-advanced/field-detail-advanced-relationship-m2m.vue:20-21 | 確定 | dependency cycle |
| Configure relationship | input / save | field-detail components | relations | RW | relations; upsertRelation; getRelationForField | action + property read | app/src/modules/settings/routes/data-model/field-detail/field-detail.vue:50-51; app/src/modules/settings/routes/data-model/field-detail/field-detail-advanced/field-detail-advanced-relationship-m2m.vue:20-21 | 確定 | dependency cycle |
| Configure relationship | input / save | field-detail components | collections | RW | collections; getCollection | action + property read | app/src/modules/settings/routes/data-model/field-detail/field-detail.vue:50-51; app/src/modules/settings/routes/data-model/field-detail/field-detail-advanced/field-detail-advanced-relationship-m2m.vue:20-21 | 確定 | dependency cycle |

### Permissions (`/settings/roles/:id`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Edit role/policy access | input / save | permissions UI | permissions | RW | permissions; hasPermission | action + property read | app/src/interfaces/_system/system-permissions/system-permissions.vue:54; app/src/modules/settings/routes/roles/item.vue:33-35 | 確定 | - |
| Edit role/policy access | input / save | permissions UI | collections | RW | collections | action + property read | app/src/interfaces/_system/system-permissions/system-permissions.vue:54; app/src/modules/settings/routes/roles/item.vue:33-35 | 確定 | - |
| Edit role/policy access | input / save | permissions UI | fields | RW | fields | action + property read | app/src/interfaces/_system/system-permissions/system-permissions.vue:54; app/src/modules/settings/routes/roles/item.vue:33-35 | 確定 | - |
| Edit role/policy access | input / save | permissions UI | user | RW | currentUser; isAdmin | action + property read | app/src/interfaces/_system/system-permissions/system-permissions.vue:54; app/src/modules/settings/routes/roles/item.vue:33-35 | 確定 | - |
| Render role navigation authorization | render | NavigationRole | user | R | isAdmin | direct state destructuring | app/src/modules/users/components/NavigationRole.vue:24 | 確定 | リアクティビティ喪失候補: storeToRefs を使わず分割代入 |
| Edit role/policy access | input / save | permissions UI | license | RW | entitlements | action + property read | app/src/interfaces/_system/system-permissions/system-permissions.vue:54; app/src/modules/settings/routes/roles/item.vue:33-35 | 確定 | - |

### Content navigation / presets (`/content and /settings/presets`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Save or select preset/bookmark | click / submit | navigation-bookmark / preset item | presets | RW | collectionPresets; bookmarks; preset actions | action + property read | app/src/modules/content/components/navigation-bookmark.vue:36; app/src/modules/settings/routes/presets/item.vue:53-54 | 確定 | - |
| Save or select preset/bookmark | click / submit | navigation-bookmark / preset item | collections | RW | collections | action + property read | app/src/modules/content/components/navigation-bookmark.vue:36; app/src/modules/settings/routes/presets/item.vue:53-54 | 確定 | - |

### Files (`/files/:collection`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Upload files and update progress | file event / progress | v-upload | files | W | upload; start; progress; done; finish | action + upload-local writable computed | app/src/components/v-upload.vue:140,317 | 確定 | - |
| Render files and upload messages | render / click | file collection | files | RW | activeUploads; isAnyUploadActive | property/action read | app/src/modules/files/routes/collection.vue:51-60,239 | 確定 | - |
| Render files and upload messages | render / click | file collection | user | RW | currentUser | property/action read | app/src/modules/files/routes/collection.vue:51-60,239 | 確定 | - |
| Render files and upload messages | render / click | file collection | server | RW | info.files | property/action read | app/src/modules/files/routes/collection.vue:51-60,239 | 確定 | - |
| Render files and upload messages | render / click | file collection | notifications | RW | add; notifications | property/action read | app/src/modules/files/routes/collection.vue:51-60,239 | 確定 | - |
| Render files and upload messages | render / click | file collection | presets | RW | collectionPresets | property/action read | app/src/modules/files/routes/collection.vue:51-60,239 | 確定 | - |

### Flows (`/flows; /flows/:id`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| List/filter available flows | render | flows overview | flows | R | flows; getManualFlowsForCollection | property/action read | app/src/modules/flows/overview.vue:68,170,180 | 確定 | - |
| List/filter available flows | render | flows overview | permissions | R | hasPermission | property/action read | app/src/modules/flows/overview.vue:68,170,180 | 確定 | - |
| List/filter available flows | render | flows overview | relations | R | relations; relation lookups | property/action read | app/src/modules/flows/overview.vue:68,170,180 | 確定 | - |
| List/filter available flows | render | flows overview | license | R | entitlements | property/action read | app/src/modules/flows/overview.vue:68,170,180 | 確定 | - |
| List/filter available flows | render | flows overview | user | R | currentUser | property/action read | app/src/modules/flows/overview.vue:68,170,180 | 確定 | - |
| Validate flow route exists | navigation | beforeEnter | flows | R | flows | direct state destructuring | app/src/modules/flows/index.ts:34 | 確定 | リアクティビティ喪失候補: storeToRefs を使わず分割代入（同期的な一時参照） |
| Edit, run, or save flow | click / input / save | flow route and drawer | flows | RW | flows; flow actions | action + property read | app/src/modules/flows/flow.vue:65-66; app/src/modules/flows/flow-drawer.vue:50-51 | 確定 | - |
| Edit, run, or save flow | click / input / save | flow route and drawer | license | RW | entitlements; limits | action + property read | app/src/modules/flows/flow.vue:65-66; app/src/modules/flows/flow-drawer.vue:50-51 | 確定 | - |
| Edit, run, or save flow | click / input / save | flow route and drawer | user | RW | currentUser | action + property read | app/src/modules/flows/flow.vue:65-66; app/src/modules/flows/flow-drawer.vue:50-51 | 確定 | - |

### Insights (`/insights`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| View dashboard and panel setup | navigation / render | dashboard and panel routes | insights | R | dashboards; panels; variables | property read | app/src/modules/insights/routes/dashboard.vue:44; app/src/modules/insights/routes/panel-configuration.vue:45; app/src/panels/list/panel-list.vue:34-35 | 確定 | - |
| View dashboard and panel setup | navigation / render | dashboard and panel routes | fields | R | fields; field metadata | property read | app/src/modules/insights/routes/dashboard.vue:44; app/src/modules/insights/routes/panel-configuration.vue:45; app/src/panels/list/panel-list.vue:34-35 | 確定 | - |
| Create/update dashboards and panels | click / submit | dashboard dialog / panel configuration | insights | W | dashboard and panel actions | action | app/src/modules/insights/components/dashboard-dialog.vue:28; app/src/modules/insights/routes/panel-configuration.vue:45 | 確定 | - |

### Settings (`/settings/project; /settings/extensions; /settings/license`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Read and update project settings | navigation / input / save | project settings | settings | RW | settings; updateSettings | action + property read | app/src/modules/settings/routes/project/project.vue:23-24 | 確定 | - |
| Read and update project settings | navigation / input / save | project settings | server | RW | info.project | action + property read | app/src/modules/settings/routes/project/project.vue:23-24 | 確定 | - |
| Refresh/toggle/install extensions | navigation / click | extension routes | extensions | RW | extensions; refresh; install/toggle actions | action + property read | app/src/modules/settings/routes/extensions/extensions.vue:17; app/src/modules/settings/routes/extensions/components/ExtensionItem.vue:31 | 確定 | - |
| Refresh/toggle/install extensions | navigation / click | extension routes | notifications | RW | add; reload notice | action + property read | app/src/modules/settings/routes/extensions/extensions.vue:17; app/src/modules/settings/routes/extensions/components/ExtensionItem.vue:31 | 確定 | - |
| Refresh/toggle/install extensions | navigation / click | extension routes | server | RW | info.extensions | action + property read | app/src/modules/settings/routes/extensions/extensions.vue:17; app/src/modules/settings/routes/extensions/components/ExtensionItem.vue:31 | 確定 | - |
| View license and apply resolution | navigation / click | license page and dialogs | license | RW | license state; resolution actions | action + property read | app/src/modules/settings/routes/license/license.vue:38-39; app/src/modules/settings/routes/license/components/license-resolution-dialog.vue:48,51 | 確定 | - |
| View license and apply resolution | navigation / click | license page and dialogs | user | RW | currentUser; isAdmin | action + property read | app/src/modules/settings/routes/license/license.vue:38-39; app/src/modules/settings/routes/license/components/license-resolution-dialog.vue:48,51 | 確定 | - |
| View license and apply resolution | navigation / click | license page and dialogs | server | RW | info.license | action + property read | app/src/modules/settings/routes/license/license.vue:38-39; app/src/modules/settings/routes/license/components/license-resolution-dialog.vue:48,51 | 確定 | - |
| View license and apply resolution | navigation / click | license page and dialogs | notifications | RW | add; resolution messages | action + property read | app/src/modules/settings/routes/license/license.vue:38-39; app/src/modules/settings/routes/license/components/license-resolution-dialog.vue:48,51 | 確定 | - |

### License enforcement (`private routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Redirect when license becomes locked | navigation / watch | router guard and app.vue | license | R | isLocked | property read + watch | app/src/router.ts:181-196; app/src/app.vue:26-27 | 確定 | - |
| Redirect when license becomes locked | navigation / watch | router guard and app.vue | user | R | isAdmin; currentUser | property read + watch | app/src/router.ts:181-196; app/src/app.vue:26-27 | 確定 | - |

### Notification center (`private shell`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Show unread count and notification dialogs | click / render | notifications drawer | notifications | RW | unread; queue; dialogs; notification actions | action + property read | app/src/views/private/components/notifications-drawer.vue:47-49; app/src/views/private/components/notification-dialogs.vue:19-21 | 確定 | - |
| Show unread count and notification dialogs | click / render | notifications drawer | user | RW | currentUser; isAdmin | action + property read | app/src/views/private/components/notifications-drawer.vue:47-49; app/src/views/private/components/notification-dialogs.vue:19-21 | 確定 | リアクティビティ喪失候補: storeToRefs を使わず isAdmin / currentUser を分割代入 |
| Show unread count and notification dialogs | click / render | notifications drawer | collections | RW | collections; collection names | action + property read | app/src/views/private/components/notifications-drawer.vue:47-49; app/src/views/private/components/notification-dialogs.vue:19-21 | 確定 | - |

### Global request indicator (`private shell`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Show pending API request state | API lifecycle / render | module bar logo | requests | R | queueHasItems; queue | getter/state read | app/src/views/private/components/module-bar-logo.vue:11 | 確定 | - |

### Translated fields (`system field interfaces`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Load or create translated value | input / save | translated-string interface | translations | RW | translations; loadTranslations; create | action + property read | app/src/interfaces/_system/system-input-translated-string/input-translated-string.vue:44,48 | 確定 | - |
| Load or create translated value | input / save | translated-string interface | user | RW | language; currentUser | action + property read | app/src/interfaces/_system/system-input-translated-string/input-translated-string.vue:44,48 | 確定 | - |
| Load or create translated value | input / save | translated-string interface | settings | RW | settings | action + property read | app/src/interfaces/_system/system-input-translated-string/input-translated-string.vue:44,48 | 確定 | - |
| Load or create translated value | input / save | translated-string interface | permissions | RW | hasPermission | action + property read | app/src/interfaces/_system/system-input-translated-string/input-translated-string.vue:44,48 | 確定 | - |

### Visual editor (`visual module route`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Build visual editor URLs | render / state change | useVisualEditorUrls | settings | R | settings | direct state destructuring in computed | app/src/modules/visual/composables/use-visual-editor-urls.ts:7-8 | 確定 | リアクティビティ喪失候補: storeToRefs を使わず分割代入し computed で参照 |

### Shared item view (`/shared/:id`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Render read-only shared collection item | route render | shared route | collections | R | collections; getCollection | property/action read | app/src/routes/shared/shared.vue:129-132 | 確定 | - |
| Render read-only shared collection item | route render | shared route | fields | R | fields; getField | property/action read | app/src/routes/shared/shared.vue:129-132 | 確定 | - |
| Render read-only shared collection item | route render | shared route | relations | R | relations; getRelationsForField | property/action read | app/src/routes/shared/shared.vue:129-132 | 確定 | - |
| Render read-only shared collection item | route render | shared route | permissions | R | hasPermission | property/action read | app/src/routes/shared/shared.vue:129-132 | 確定 | - |

## 4. ストア別マトリクス

`RW` は画面イベントで state 更新と参照の両方があることを示す。全ストアは認証後の全体 hydrate/dehydrate にも参加するものがある。

| ストア | Global lifecycle / auth | Content / item | Files | Data Model / Permissions | Settings / Users | Flows | Insights | Visual editor | 共通 UI |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| user | RW | R | R | R | R | R | R | - | R |
| translations | W | R | - | R | R | - | - | - | - |
| settings | W | R | - | R | RW | R | - | R | R |
| server | RW | R | R | R | R | - | - | - | R |
| requests | W | - | - | - | - | - | - | - | R |
| relations | W | R | - | RW | - | R | - | - | - |
| presets | W | RW | R | - | RW | - | - | - | - |
| permissions | W | R | - | RW | R | R | - | - | - |
| notifications | W | RW | RW | - | - | - | - | - | RW |
| license | W | R | - | R | RW | R | - | - | R |
| insights | W | - | - | - | - | - | RW | - | - |
| flows | W | - | - | - | - | RW | - | - | - |
| files | - | - | RW | - | - | - | - | - | - |
| fields | W | R | - | RW | R | - | R | - | - |
| extensions | W | - | - | - | RW | - | - | - | - |
| collections | W | R | R | RW | R | - | - | - | - |

## 5. 違反候補・設計所見

- **Store 間循環依存の確認候補** — `collections` が `relations` を利用し、`relations` が `fields` を利用し、`fields` は `collections` と `relations` を利用する。依存方向は `collections → relations → fields → collections` と循環する。実際の呼び出しは action 内など遅延箇所を含むため、初期化時の障害と断定せず、依存グラフの保守性・初期化順序の確認候補とした。
- **リアクティビティ喪失候補（5 箇所）** — state / getter の分割代入が `storeToRefs` を使わず行われている。[`NavigationItem.vue`](../../directus/app/src/modules/content/components/NavigationItem.vue) と [`NavigationRole.vue`](../../directus/app/src/modules/users/components/NavigationRole.vue) の `isAdmin`、[`notification-dialogs.vue`](../../directus/app/src/views/private/components/notification-dialogs.vue) の `currentUser` / `isAdmin`、[`modules/flows/index.ts`](../../directus/app/src/modules/flows/index.ts) の `flows`、[`use-visual-editor-urls.ts`](../../directus/app/src/modules/visual/composables/use-visual-editor-urls.ts) の `settings`。`flows` は route guard 内で同期的に読むため影響は限定的とみられるが、他は値の置き換え後に古い参照となる可能性がある。
- 画面外の composable / API lifecycle も hydration に結びつけて記録した。共通部品からの store 利用は Directus の共有 UI / interface 設計の一部であり、汎用コンポーネントの利用だけを違反とは判定していない。
- 対象の16ストアでは store state への直接 `v-model` や store 外からの直接代入を確認できなかった。範囲外の `ai/stores/` および private-view の局所ストアは調査対象外。

## 6. 調査の限界

- 対象を依頼文どおり `app/src/stores/` に限定した。`app/src/ai/stores/`、`views/private/private-view/stores/`、`modules/settings/.../store/` はストア総数には含めたが、詳細調査対象外。
- Directus は大規模なため、CSV は画面・操作単位で特定したストアアクセスを記録し、同じ操作に属する関連 state / getter / action を対象欄にまとめた。テンプレートの全プロパティ参照を個別レコード化したものではない。
- Vue ファイルの検索結果を中心に追跡した。動的キー、props 経由、外部 extension が追加する route / UI の経路は一部推定を含みうる。
- `app/src/hydrate.ts` で user を先に hydrate し、その後 permissions / fields と残りの stores を並列 hydrate する順序は確認できた。ストア利用の全ての実行時条件までは実行テストで検証していない。
