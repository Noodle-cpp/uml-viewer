# UML viewer

[English documentation](README.md)

## Кратко

Проект показывает кодовую базу как UML. Диаграмма *динамическая*: можно кликать и углубляться вплоть до исходников. Можно играть в «что если» — предлагать изменения архитектуры.

Приложение связано с агентом-помощником. Вы смотрите UML, уточняете замечания — агент правит код. Опишите предложение — агент построит новую диаграмму. Уточняйте, пока не понравится, затем попросите привести код в соответствие.

Цвета — из метрик CRAP и мутаций. Пороги настраиваются в config.

Viewer и агент трактуют namespace как компоненты, модули внутри — элементы компонентов. Вложенность может быть любой глубины. Архитектура строится переносом модулей в нужные namespace с нужными зависимостями.

Уровни архитектуры задаются критериями связности и сцепления для агента (например, группировка по Common Closure Principle).

Зависимости с более высокого уровня на более низкий — красные (Dependency Rule Clean Architecture).

Итого: структура кода, покрытие тестами, what-if и управление системой с высокого уровня.

## Технологии

Живое Quil-приложение: раскладка и отрисовка UML из EDN IR. **Policy** и языковой парсер задают топологию; viewer рисует, прокладывает стрелки, красит CRAP и обрабатывает клики.

