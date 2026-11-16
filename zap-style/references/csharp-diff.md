# zap-style · C# diff discipline & surgical change (concern shard)

The heaviest, highest-signal axis. Loaded for any C# change. Score against the rubric in `../SKILL.md`.

## ⭐ LEARNING (C#) — a senior rewrote skill-authored code (highest signal)

These OUTWEIGH the plain examples. If a learning conflicts with a plain example, follow the learning.

**Extend the existing helper; do not fork a parallel one.** ⭐ `[C#] [learning]`
```csharp
// ❌ Claude added a SECOND, near-identical helper
static T? ToSerializedEnum<T>(this string? value) where T : struct, Enum =>
    Enum.TryParse<T>(value, ignoreCase: true, out var result) ? result : null;
Align = theme[PivotTable.Base.TextAlign].ToSerializedEnum<TextAlign>(),

// ✅ Simon overloaded the EXISTING ToEnumOrNull family (+ TODO to push down), kept ?.
// TODO move to SharedPackages
static T? ToEnumOrNull<T>(this string input, bool ignoreCase) where T : struct, Enum =>
    Enum.TryParse<T>(input, ignoreCase, out var result) ? result : null;
Align = theme[PivotTable.Base.TextAlign]?.ToEnumOrNull<TextAlign>(true),
```
One helper family, not two. — 1ba39ddb45e

**The "why" lives in the commit message, not a doc-comment essay.** ⭐ `[C#] [learning]`
```csharp
// ❌ Claude justified a special-case map with a prose <summary>
/// <summary>
/// The domain stores these as lower-case CSS values, which a case-sensitive enum parse rejects.
/// </summary>
// ✅ Simon KEPT the explicit hyphen->enum switch (values Enum.TryParse can't handle) but DROPPED the essay
static TextDecoration? ToSerializedTextDecoration(this string? value) => value switch
    {
        "none" => TextDecoration.None,
        "underline" => TextDecoration.Underline,
        "overline" => TextDecoration.OverLine,
        "line-through" => TextDecoration.LineThrough,
        _ => null
    };
```
Keep the behavioural code; cut the prose. — 1ba39ddb45e

**Trust the code + commit message; don't annotate every non-obvious line.** ⭐ `[C#] [learning]`
```csharp
// Simon kept Claude's behavioural fix (the aggregate-flagged `true`) but DELETED the inline comment:
case { RowHeader: MemberTuple rowHeader }:
    table.HeaderStyles[new NodeAddress(rowHeader.ToDomainTuple(domain.Query, cubeDef), true)] = styling.ToDomainTableStyles();
    break;
```
— 1ba39ddb45e

**A demo hack gets house rules too — one predicate on the existing lines, not a defensive subsystem.** ⭐ `[C#] [learning]`
```csharp
// ❌ Claude, "because it's just a hack": 3 helper predicates, a null-resolution ladder against a
// self-inflicted collision, cubeDef threaded through ToMeasure + both call sites, 20 banner-comment lines
private static bool IsDaysBased(IZapDimension dimension, ICubeDefinition cubeDef) => ...
private static bool IsNotDaysBased(string relatedDimension, ICubeDefinition cubeDef)
{
    var dimension = cubeDef.Dimensions.FirstOrDefault(d => d.UniqueName == relatedDimension)
        ?? cubeDef.Dimensions.FirstOrDefault(d => d.Name == relatedDimension);
    ...
}
private static bool MentionsDays(string? text) => ...

// ✅ the user's bar: the whole change is two predicates on existing lines + one helper
.Where(d => !d.IsHidden && !MentionsDays(d.Name))
? relatedDims.Where(d => !MentionsDays(d)).Select(refs.RefOrSlug).Distinct().ToList()
private static bool MentionsDays(string name) => name.IndexOf("days", StringComparison.OrdinalIgnoreCase) >= 0;
```
45 lines where 3 sufficed; the extra 42 defended against problems the 3-line version cannot have. — (uncommitted demo hack, 2026-07-19)

---

## Diff shape & surgical change

**A two-line guard fixes the whole bug; surrounding idiom untouched.** `[C#]`
```csharp
var value = valueFactory();
if (value is not null)
    logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty(key, value));
```
— c481ebc12db

