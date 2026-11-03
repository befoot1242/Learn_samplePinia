# Pinia ストア使用箇所調査

- 対象: [soybeanjs/soybean-admin](https://github.com/soybeanjs/soybean-admin)
- 対象範囲: `src/` 全体（ストア 5 件）
- 調査日: 2026-11-04

## 1. 概要

| 項目 | 件数 |
| --- | ---: |
| Pinia ストア | 5 |
| ストア利用ファイル | 53（定義・利用箇所を含む検索結果） |
| 画面グループ | 15 |
| 画面別ストアアクセス明細 | 56 |
| 違反候補 | 35（直接 v-model 34、リアクティビティ喪失 1） |
| 設計確認候補 | 2（ストア自己参照、ストア間循環依存） |
| 連鎖更新候補 | 1（locale watcher から route / tab 更新） |

`vue` 3.5.43、`pinia` 4.0.3、`vue-router` 5.4.0。`SetupStoreId` で各ストア ID を定義し、すべて Setup Store。`pinia-plugin-persistedstate` は依存にない。永続値は `localStg` などで個別に扱う。

ルーターは `createBuiltinVueRoutes()` と生成ルートを利用する。`/login` のほか、認証後のホーム・各機能画面は route generator / サーバー route mode によって決まるため、画面操作は共通レイアウト・テーマ drawer・タブ・認証ガード単位でまとめた。

## 2. ストア一覧

| ID / export | 種別 | state / getter | 主な action | 依存ストア | 永続化 |
| --- | --- | --- | --- | --- | --- |
| `app-store` / `useAppStore` | Setup | `isMobile`, `reloadFlag`, `fullContent`, `locale`, `themeDrawerVisible`, `contentXScrollable`, `siderCollapse`, `mixSiderFixed`; locale 候補 | `reloadPage`, `changeLocale`, drawer open/close, content / sider toggle | theme, route, tab | locale・mix sider 設定を localStg |
| `auth-store` / `useAuthStore` | Setup | `token`, `userInfo`, `isStaticSuper`, `isLogin`, `loginLoading` | `login`, `initUserInfo`, `resetStore` | auth（自己参照）, route, tab | token / refresh token / user ID を localStg |
| `route-store` / `useRouteStore` | Setup | routes, menus, search menus, cache routes, breadcrumbs, route mode/home | route init、menu/cache 更新、`resetStore` | auth, tab | 画面状態はメモリ |
| `tab-store` / `useTabStore` | Setup | tabs, home tab, all tabs, active tab ID | `initTabStore`, add/remove/clear/switch/replace, `cacheTabs` | route, theme | 設定に応じて globalTabs を localStg |
| `theme-store` / `useThemeStore` | Setup | settings の各 ref、`darkMode`, `themeColors`, `naiveTheme`, `settingsJson`, watermark 表示値 | scheme/color/layout/watermark 設定、`resetStore` | auth | themeSettings / darkMode / themeColor 等を localStg |

定義: [`store/modules/`](../../soybean-admin/src/store/modules/)、ID: [`enum/index.ts`](../../soybean-admin/src/enum/index.ts)

## 3. 画面別・操作単位の一覧

CSV は1行を1ストアアクセスとして記録する。同一画面・操作でも利用ストアごとに行を分け、画面別にまとめた。

### Application startup (`all routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Initialize app-level stores | lifecycle | App setup | app | RW | locale; responsive state; drawer and layout state | store initialization + actions | src/App.vue:13-14; src/store/modules/app/index.ts | 確定 | - |
| Initialize app-level stores | lifecycle | App setup | theme | RW | settings; darkMode; themeColors | store initialization + watch | src/App.vue:14; src/store/modules/theme/index.ts | 確定 | - |

### Route guard (`all routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Initialize authentication | navigation | createRouterGuard | auth | RW | token; userInfo; login state | action | src/router/guard/route.ts:21; src/store/modules/auth/index.ts | 確定 | self-reference candidate |

### Login (`/login`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Submit credentials | submit | password login | auth | W | login; token; userInfo | action | src/views/_builtin/login/modules/pwd-login.vue:39,73; src/store/modules/auth/index.ts | 確定 | - |

### Login / route guard (`/login; protected routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Fetch user and permitted routes | navigation | route guard | route | RW | authRoutes; menus; cacheRoutes; breadcrumbs | action | src/router/guard/route.ts:65; src/store/modules/route/index.ts:27-29 | 確定 | cross-store dependency |

### Logout (`authenticated routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Clear authentication and route state | click / request error | user-avatar / request shared | auth | W | resetStore; userInfo; token | action | src/layouts/modules/global-header/components/user-avatar.vue:13; src/service/request/shared.ts:15 | 確定 | self-reference candidate |
| Reset cached routes and tabs | click / request error | authStore.resetStore | route | W | resetStore; route cache | action | src/store/modules/auth/index.ts; src/store/modules/route/index.ts | 確定 | cross-store dependency |
| Cache or clear tabs | click / request error | authStore.resetStore | tab | W | cacheTabs; clearTabs | action | src/store/modules/auth/index.ts; src/store/modules/tab/index.ts | 確定 | cross-store dependency |

### Shared layout (`all authenticated routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Render responsive layout | render / viewport change | base layout | app | RW | isMobile; fullContent; contentXScrollable | property read | src/layouts/base-layout/index.vue:19,121-138 | 確定 | - |
| Render layout theme and dimensions | render | base layout | theme | R | layout; sider; header; tab; footer settings | property read | src/layouts/base-layout/index.vue:20,28-32,70-138 | 確定 | - |
| Edit appStore.siderCollapse with v-model | input change | v-model | app | W | siderCollapse | direct v-model | layouts/base-layout/index.vue:121 | 確定 | ストア state への直接 v-model |

### Sidebar (`all authenticated routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Click sidebar toggler | click / v-model | MenuToggler | app | W | toggleSiderCollapse | action | src/layouts/modules/global-header/index.vue:35 | 確定 | - |

### Header (`all authenticated routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Change language | input change | change-lang -> changeLocale | app | W | locale; changeLocale | action | src/layouts/modules/global-header/index.vue:44-47; src/store/modules/app/index.ts | 確定 | watch連鎖更新 |

### Language watcher (`all routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Refresh translated menus and tabs | watch | app locale watcher | route | W | updateGlobalMenusByLocale | action | src/store/modules/app/index.ts | 確定 | watch連鎖更新 |
| Refresh translated menus and tabs | watch | app locale watcher | tab | W | updateTabsByLocale | action | src/store/modules/app/index.ts | 確定 | watch連鎖更新 |

### Theme controls (`all authenticated routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Switch color scheme | click | theme switch | theme | W | toggleThemeScheme; settings.themeScheme | action | src/layouts/modules/global-header/index.vue:50-52; src/store/modules/theme/index.ts | 確定 | - |

### Theme drawer (`all authenticated routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Open theme settings | click | theme button | app | W | openThemeDrawer; themeDrawerVisible | action | src/layouts/modules/global-header/components/theme-button.vue:9,16 | 確定 | - |
| Edit appStore.themeDrawerVisible with v-model | input change | v-model | app | W | themeDrawerVisible | direct v-model | layouts/modules/theme-drawer/index.vue:31 | 確定 | ストア state への直接 v-model |

### Tab bar (`all authenticated routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Add/switch/close tabs | navigation / click | global tab | tab | RW | tabs; activeTabId; cacheTabs; clearTabs | actions | src/layouts/modules/global-tab/index.vue:174-189; src/layouts/modules/global-tab/context-menu.vue:29 | 確定 | リアクティビティ喪失候補: `homeTab` を `storeToRefs` なしで分割代入 |
| Reset route component cache | click | tab removal / reload | route | W | resetRouteCache; cacheRoutes | action | src/layouts/modules/global-tab/index.vue:118-122 | 確定 | - |

### Global search (`all authenticated routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Render and search routes | click / keyboard | search modal | route | R | searchMenus; route metadata | property read | src/layouts/modules/global-search/components/search-modal.vue:14-15 | 確定 | - |

### Home (`home route`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Render locale-aware dashboard | render | home and chart modules | app | R | locale; isMobile | property read | src/views/home/index.vue:11; src/views/home/modules/pie-chart.vue:11 | 確定 | - |
| Render themed dashboard | render | home and chart modules | theme | R | themeColor; darkMode | property read | src/views/home/modules/card-data.vue:76; src/hooks/common/echarts.ts:86-102 | 確定 | - |
| Render user display information | render | header banner | auth | R | userInfo | property read | src/views/home/modules/header-banner.vue:12 | 確定 | - |

### Theme drawer controls (`all authenticated routes`)

| 操作 | トリガー種別 | ハンドラ | ストア | R/W | 対象 | 経路 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Edit themeStore.layout.scrollMode with v-model | input change | v-model | theme | W | layout.scrollMode | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/content-settings.vue:26 | 確定 | ストア state への直接 v-model |
| Edit themeStore.page.animate with v-model | input change | v-model | theme | W | page.animate | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/content-settings.vue:33 | 確定 | ストア state への直接 v-model |
| Edit themeStore.page.animateMode with v-model | input change | v-model | theme | W | page.animateMode | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/content-settings.vue:37 | 確定 | ストア state への直接 v-model |
| Edit themeStore.fixedHeaderAndTab with v-model | input change | v-model | theme | W | fixedHeaderAndTab | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/content-settings.vue:44 | 確定 | ストア state への直接 v-model |
| Edit themeStore.footer.visible with v-model | input change | v-model | theme | W | footer.visible | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/footer-settings.vue:24 | 確定 | ストア state への直接 v-model |
| Edit themeStore.footer.fixed with v-model | input change | v-model | theme | W | footer.fixed | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/footer-settings.vue:31 | 確定 | ストア state への直接 v-model |
| Edit themeStore.footer.height with v-model | input change | v-model | theme | W | footer.height | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/footer-settings.vue:34 | 確定 | ストア state への直接 v-model |
| Edit themeStore.footer.right with v-model | input change | v-model | theme | W | footer.right | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/footer-settings.vue:41 | 確定 | ストア state への直接 v-model |
| Edit themeStore.header.height with v-model | input change | v-model | theme | W | header.height | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/header-settings.vue:17 | 確定 | ストア state への直接 v-model |
| Edit themeStore.header.breadcrumb.visible with v-model | input change | v-model | theme | W | header.breadcrumb.visible | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/header-settings.vue:20 | 確定 | ストア state への直接 v-model |
| Edit themeStore.header.breadcrumb.showIcon with v-model | input change | v-model | theme | W | header.breadcrumb.showIcon | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/header-settings.vue:27 | 確定 | ストア state への直接 v-model |
| Edit themeStore.layout.mode with v-model | input change | v-model | theme | W | layout.mode | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/layout-mode.vue:17 | 確定 | ストア state への直接 v-model |
| Edit themeStore.sider.width with v-model | input change | v-model | theme | W | sider.width | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/sider-settings.vue:22 | 確定 | ストア state への直接 v-model |
| Edit themeStore.sider.collapsedWidth with v-model | input change | v-model | theme | W | sider.collapsedWidth | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/sider-settings.vue:25 | 確定 | ストア state への直接 v-model |
| Edit themeStore.sider.mixWidth with v-model | input change | v-model | theme | W | sider.mixWidth | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/sider-settings.vue:28 | 確定 | ストア state への直接 v-model |
| Edit themeStore.sider.mixCollapsedWidth with v-model | input change | v-model | theme | W | sider.mixCollapsedWidth | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/sider-settings.vue:31 | 確定 | ストア state への直接 v-model |
| Edit themeStore.sider.mixChildMenuWidth with v-model | input change | v-model | theme | W | sider.mixChildMenuWidth | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/sider-settings.vue:34 | 確定 | ストア state への直接 v-model |
| Edit themeStore.sider.autoSelectFirstMenu with v-model | input change | v-model | theme | W | sider.autoSelectFirstMenu | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/sider-settings.vue:40 | 確定 | ストア state への直接 v-model |
| Edit themeStore.tab.visible with v-model | input change | v-model | theme | W | tab.visible | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/tab-settings.vue:19 | 確定 | ストア state への直接 v-model |
| Edit themeStore.tab.cache with v-model | input change | v-model | theme | W | tab.cache | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/tab-settings.vue:25 | 確定 | ストア state への直接 v-model |
| Edit themeStore.tab.height with v-model | input change | v-model | theme | W | tab.height | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/tab-settings.vue:28 | 確定 | ストア state への直接 v-model |
| Edit themeStore.tab.mode with v-model | input change | v-model | theme | W | tab.mode | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/tab-settings.vue:32 | 確定 | ストア state への直接 v-model |
| Edit themeStore.tab.closeTabByMiddleClick with v-model | input change | v-model | theme | W | tab.closeTabByMiddleClick | direct v-model | layouts/modules/theme-drawer/modules/layout/modules/tab-settings.vue:42 | 確定 | ストア state への直接 v-model |
| Edit themeStore.header.multilingual.visible with v-model | input change | v-model | theme | W | header.multilingual.visible | direct v-model | layouts/modules/theme-drawer/modules/general/modules/global-settings.vue:16 | 確定 | ストア state への直接 v-model |
| Edit themeStore.header.globalSearch.visible with v-model | input change | v-model | theme | W | header.globalSearch.visible | direct v-model | layouts/modules/theme-drawer/modules/general/modules/global-settings.vue:20 | 確定 | ストア state への直接 v-model |
| Edit themeStore.watermark.visible with v-model | input change | v-model | theme | W | watermark.visible | direct v-model | layouts/modules/theme-drawer/modules/general/modules/watermark-settings.vue:23 | 確定 | ストア state への直接 v-model |
| Edit themeStore.watermark.timeFormat with v-model | input change | v-model | theme | W | watermark.timeFormat | direct v-model | layouts/modules/theme-drawer/modules/general/modules/watermark-settings.vue:37 | 確定 | ストア state への直接 v-model |
| Edit themeStore.watermark.text with v-model | input change | v-model | theme | W | watermark.text | direct v-model | layouts/modules/theme-drawer/modules/general/modules/watermark-settings.vue:45 | 確定 | ストア state への直接 v-model |
| Edit themeStore.recommendColor with v-model | input change | v-model | theme | W | recommendColor | direct v-model | layouts/modules/theme-drawer/modules/appearance/modules/theme-color.vue:58 | 確定 | ストア state への直接 v-model |
| Edit themeStore.isInfoFollowPrimary with v-model | input change | v-model | theme | W | isInfoFollowPrimary | direct v-model | layouts/modules/theme-drawer/modules/appearance/modules/theme-color.vue:67 | 確定 | ストア state への直接 v-model |
| Edit themeStore.themeRadius with v-model | input change | v-model | theme | W | themeRadius | direct v-model | layouts/modules/theme-drawer/modules/appearance/modules/theme-radius.vue:17 | 確定 | ストア state への直接 v-model |
| Edit themeStore.sider.inverted with v-model | input change | v-model | theme | W | sider.inverted | direct v-model | layouts/modules/theme-drawer/modules/appearance/modules/theme-schema.vue:54 | 確定 | ストア state への直接 v-model |

## 4. ストア別マトリクス

| ストア | 起動 / guard | Login | 共通レイアウト | テーマ drawer | タブ / 検索 | Home | request error / logout |
| --- | --- | --- | --- | --- | --- | --- | --- |
| app | RW | R | RW | RW | RW | R | - |
| auth | RW | W | R | - | - | R | W |
| route | RW | R | R | - | RW | - | W |
| tab | RW | - | RW | - | RW | - | W |
| theme | RW | R | R | RW | R | R | - |

## 5. 違反・設計確認候補

- **ストアへの直接 v-model（34 箇所）** — [`base-layout/index.vue`](../../soybean-admin/src/layouts/base-layout/index.vue) の sidebar binding、[`theme-drawer/`](../../soybean-admin/src/layouts/modules/theme-drawer/) の theme / layout / watermark controls が store state を action を通さず変更する。編集途中の値も即時に store に反映される。
- **ストア自己参照** — [`auth/index.ts`](../../soybean-admin/src/store/modules/auth/index.ts) 内で `useAuthStore()` を再度呼び、`resetStore` から `$reset()` している。Setup Store 初期化時の自己参照を含むため、初期化順序・実行時挙動を個別に確認したい。
- **ストア間の相互依存** — auth → route → tab → route および route / tab / theme / auth の相互呼び出しがある。依存を図として維持すると初期化順序の確認に有用。
- **リアクティビティ喪失候補** — [`global-tab/context-menu.vue`](../../soybean-admin/src/layouts/modules/global-tab/context-menu.vue) が `homeTab` state を `storeToRefs` なしで分割代入し、computed 内から参照している。
- locale の watch が route / tab の action を呼ぶ。意図した連鎖更新として記録し、直接の画面操作と区別した（watch による連鎖更新候補）。

## 6. 調査の限界

- auto-generated route name / API route mode に依存するため、全 URL を固定一覧にはしていない。`/login` 以外の route は環境設定・生成設定を参照する。
- `theme-drawer` と `global-menu` は複数の小コンポーネントに分割されているため、CSV では機能操作単位にまとめた。
- CSV の 56 行は特定できた store access を記録する。複数 template access を一つの store property に集約した値ではなく、検出した store-bound `v-model` は各 binding を個別に記録した。