IR — **топология**: дерево namespace, классы, рёбра. **Метрики** (CC, coverage, CRAP, killed/survived/uncovered) — из снимков `.metrics/` ([crap4clj](https://github.com/unclebob/crap4clj), [clj-mutate](https://github.com/unclebob/clj-mutate)). Viewer накладывает их при загрузке по namespace и имени функции. Агенты правят policy, не IR. См. [Политика](#политика).

## Запуск

Нужны Clojure CLI и Java 21+.

**В этом репозитории**:

```bash
clj -M:ir                            # policy → examples/uml-viewer.edn
clj -M:run
clj -M:run examples/library.edn
clj -M:run examples/uml-viewer.edn
clj -M:run --help
```

**В любом Clojure-проекте** (локальный checkout uml-viewer не нужен):

```bash
cd /path/to/the-project
curl -fsSL https://raw.githubusercontent.com/unclebob/uml-viewer/master/scripts/get-uml-viewer -o get-uml-viewer
chmod +x get-uml-viewer
./get-uml-viewer               # скачать, записать ./uml, запустить
./uml                          # последующие старты (сразу возвращает управление)
./uml --restart                # только companion: новая JVM, восстановление вида
```

`scripts/get-uml-viewer` в этом репо. Клонирует uml-viewer в **gitignored** `.uml-viewer/uml-viewer/`, пишет `./uml`, запускает viewer. `--install-only` — без старта. `UML_VIEWER_REPO_URL` и `UML_VIEWER_REF` переопределяют источник (по умолчанию `master`).

`./uml` возвращает управление сразу (`clojure` в фоне). Диаграмма — отдельная JVM. Исключения и вывод JVM — в **`uml-viewer-log.txt`** (gitignored). `clj -M:run` держит терминал; предпочтительнее `./uml`.

**tmux**-сессия, уникальная для проекта, запускает интерактивный Grok (`--yolo --trust --rules …`). Имя `uml-viewer-<project>-<hash>` в `.uml-viewer/companion.edn`. При старте пишется иерархическая policy по namespace проекта и перегенерируется IR. Esc — прерывание TUI. Закрытие диаграммы убивает **только эту** tmux-сессию. При падении Grok tmux перезапускает процесс в той же pane. После изменений — `clj -M:crap`, `clj -M:mutate`, генерация IR. Правила проекта — `.grok/rules/uml-viewer.md`.

Предпочтительно `./uml` в анализируемом проекте. Алиасы `:uml-viewer` / `:uml-viewer-restart` работают, если заданы.

| Команда | Кто | Действие |
|---------|-----|----------|
| `./uml` | любой | Новое окно в фоновой JVM. Стартует companion. Ждёт `:display`. |
| `./uml --restart` | **только связанный агент** | Новая JVM, тот же companion. Восстанавливает последний вид. |

**Не** передавайте `--restart`, если вы не companion, перезапускающий окно после правок. Лишний `--restart` не создаст Grok — диаграмма без агента. Companion пишет `:quit-for-restart` в `.uml-viewer/to-viewer.edn`, ждёт выхода JVM, затем `./uml --restart`. Новая JVM восстанавливает глубину, pan, zoom и proposal (`.uml-viewer/session.edn`). Без SIGKILL. Закрытие окна убивает companion только этого проекта.

При чистом старте холст пуст до `:display` от companion — **Waiting for agent to create diagram.** `R` перезагружает текущий EDN сразу. Отсутствующий файл — `UML viewer: file not found: …` в inspector.

```bash
clj -M:spec
clj -M:cov
clj -M:ir                            # пишет examples/uml-viewer.edn
clj -M:crap                          # пишет .metrics/crap.edn
clj -M:mutate src/uml_viewer/engine/layout.clj
```

`:crap` здесь использует `../clojure/crap4clj`. `:mutate` фиксирует [clj-mutate](https://github.com/unclebob/clj-mutate) по git SHA. Коммитьте `.metrics/`, чтобы клон имел числа без перезапуска инструментов.

Переименование или перенос функции — новая форма: overlay не сопоставит старые имена.

## Навигация

**Layer** и **component** — одно и то же: группировка namespace (первый сегмент после prefix или именованная группа proposal).

- Первый вид: **компоненты namespace** (слои). Зависимости между ними схлопываются в одну стрелку. В компоненте — вложенные namespace.
- Двойной клик по компоненту — следующий уровень. Esc или ← — вверх, в том числе из proposal. Esc не выходит из приложения. Закрытие окна или `:quit-for-restart` — выход.
- Наведение на стрелку — popup всех `from -> to` в любом режиме declutter. Нарушающие пары — красные.
- ПКМ по классу или компоненту: **Refresh CRAP**, **Refresh Mutation**, **Refresh All Mutation**, **Omit** — пишут `to-agent.edn` и будят companion. CRAP по классу или файлам компонента. Mutation — дифференциальный `clj -M:mutate`; All Mutation — `--mutate-all`. Omit добавляет id в `:omit` proposal или policy.
- Заголовки компонентов рисуются **поверх** стрелок.
- Inspector: **real diagram** (дерево namespace) над **Proposals**. Клик по real — дерево; по proposal — просмотр (не в коде). Клик пишет `:context`. **New Proposal** — proposal с меткой времени. ПКМ — переименовать/удалить. **Declutter** переключает режимы стрелок и элементов. **Remove arrows** — треугольники вместо линий; красный, если есть нарушающая пара.
- Двойной клик по листовому модулю — **карточка класса**.
- Карточка показывает **модуль** (`:ns`). Клик — файл с начала. Клик по члену — к defn. См. [Извлечение исходников](#извлечение-исходников).
- Методы: `+` public, `-` private. `defn-` на ящике класса не рисуется.
- Абстрактный класс — **α**; интерфейс — **I**. Не-классы — курсив. Foreign из policy — овалы снаружи.
- Окно масштабируется. Scroll — pan; Shift-scroll — горизонтально. **Ctrl+/Ctrl-** — zoom 10%; **Ctrl+0** — 100%.
- **Regen** — companion переписывает policy и IR ([Почтовый ящик companion](#почтовый-ящик-companion)).
- `R` и watcher при сохранении EDN и изменении `.metrics/`. Esc на карточке закрывает её.

## Политика

Диаграмма этого проекта **генерируется**. Не правьте `examples/uml-viewer.edn`. Правьте `examples/uml-viewer.policy.edn`, затем `clj -M:ir` (или Regen).

**Парсер** (`LanguageGraph`) читает исходники и выдаёт факты. Clojure — один класс на namespace. `:require` / `:use` другого namespace проекта — `:dependency`. `requiring-resolve` quoted var — `:dependency` на namespace var. `defprotocol` — `:stereotype :interface`. `defrecord` / `deftype` протокола — `:implements`. Внешние `:require` / `:import` — **foreign** классы. Overlay заполняет члены Clojure из `.metrics/`. TypeScript, Rust, Python — отдельные сканеры ([Графы языков](#графы-языков)): класс на модуль, экспорт как `:ops`. Overlay стыкует снимок по `:ns`, затем по id класса.

### Не выдумывайте слои (компоненты)

Дерево **есть** namespace. После `:prefix` каждая `.` — уровень вложенности. Policy **не** назначает ns выдуманным пакетам. Domain / Engine / Adapters **в дереве исходников** требуют таких сегментов в namespace. **Просмотр** группировки не из кода — `:proposals` ([Предлагаемые компоненты](#предлагаемые-компоненты)), не переписывание namespace.

Как написать policy:

1. `:prefix` — префикс namespace. Один язык — `:src` (по умолчанию `src`) и `:lang`, если не Clojure. Несколько языков — `:sources` вместо `:src`.
2. `:hierarchical true` (или без `:packages` и `:diagrams`).
3. В `:order` — **существующие** top-level сегменты, не новые имена.
4. `:foreign` — реальные библиотеки-овалы.
5. `:edge-kinds {[:from :to] :association}` с **leaf** id.
6. `:levels` для нарушений dependency rule ([Правило зависимостей](#правило-зависимостей)).
7. Опционально `:proposals`.
8. `clj -M:ir` (или Regen).

Если есть `foo.bar` и `foo.bar.baz`, в ящике `bar` — модуль `bar` и дочерний `baz`. Заголовок модуля — последний сегмент (`layout`). Дефисы — CapitalCase частей; существующая capital сохраняется.

Неверно (выдуманные разделы):

```edn
:packages [{:id :domain :nses [ir geom source]}
           {:id :engine :nses [layout route]}]
```

Верно (дерево ns):

```edn
{:title "UML viewer"
 :src "src"
 :prefix "uml-viewer"
 :lang :clojure
 :out "examples/uml-viewer.edn"
 :hierarchical true
 :foreign [quil]
 :order [main adapters application engine source graph
         clojure-language typescript-language rust-language python-language
         domain]
 :levels [[domain source graph clojure-language typescript-language
           rust-language python-language]
          [engine]
          [application]
          [adapters]
          [main]]
 :edge-kinds {[:engine.compose :engine.layout] :association}}
```

Полиглот — каждое дерево в `:sources`. `:prefix` policy срезается с id. `src/model.ts` → `bookwriter.model` (`:model`).

```edn
{:title "Bookwriter"
 :prefix "bookwriter"
 :sources [{:lang :typescript :root "src"}
           {:lang :rust :root "src-tauri/src" :prefix "bookwriter.rust"}]
 :foreign [at.tauri-apps markdown-it highlight.js node tauri serde std]
 :hierarchical true}
```

| Ключ | Роль |
|-----|------|
| `:prefix` | Срезать с каждого ns; остальные `.` — дерево |
| `:hierarchical` | Дерево namespace |
| `:order` | Порядок **существующих** top-level сегментов |
| `:levels` | Группы сегментов, **внутренние (выше) первыми** |
| `:proposals` | Именованные группы сегментов, не в исходниках |
| `:omit` | ns (и дети) вне диаграммы |
| `:edge-kinds` | Переопределить kind для `[from to]` |
| `:omit-edges` | Убрать `[from to]` |
| `:lang` | Сканер без `:sources` (по умолчанию `:clojure`) |
| `:src` | Корень исходников без `:sources` |
| `:sources` | Скан на `{:lang :root :prefix?}` |
| `:foreign` | Внешние libs; префикс схлопывает `quil.core` → `quil` |

**Цикл Grok viewer** (только companion с `--rules`)

При старте: иерархическая policy и IR, затем ожидание.

После **каждого** изменения исходников или policy — метрики и IR.

Clojure: `clj -M:crap`, `clj -M:mutate` на изменённые `src/`, `clj -M:ir`. Непокрытые мутанты — пробелы покрытия. Не перезапускайте из-за non-zero exit mutate.

Другие языки — их CRAP/mutation при наличии. Потом IR.

- namespace add/rename/delete — `clj -M:ir`; новый top-level сегмент — в `:order`.
- «require как association» — `:edge-kinds`.
- quil как oval — `:foreign`.
- Не `:packages` для фейковой Clean Architecture — `:proposals`.
- Сохраняйте `:proposals` при перезаписи policy.

Ручные IR (например `examples/library.edn`) остаются валидными.

### Правило зависимостей

`:dependency` **нарушает**, если идёт с **более высокого** (внутреннего) уровня на **более низкий** (внешний) — зависимости в коде направлены внутрь.

При заданных `:levels`:

1. Первый сегмент каждого конца (`engine.layout` → `engine`).
2. Ранг — индекс группы в `:levels`; **меньше — внутреннее / выше**.
3. Оба с рангом и `from-rank < to-rank` → `:violating true`. `:implements` и `:association` не нарушают.
4. Схлопнутые стрелки сохраняют флаг, если любая leaf-зависимость нарушала.

`:order` — порядок ящиков, не ранг. Без `:levels` ничего не помечается. Без `:levels`, но с `:proposals` — ранг по первому proposal.

### Предлагаемые компоненты

`:proposals` — список именованных групп **существующих** top-level сегментов (не namespace). Элемент `{:id :name :layers [...] :omit [...]}` (`:omit` убирает ns из **Unassigned**). Диаграмма as-is остаётся деревом ns. Клик по proposal помечает холст **PROPOSAL — not instantiated in code** и задаёт контекст обсуждения. **New Proposal** создаёт пустой proposal с меткой времени. Двойной клик по ns — drill; **←** возвращает к proposal.

Кнопка **Declutter**: Declutter arrows → Remove arrows → Declutter elements → Declutter classes → Declutter none. На proposal ранг берётся из порядка компонентов proposal; на real diagram — из `:levels`. Уровень 0 рисуется **внизу**; «хорошие» стрелки (внешний → внутренний) вниз, нарушающие — вверх и красные. Ящики классов показывают rank (внутренний **0**) слева сверху; на карточке — **Level n**.

```edn
:proposals [{:id :ccp
             :name "2026-09-18 10:30:00"
             :layers [{:id :playfield :label "Playfield"
                       :nses [entities world missiles cities batteries flyers]}
                      {:id :hosts :label "Hosts" :nses [jvm browser]}
                      {:id :jvm :label "JVM"
                       :nses [jvm.cli jvm.main
                              {:id :quil-swing :label "Quil/Swing"
                               :nses [jvm.sketch jvm.window]}]}]
             :omit [cli]}]
```

Companion не выдумывает `:proposals` при старте. Нарушающие стрелки — **красные**, **жирно красные** при выделении.

## Почтовый ящик companion

Обмен через `.uml-viewer/` (gitignored). Файл — почта; tmux — «дверной звонок».

| Файл | Направление |
|------|-------------|
| `.uml-viewer/to-viewer.edn` | Grok → viewer |
| `.uml-viewer/to-agent.edn` | viewer → Grok |

Очередь `{:next-id n :queue [cmd …]}`. Команды с растущим `:id`. Обработка **pop** из `:queue`. `.uml-viewer/session.edn` — последний вид. `.uml-viewer/companion.edn` — tmux и окно Terminal.

| `:op` | Смысл |
|-------|--------|
| `:display` | Viewer грузит `:path` |
| `:regen` | Policy, IR, `:display` |
| `:quit-for-restart` | Выход JVM без убийства Grok |
| `:refresh-crap` | CRAP на `:target`, затем IR |
| `:refresh-mutate` | Дифф. mutate, затем IR |
| `:refresh-mutate-all` | `--mutate-all`, затем IR |
| `:omit` | Добавить id в `:omit`, затем IR |
| `:context` | Контекст обсуждения в inspector |

**Regen** ставит `:regen` и будит Grok (150ms, `C-m`, 50ms, `C-j`).

## Графы языков

IR генерируется сканом дерева через `uml-viewer.graph`. Регистрация: `(graph/register! :java my-java-scanner)`. Протокол `LanguageGraph`: метод `scan` → `{:classes :edges}`.

Классы `{:id :name :ns :stereotype :lang :file}`. Рёбра `{:from :to :kind}`. Policy языконейтральна.

**Clojure** — `ns`, `requiring-resolve`, `defprotocol`, `defrecord`, `deftype`. Без `:ops`.

**TypeScript** — класс на модуль; импорты, foreign, `:implements` по типам проекта, `invoke("name")` для связи с Rust.

**Rust** — модули от `lib.rs`/`main.rs`; `pub fn` и `#[tauri::command]` как `:ops`; `impl Trait for Type` как `:implements`.

**Python** — класс на `.py`; импорты; наследование и `Protocol`/`ABC`; публичные `def`/`class` как `:ops`.

`merge-scans` связывает TS `invoke("read_text")` с Rust `#[tauri::command] fn read_text`. CRAP/mutation для TS/Rust/Python — отдельные инструменты; overlay по `:ns` и id. Нет данных — красный.

## IR

Иерархическая policy — один EDN всех классов и рёбер. Ручной IR с `:packages` — статическая диаграмма (`examples/library.edn`).

Метрики на карточке из `.metrics/` или из authored `:crap` / `:coverage` / `:ops`.

```edn
{:title "Lending library"
 :direction :tb
 :packages
 [{:id :domain
   :label "Domain"
   :classes
   [{:id :book
     :name "Book"
     :stereotype :class
     :fields [{:name "isbn" :type "String"}]
     :ops [{:name "find" :args ["isbn"] :returns "Book"}]}]}
  {:id :app
   :label "Application"
   :classes
   [{:id :repo
     :name "CatalogRepo"
     :stereotype :interface
     :ops [{:name "get" :args ["isbn"] :returns "Book"}]}]}]
 :edges
 [{:from :sql-repo :to :repo :kind :implements}
  {:from :loan :to :book :kind :association :label "borrows"}]}
```

**Цвет** — CRAP (μ+σ) и mutation 1–10 из `uml-viewer.domain.config`, среднее, красно-зелёная шкала. Нет данных — красный (1). Точки **C** и **M** в углу.

На карточке: `--crap--`, `--mutation--`; `---no mutation sites---` при `:sites` 0.

| kind | линия | наконечник |
|------|------|------|
| `:inheritance` | серая | пустой треугольник |
| `:implements` | серая | пустой треугольник |
| `:association` | серая | открытая стрелка |
| `:dependency` | серая | открытая стрелка |
| `:aggregation` | серая | пустой ромб |
| `:composition` | серая | закрашенный ромб |

Раскладка в три этапа (как Mermaid): размер → размещение Sugiyama → маршрутизация ортогональная с `curveBasis`. Нарушающие зависимости — красные.

## Извлечение исходников

Клик по члену — `uml-viewer.source` → `{:title :file :body :line}`. Регистрация: `(source/register! :java my-java-extractor)`. Протокол `LanguageSource`: `locate`, `extract`, `title`.

```clojure
(source/member-source {:lang :clojure
                       :ns "uml-viewer.engine.layout"
                       :name "layout"})
```

**Clojure** — путь из `:ns`, поиск `(defn name …)`. **TypeScript / Rust / Python** — `:file` и объявление. Quil только в `adapters.draw` и `adapters.sketch`.