**One `{ Count: > 0 }` guard folds null-check, empty-check and binding into a minimal edit.** `[C#]`
```csharp
// ❌ table.RowAndColumnStyles[cellAddress] = styling.ToDomainTableStyles();
// ✅
if (styling.ToDomainTableStyles() is { Count: > 0 } tableStyles)
    table.RowAndColumnStyles[cellAddress] = tableStyles;
```
— a3443764568

**A dropped argument is the entire bug — fix exactly one token, nothing more.** `[C#]`
```csharp
// ❌ : this(levelUniqueName, propertyName, caption, description)
// ✅ : this(levelUniqueName, propertyName, propertyType, caption, description)
```
— 9710891693f

**Push deps to the constructor so every call site sheds args.** `[C#]`
```csharp
// ❌ class AdomdDataManagerFactory(ILongRunningQueryLoggingService log) { IAdomdDataManager? GetAdomdDataManager(ICubeConnectionInfo connection, ICanQueryCache canQueryCache, IApplicationInsightsTelemetryClient telemetryClient); }
// ✅
public class AdomdDataManagerFactory(ICanQueryCache canQueryCache, IApplicationInsightsTelemetryClient telemetryClient, ILongRunningQueryLoggingService longRunningQueryLoggingService)
    IAdomdDataManager? GetAdomdDataManager(ICubeConnectionInfo connection);
```
…and in the SAME commit delete the now-dead static back-channel field it justified. — c37b27a06da

**Remove a threaded-but-unused parameter top-to-bottom, dissolving the wrapper it justified.** `[C#]`
```csharp
// ❌ BuildSchema(p.ParameterType, ["properties", p.Name!]);  CopilotJsonSchema BuildSchema(Type type, string[] path) { var context = new SchemaContext(path, serializerOptions.GetTypeInfo(type)); ...
// ✅ BuildSchema(p.ParameterType);                            CopilotJsonSchema BuildSchema(Type type) { var typeInfo = serializerOptions.GetTypeInfo(type); ...
```
— 7cc07aaf634

**Delete a single superfluous call — smallest possible blast radius.** `[C#]`
```csharp
response.Write("event:complete\ndata:[DONE]\n\n");
await response.FlushAsync().ConfigureAwait(false);
// response.End();  ← removed; one line, one file
```
— abc1e49dbd6

**Deleting dead code means deleting it everywhere — class, tests, AND `<Compile Include>` in both csprojs.** `[C#]`
```xml
<!-- both deleted in one commit -->
<Compile Include="Resources\Serialization\JsonConverters\TupleConverter.cs" />
<Compile Include="Resources\Serialization\JsonConverters\TupleConverterTests.cs" />
```
— bcf42ff3a5d

**Net -38 lines: inline a redundant local, rename to true intent, drop the now-unused param & usings together.** `[C#]`
```csharp
// ❌ var analysis = (Analysis)viewState.Resource; var diagnoses = ValidateQuery(analysis); var issues = diagnoses.Any() ? QueryIssues(diagnoses, analysis) : [];
// ✅ var diagnoses = DiagnoseQueryErrors((Analysis)viewState.Resource); var issues = diagnoses.Any() ? QueryIssues(diagnoses) : [];
```
— 52c89aa53ea

**Break a dependency cycle by lifting ONE tiny interface into the shared assembly — not a rewrite.** `[C#]`
```csharp
// SharedSource/Web.Core/Copilot/ICopilotChatProvider.cs (new)
public interface ICopilotChatProvider
{
    IChatClient GetClient(CopilotContext context);
}
```
Commit body names the exact cycle. — 875fb882822

**A clarity rename touches only the symbol, its refs, and the `.csproj` move — nothing else perturbed.** `[C#]`
```
SharedSource/Web.Core/Client/{Role.ts => MessageRole.ts}   // 5 files, +11/-12
```
— ba6fd6ee185

**A pure typo-fix + reformat lands in its OWN honestly-titled commit — never smuggled into a feature diff.** `[C#]`
```csharp
// "Unrelated cleanup": AssertCalculationOrderNotOverriden → AssertCalculationOrderNotOverridden
```
— 581c0700be7

**Delete an obsolete branch; the commit body explains the upstream change that made it dead.** `[C#]`
```csharp
if (hierarchy != null)
{
    // (removed: if (hierarchy.IsTimeDimension) return true;)
    // Check attribute of lowest level
```
— 0700a368239
