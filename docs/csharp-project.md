# C# / .NET project layout for uml-viewer

Short checklist for **diagram scan**, **`.metrics` overlay**, and policy setup.

The viewer scans `.cs` with regex (like TypeScript/Python), builds classes and `:ops`, then matches **crapper/mutator** snapshots on `:namespace` (`Namespace.Type`).

## Repository

- Production `.cs` sources
- `.metrics/crap.edn`, `.metrics/mutate/*.edn` from crapper/mutator
- Policy entry `{:lang :csharp :root "..."}`

Coverlet and passing tests are not required for display — only for generating metrics.

## Overlay keys

- Class `:ns` = crapper `:namespace`
- Op `:name` = method name; `defn-/` = private

## C# vs Java

C# adds file-scoped namespaces, expression-bodied members, `using` edges, and skips `bin`/`obj`/`.vs`. Parsing is regex-based, not Roslyn.

## Risks

Regex drift; test trees under `*.Tests` folders are skipped; partial classes merge by `:ns`; explicit interface members use a second line pattern. Roslyn only if metrics drift persists.

## Running

```powershell
clj -M:ir policy.edn diagram.edn
clj -M:uml-viewer --root path\to\solution diagram.edn
```

---

[Russian version](csharp-project.ru.md)
