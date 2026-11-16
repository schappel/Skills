# zap-style · BI analytics domain idioms (domain shard)

Area-specific conventions for the analytics / query side — *consuming* deployed SSAS cubes: generating MDX, running it via ADOMD, and rendering CellSets/pivots. Lives under `BI/Resources/Query`, `BI/Resources/Olap`, `BI/Resources/CellSets`, `BI/Presentation`. Load this in addition to the C# concern shards (`csharp-core.md`, `csharp-diff.md`) for any change in the analytics/query code. Score against the rubric in `../SKILL.md`.

**Modeller vs BI:** Modeller *writes* the cube (authoring, warehouse, ETL — see `modeller.md`); BI *reads* it. If the change generates/executes MDX or renders query results, it's BI.

**Sources:** some entries below cite a commit sha; others cite a `convention: <file>` — a repo-wide pattern with no single originating commit. Both are house style.

**Contents:** Building MDX through the DOM · Dimensionality & UniqueName identity · Special cases · Parsing & name collisions · Cell/result pipeline

---

## Building MDX through the DOM

**Build MDX through the strongly-typed DOM, never by string concatenation.** `[C#]` [domain:bi]
```csharp
// ❌ "Iif(" + cond + "," + a + "," + b + ")"
// ✅
return new MdxFunction(FunctionName.IIf)
    .Parameter(condition).Parameter(trueValue).Parameter(falseValue);
```
All MDX is an `IMdxObject` tree, so generators compose rather than concatenate. — convention: `MdxDom.cs` (repo-wide, `BI/Resources/Query/DomGenerators/**`)

**Use the `FunctionName` enum for MDX functions, not literal strings.** `[C#]` [domain:bi]
```csharp
new MdxFunction(FunctionName.CurrentMember)
new MdxFunction(FunctionName.Members)
new MdxFunction(FunctionName.Extract, parameters)
```
Function identity is type-checked; raw operators (`"AND"`, `"+"`) only appear inside `Operation`. — convention: `DomGeneratorHelper.cs`

**Compose MDX via fluent extension verbs on `IMdxObject`, not ad-hoc construction.** `[C#]` [domain:bi]
```csharp
mdxObjects.AggregateWithCrossJoin();   // (a * b * c) wrapped in Parenthesis
parts.AggregateWithConcat();           // builds an MdxSet
hierarchy.BuildCurrentMember();        // [Dim].CurrentMember
```
Domain verbs (CrossJoin/Union/Aggregate/CurrentMember) read like MDX. — convention: `DomGeneratorHelper.cs`

---

## Dimensionality & UniqueName identity

**Model dimensionality as a value object keyed on hierarchy UniqueNames, with value equality.** `[C#]` [domain:bi]
```csharp
public class Dimensionality {
    public IEnumerable<string> HierarchyUniqueNames => _hierarchies;
    public bool ContainsMeasures => _hierarchies.Contains(SsasConstants.MeasuresUniqueName);
    public bool IntersectsWith(Dimensionality other) => _hierarchies.Intersect(other._hierarchies).Any();
    // Equals = SequenceEqual on hierarchies; GetHashCode via Jenkins hash
}
```
Axis grouping / cross-join logic depends on dimensionality equality — make it a first-class type. — convention: `Dimensionality.cs`

**Reference hierarchies/members by UniqueName, never by caption.** `[C#]` [domain:bi]
```csharp
member.Hierarchy == SsasConstants.MeasuresUniqueName
Query.Axes.IndexOf(a => a.Dimensionality.Contains(SsasConstants.MeasuresUniqueName));
```
Captions are localized/ambiguous; UniqueName is the identity. `Caption()` is display-only. — convention: `Dimensionality.cs`, `CellSetEvaluator.cs`

---

## Special cases

**Treat the Measures hierarchy as the special-case fallback axis explicitly.** `[C#]` [domain:bi]
```csharp
if (query.Axes[axisIndex].Sets.Dimensionality.None(h => h == SsasConstants.MeasuresUniqueName))
    outerHierarchy = SsasConstants.MeasuresUniqueName; // empty axis -> use measures
```
Measures behaves unlike other hierarchies; encode that, don't let it fall through generic paths. — convention: `CellSetEvaluator.cs`

---

## Parsing & name collisions

**Parse MDX unique names with the dedicated `UniqueNameParser`, don't hand-split on brackets.** `[C#]` [domain:bi]
```csharp
// brackets, &keys, and . separators have MDX semantics
Member member = new UniqueNameParser().Parse(hierarchy.UniqueName);
```
`[Dim].[Hier].&[key]` is a grammar, not a string — round-trip it through the parser. — convention: `UniqueNameParser.cs`, `HierarchyDomGenerator.cs`

**Alias the MDX `Tuple`/`Member` types to disambiguate from `System.Tuple` and `QueryParts`.** `[C#]` [domain:bi]
```csharp
using MdxTuple = Zap.Mdx.MdxClass.Tuple;
using SetList = Zap.Query.QueryParts.SetList;
```
The domain has many collision-prone names (Tuple, Member, SetList); alias at the top of generator files. — convention: `SetListJoiner.cs`, `HierarchyDomGenerator.cs`

---

## Cell / result pipeline

**Carry SSAS member metadata (FormatString) explicitly through the cell pipeline.** `[C#]` [domain:bi]
```csharp
new Dictionary<string, string> { [SsasConstants.FormatStringProperty] = $@"""{v}"";""{v}""" }
```
Cells are re-formatted only when a FormatString is present; format strings are SSAS metadata, not arbitrary props. — 347c6a5787f
