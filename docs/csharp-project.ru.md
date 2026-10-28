# C# / .NET проект для uml-viewer

Краткий чеклист: как показать **C#** на диаграмме и наложить **CRAP / mutation / dry** из `.metrics/`.

Viewer сканирует `.cs` (regex, как TypeScript/Python), строит классы и `:ops`, затем **overlay** стыкует метрики по `:namespace` из crapper/mutator — **точечно как у Java в README**, но Java-сканера в репозитории пока нет; C# — первый JVM-style язык с полным `:ns` = `Namespace.Type`.

## Что нужно в репозитории

| Что | Зачем |
|-----|--------|
| Production `.cs` | Классы на диаграмме, `:ops` = методы для клика и overlay |
| `.metrics/crap.edn` | CRAP/CC/coverage (crapper) |
| `.metrics/mutate/*.edn` | Mutation score по `:namespace` + `defn/Method` |
| `.metrics/dry.edn` | Опционально; dry не рисуется на карточке, но лежит рядом |
| Policy / `:sources` | Указать `:lang :csharp` и корень solution |

**Не нужны** для отображения: Coverlet, зелёные тесты (метрики вы уже генерируете crapper/mutator отдельно).

## Стыковка с crapper / mutator

| Поле | Значение |
|------|----------|
| `:ns` класса на IR | **`Namespace.Type`** или **`Namespace.Outer.Inner`** — как `:namespace` в `crap.edn` / `mutate/*.edn` |
| `:name` операции | имя метода (`Run`, `ValidateAsync`) |
| `defn-/Method` | private-метод (`private` в заголовке) |

Overlay (`uml-viewer.application.overlay`) сопоставляет snapshot **по точному `:ns`**, затем по id класса и dotted-потомку. Rust использует `::`; C# — только **точки**.

## Чем C# отличается от Java (в viewer)

| Тема | Java (задумка README) | C# (реализация) |
|------|------------------------|-----------------|
| Класс на диаграмме | тип в `.java` | `class` / `struct` / `record` / `interface` |
| Namespace в overlay | `package.Type` | `namespace` + nested **`Ns.Outer.Inner`** |
| Expression body | нет | методы с `=>` попадают в `:ops` |
| Зависимости | `import` | `using` → `:dependency` или **foreign** (`System.*`, …) |
| Наследование | `extends` / `implements` | `: Base, IFace` на объявлении типа |
| Обход | `target`, tests | + **`bin`**, **`obj`**, **`.vs`**, `*Tests.cs`, `*.Tests` dirs, generated |
| Парсинг | (план) | **regex + mask**, не Roslyn — см. риски |

## Policy и запуск

Пример для solution вроде GLPI:

```edn
{:title "Glpi Sync"
 :prefix "TP.Cmdb.Glpi.Sync"
 :hierarchical true
 :sources [{:lang :csharp :root "."}]}
```

Генерация IR:

```powershell
cd path\to\uml-viewer
clj -M:ir your.policy.edn your.ir.edn
```

Просмотр (Quil):

```powershell
clj -M:uml-viewer --root "D:\path\to\YourSolution" your.ir.edn
```

`:root` для viewer — каталог с `.metrics`. Метрики подхватываются при старте и после regen crapper/mutator.

Один язык в policy — `:lang :csharp` на корне вместо `:sources`.

## Риски и ограничения

1. **Regex, не компилятор** — редкие конструкции (`#if`, file-scoped + nested, строки с `class`) могут дать лишние/пропущенные `:ops`. Для CRAP overlay важнее совпадение `:ns` с crapper, чем полнота рёбер.
2. **Хелперы в test-проектах** — каталоги `*.Tests` / `*.IntegrationTests` **не сканируются** (dryer/crapper/viewer); `*Tests.cs` — по имени файла.
3. **Partial types** — несколько `partial class` с одним `:ns` **сливаются** в один класс (`:ops` объединяются).
4. **Explicit interface implementation** — `void IFace.Method()` попадает в `:ops`; редкие сигнатуры возможны.
5. **Клик по члену** — `:file` + `:lang :csharp`; source ищет объявление (в т.ч. EII).
6. **Java** в README описан, но **отдельного `java_language` нет** — для JVM-стека сейчас ориентир **C#**.

Сузьте scan через `:sources` на production-проекты; Roslyn — если regex перестанет совпадать с метриками.

## Минимальный smoke-test

```powershell
# метрики уже есть от crapper/mutator
clj -M:ir your.policy.edn out.edn
# в out.edn — :classes с :lang :csharp и :ns "Your.Namespace.Type"

clj -M:spec -e "(require 'uml-viewer.csharp-language.graph-csharp-spec)"  ; или полный clj -M:spec
```

Откройте класс с CRAP: на карточке — `--crap--`, на методах — CC/CRAP; после mutator — **M** и killed/survived.

---

[English version](csharp-project.md)
