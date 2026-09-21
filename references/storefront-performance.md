# Storefront performance contracts

## Measure the whole request

Use separate Yii profiles for the initial HTML and each PJAX request, and compare
like-for-like guest/authenticated sessions. SQL duration is only part of request
time. A reduction in SQL count alone is not a measured page speedup. Normalize
SQL literals before saving diagnostics; never print sessions, cookies or raw
configuration from debug files.

## HTML compression

`skeeks/yii2-assets-auto-compress` owns `HtmlCompressor` and the Tyler formatter.
Read string input using an advancing byte offset. Copying the remaining document
for every line makes multiline HTML compression quadratic in document size.
Preserve existing whitespace, comment/extra flags, pre/textarea handling and
statistics behavior; verify output equivalence against the previous implementation.
The package's `tests/html-compressor.php` checks fixed cases and a large document.

## Product lists

`skeeks/cms-shop/helpers/ProductCardData` owns the snapshot of stock rows,
favorite flags and comparison flags for one rendered page/list. It queries only
the listed product IDs, uses the existing cart/shop-user owner relations, and
uses the current shop's stores. An empty store list intentionally preserves
`ShopProduct::getShopStoreProducts([])` semantics (no store restriction).

Never put this snapshot in shared cache or retain it across user/store changes
or mutations. Load a fresh snapshot for the next render. Do not replace the
unfiltered `shopStoreProducts` ActiveRecord relation with a filtered subset.
The snapshot keeps filtered stock separately and does not change price,
discount, offer selection or permission logic.

The existing `theme-unify-shop` catalog and product slider preload `image`,
`images`, `shopProduct.baseProductPrice` and `shopProduct.shopProductPrices`, then
pass the snapshot through Yii ListView `viewParams`. Individual/custom card
renders retain the old query path when no matching snapshot is supplied. Theme
updates tolerate an older cms-shop without ProductCardData; stock/flag batching
starts when the helper becomes available. Project-overridden list templates must
adopt the same preload/viewParams contract to obtain the same improvement.

Reuse ActiveDataProvider's current-page models for emptiness checks; do not run
`query->one()` before rendering the same list. Preserve query filters, ordering,
limits, page selection and existing price calculation.

## Verification

Run PHP 8.2 checks with the project's Composer autoloader, without bootstrapping
a production application:

- `cms-shop/tests/product-card-data.php <vendor/autoload.php>` uses an isolated
  in-memory SQLite fixture and real Yii SQL/owner relations. It compares batched
  results with individual queries, including users, guests, store scopes, absent
  and negative stock, empty lists and fresh reads after mutation.
- `theme-unify-shop/tests/product-lists.php <vendor/autoload.php>
  <cms-shop/tests/product-card-data.php>` runs the real shared list templates,
  ListView and ActiveDataProvider with isolated fixtures. Enable
  `short_open_tag=1`. The per-card markup is intercepted: this checks data flow,
  eager loading and pagination, not full visual rendering or real discount rules.

After deployment, measure actual page and PJAX profiles again. Do not present
microbenchmarks or fixture query counts as production before/after results.

## CSS dependency boundary

For CSS compression dependency ownership in `skeeks/yii2-assets-auto-compress`,
read that package's `AGENTS.md` and `src/vendor/mrclay/README.md`. Its namespaced
CSS subset is self-contained; the legacy Mrclay HTML adapter is optional.
Copying PHP files to production does not update the consumer's Composer lock.

## Списки коллекций

В `theme-unify-shop` шаблоны `collections/collection-list.php` и
`collections/collection-list-no-page.php` добавляют `image`, `images`,
`shopCollectionStickers`, `brand.country` в eager loading запроса провайдера
до загрузки его моделей. Сохранять текущие фильтры, сортировку, пагинацию,
порядок изображений и пустые связи. Это пакетная загрузка в рамках запроса,
а не общий кеш; переопределённые проектом шаблоны должны принять тот же подход.
`theme-unify-shop/tests/collection-lists.php <vendor/autoload.php>` проверяет
оба настоящих шаблона и ListView на изолированной SQLite; включать
`short_open_tag=1`. Тест проверяет отсутствие SQL при чтении связей карточек,
а HTML реальных карточек следует сравнивать отдельно на сайте.

