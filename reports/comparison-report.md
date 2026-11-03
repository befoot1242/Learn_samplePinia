# Pinia ストア使用箇所調査：v1 / v2 比較

## 保存先

| 対象 | v1（無印） | v2 |
| --- | --- | --- |
| RealWorld | [レポート](./v1/vue3-realworld-app/store-usage-report.md) / [CSV](./v1/vue3-realworld-app/store-usage.csv) | [レポート](./v2/vue3-realworld-app/store-usage-report.md) / [CSV](./v2/vue3-realworld-app/store-usage.csv) |
| Soybean Admin | [レポート](./v1/soybean-admin/store-usage-report.md) / [CSV](./v1/soybean-admin/store-usage.csv) | [レポート](./v2/soybean-admin/store-usage-report.md) / [CSV](./v2/soybean-admin/store-usage.csv) |
| Directus | [レポート](./v1/directus/store-usage-report.md) / [CSV](./v1/directus/store-usage.csv) | [レポート](./v2/directus/store-usage-report.md) / [CSV](./v2/directus/store-usage.csv) |

v2 は初回調査時点の資料（親リポジトリ commit `6ef5047`）を保存し、v1 は後続の無印スキルによる調査資料を保存した。v1 Directus CSV は `isAdmin` をストア名としていた1行を、実際のストア `user` と対象プロパティ `isAdmin` に訂正している。

## 件数と記載内容の比較

| リポジトリ | v2 | v1（無印） | 主な差 |
| --- | --- | --- | --- |
| vue3-realworld-app | CSV 15行、1ストア、ルート6、直接更新候補2 | CSV 15行、1ストア、ルート6、直接更新候補2 | アクセス数と候補の場所・内容は同じ。記載内容に実質的な差はない。 |
| soybean-admin | CSV 23行、5ストア、レポート上の操作グループ13。直接 v-model は共通レイアウト中心に記載 | CSV 56行、5ストア、画面グループ15。直接 v-model 34箇所と `homeTab` のリアクティビティ喪失候補1箇所を記載 | テーマ drawer 等の各 v-model を個別明細に展開。v2でまとめていたテーマ設定や共通 UI を細分化し、候補箇所を具体化した。 |
| directus | CSV 37行。複数ストアを1行に含む行が21行あり、ストア単位に展開すると94アクセス。対象ストア16、循環依存候補1行 | CSV 97行、1行1ストアアクセス、対象ストア16、循環依存は3ストアの明細行、リアクティビティ喪失候補5箇所 | ストアごとの行に分割し、Visual editor を追加。分割代入候補を新たに5箇所記録。循環依存は同じ依存関係を関連する各ストア行に記載した。 |

### 判定・読み方

- **RealWorld:** どちらも15行で、設定保存時とログアウト時のストア外直接更新2件を同じ場所で指摘している。
- **Soybean Admin:** v1ではv2にあった一括・代表記載を展開し、実際の34個の store-bound `v-model` を個別記録した。さらに `homeTab` の非 `storeToRefs` 分割代入を候補化した。行数の増分はストア利用が増えたことではなく、アクセス明細の粒度が細かくなったことを含む。
- **Directus:** CSV行数をそのまま比較できない。v2の37行には複数ストアをまとめたものがあるため、内訳を展開した94アクセスと、v1の97行（1行1ストア）を比較する。3行の差は、Visual editor の記録追加などによる。両版のストア対象数は16で一致する。
- 候補件数も数え方が異なる場合がある。特に Directus の循環依存は、v2では1候補行、v1では関係する3ストアそれぞれの行に付いている。同一件を3つの独立した循環依存と解釈しないこと。
- 以上は記録された調査結果の比較であり、行数や候補数の多さが実装品質を直接示すものではない。どちらも静的なコード追跡であり、実行時テストによる確認とは区別する。
