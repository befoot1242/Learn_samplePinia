# Pinia ストア使用箇所調査

- 対象: [soybeanjs/soybean-admin](https://github.com/soybeanjs/soybean-admin)
- 対象範囲: `src/` 全体（ストア 5 件）
- 調査日: 2026-11-04

## 1. 概要

| 項目 | 件数 |
| --- | ---: |
| Pinia ストア | 5 |
| ストア利用ファイル | 53（定義・利用箇所を含む検索結果） |
| 画面・共通 UI / lifecycle の操作グループ | 13 |
| CSV 操作・ストア組み合わせ | 23 |
| 違反・設計確認候補 | 2 |

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

CSV はイベントとストア単位でアクセス対象をまとめたもの。同じ共通レイアウトを全 route で使う場合は「認証後全画面」として集約した。

| 操作 | トリガー | ハンドラ / 経路 | ストア | R/W | 対象 | 場所 | 確度 | 違反候補 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| アプリ起動・ルート初期化 | lifecycle / navigation | `App.vue`, router guard | app, theme, auth, route, tab | RW | 初期状態、利用者、route・tab 初期化 | `src/App.vue:13-14`, `src/router/guard/route.ts:21,65` | 確定 | auth の自己参照 |
| ログイン送信 | submit | `pwd-login` → `authStore.login` | auth | W | token, userInfo, loading | `src/views/_builtin/login/modules/pwd-login.vue:39,73` | 確定 | - |
| 認証情報初期化・route guard | navigation | guard → `initUserInfo` / route 初期化 | auth, route, tab | RW | userInfo, auth routes, menus, tabs | `src/router/guard/route.ts:21,65`; `src/store/modules/route/index.ts:28-29` | 確定 | store 間依存 |
| 認証失効・ログアウト | click / request error | user avatar / request shared → `resetStore` | auth, route, tab | W | auth reset, route reset, cached tabs | `src/layouts/modules/global-header/components/user-avatar.vue:13`; `src/service/request/shared.ts:15` | 確定 | auth の自己参照 |
| 共通レイアウト表示 | 描画 / responsive change | `base-layout` と menu components | app, theme, route | R | viewport/layout/menu/sider/scroll state | `src/layouts/base-layout/index.vue:19-20,121-138`; `src/layouts/modules/global-sider/index.vue:12-13` | 確定 | app state への直接 v-model |
| サイドバー開閉 | click / v-model | `MenuToggler`, `base-layout` | app | W | `siderCollapse` | `src/layouts/modules/global-header/index.vue:35`; `src/layouts/base-layout/index.vue:121` | 確定 | v-model で直接更新 |
| 言語変更 | UI change | header → `changeLocale` | app → route, tab（watch 連鎖） | W | locale, localized menus/tabs/title | `src/layouts/modules/global-header/index.vue:44-47`; `src/store/modules/app/index.ts` | 確定 | watch による連鎖更新 |
| テーマスキーム切替 | click | header → `toggleThemeScheme` | theme | W | `settings.themeScheme` | `src/layouts/modules/global-header/index.vue:50-52`; `src/store/modules/theme/index.ts` | 確定 | - |
| テーマ drawer を開く | click | theme button → `openThemeDrawer` | app | W | `themeDrawerVisible` | `src/layouts/modules/global-header/components/theme-button.vue:9,16` | 確定 | - |
| テーマ・レイアウト設定 | click / input | theme drawer controls → theme actions | theme, app | RW | colors, layout, watermark, appearance, sider config | `src/layouts/modules/theme-drawer/` | 確定 | - |
| タブを追加・切替・閉じる | navigation / click / context menu | global tab component → tab actions | tab, route, app, theme | RW | tabs, activeTabId, route cache, reload | `src/layouts/modules/global-tab/index.vue:174-189`; `src/layouts/modules/global-tab/context-menu.vue:29` | 確定 | - |
| グローバル検索 | click / keyboard | search modal | app, route | R | search visibility, menu / route data | `src/layouts/modules/global-search/components/search-modal.vue:14-15` | 確定 | - |
| ホーム表示・チャート描画 | page render | home / chart modules | app, theme, auth | R | locale, viewport, theme, user display data | `src/views/home/index.vue:11`; `src/views/home/modules/header-banner.vue:11-12` | 確定 | - |

## 4. ストア別マトリクス

| ストア | 起動 / guard | Login | 共通レイアウト | テーマ drawer | タブ / 検索 | Home | request error / logout |
| --- | --- | --- | --- | --- | --- | --- | --- |
| app | RW | R | RW | RW | RW | R | - |
| auth | RW | W | R | - | - | R | W |
| route | RW | R | R | - | RW | - | W |
| tab | RW | - | RW | - | RW | - | W |
| theme | RW | R | R | RW | R | R | - |

## 5. 違反・設計確認候補

- **ストアへの直接 v-model** — [`base-layout/index.vue`](../../soybean-admin/src/layouts/base-layout/index.vue) の `v-model:sider-collapse="appStore.siderCollapse"` は app store state を action を通さず変更する。
- **ストア自己参照** — [`auth/index.ts`](../../soybean-admin/src/store/modules/auth/index.ts) 内で `useAuthStore()` を再度呼び、`resetStore` から `$reset()` している。Setup Store 初期化時の自己参照を含むため、初期化順序・実行時挙動を個別に確認したい。
- **ストア間の相互依存** — auth → route → tab → route および route / tab / theme / auth の相互呼び出しがある。依存を図として維持すると初期化順序の確認に有用。
- locale の watch が route / tab の action を呼ぶ。意図した連鎖更新として記録し、直接の画面操作と区別した。

## 6. 調査の限界

- auto-generated route name / API route mode に依存するため、全 URL を固定一覧にはしていない。`/login` 以外の route は環境設定・生成設定を参照する。
- `theme-drawer` と `global-menu` は複数の小コンポーネントに分割されているため、CSV では機能操作単位にまとめた。
- CSV の 20 行はイベント・ストア単位の整理数であり、テンプレート内の個別プロパティ参照数ではない。