На главной два блока коллекций имеют одинаковые фильтры и GROUP BY/HAVING,
но разные сортировки. Можно передавать один общий полный totalCount обоим
провайдерам; не считать только ограниченную четвёрку строк. JOIN для подсчёта
товаров не требует eager loading самих товаров. Тест
`theme-unify-shop/tests/home-collections.php <vendor/autoload.php>` проверяет
настоящий блок главной, общий COUNT, обе сортировки, следующие страницы,
пустой набор и отсутствие загрузки неиспользуемых моделей товаров.

## Характеристики карточки товара

Штатный WidgetRenderable передаёт params в представление. В theme-unify-shop
варианты _product-info-v1/v2/v3 передают готовые rpAttributes в params виджета:
проверка наличия блока и его вывод используют один результат форматирования.
RpWidget/default и two-columns используют переданный массив, включая пустой,
а при отсутствии параметра сохраняют прежний getter виджета. Это данные одного
рендера; не переносить их в статический или общий кеш и не менять глобальные
getAttributeAsHtml/getAttributeAsText ради устранения повторов шаблона.
Проектные представления должны сами принять переданный параметр.
Единицы measureMatches в _product-price загружаются по набору кодов только
в отображаемой ветке; сохранять порядок исходного массива пересчётов.
Проверки: theme-unify-shop/tests/product-properties-render.php и
product-measures.php, с автозагрузчиком Composer и short_open_tag=1.
Первый тест также принимает путь src/views предыдущей версии для сравнения HTML.

## Кеш блоков коллекций главной

`theme-unify-shop` кеширует оба блока коллекций одним HTML-фрагментом на 28800
секунд с зависимостью от `Yii::$app->skeeks->site->cacheTag`, как гребёнка брендов.
Это выбранный ручной режим: изменения коллекций видны после истечения срока
или очистки кеша сайта, а не автоматически после сохранения коллекции.
SQL и проверка пустого набора находятся внутри фрагмента. Регистрировать CSS,
VanillaLazyLoadAsset, ShopUnifyProductCardAsset и ProductListImagesAsset вне кеша:
обычный FragmentCache не повторяет вызовы регистрации из пропущенного шаблона.
Ключ разделяет сайт, страницу главной, язык, host/baseUrl, параметры пагинации
и сортировки, класс темы и параметры карточек. При расширении персонализированного
содержимого пересмотреть ключ или вынести его из общего фрагмента.
`tests/home-collections-cache.php` проверяет срок, отсутствие SQL на попадании,
сброс тегом сайта, разделение вариантов, пустой набор и регистрацию CSS/ассетов.

## SQL в правилах URL

Правила кассы и склада в cms-shop выбирают backendShopStore только после
проверки своего urlPrefix по тому же условию, что BackendUrlRule. Выбор должен
предшествовать parent::parseRequest(): родитель запускает backend->run(),
которому уже может требоваться склад. Не переносить выбор после вызова родителя.
Повторно использовать найденную модель в текущем вызове, без общего кеша.
Проверка cms-shop/tests/store-url-rules.php использует реальные правила и
родительский маршрутизатор с изолированной SQLite: чужие URL без SQL, выбор
по сайту/параметру, пустой набор и наличие склада при запуске backend.

## Повторное чтение контента и основного домена

На главной theme-unify-shop один результат shopContents используется для
проверки и обоих товарных блоков; глобальный getter не мемоизируется.
Контракт кеша CmsSite::getCmsSiteMainDomain и его тест описаны в cms/AGENTS.md.
Кнопка AdminCacheController инвалидирует тег сайта и тег схемы БД, а не все
табличные теги. Общий кеш с ручным сбросом должен зависеть от тега сайта;
автоматическая инвалидация модели отдельно использует HasTableCache.
