# zap-style · C# core idioms (concern shard)

Cross-cutting C# house idioms. Loaded for any C# change. Score against the rubric in `../SKILL.md`; the highest-signal ⭐ learnings live in `csharp-diff.md`, with review-driven corrections below.

**Contents:** ⭐ Learning-proxy (review corrections) · Nullability & null-flow · Pattern matching & switch dispatch · Fail-loudly & exceptions · Types & immutability (records) · Behaviour: LINQ, laziness, intent-named methods

## ⭐ LEARNING-PROXY (C#) — corrections a reviewer demanded in PR review

Next-highest signal after the ⭐ learnings in `csharp-diff.md`: these are changes made *because a reviewer asked*, so they're strong evidence of house preference. If one conflicts with a plain example below, follow this.

**Prefer a positive type pattern over a typed null-existence pattern.** `[C#] [learning-proxy]`
```csharp
// ❌
DateTimeDataType { TimeZoneId: { } tz } => $"{DateTimeTypeName} [{tz}]",
// ✅
DateTimeDataType { TimeZoneId: string tz } => $"{DateTimeTypeName} [{tz}]",
```
Name the type you expect (`string tz`), not the cryptic non-null pattern `{ } tz`. — 0ff97d3

**Collapse count-branching guard clauses into a `switch` expression.** `[C#] [learning-proxy]`
```csharp
// ❌
if (configs.None()) throw new NotFoundException("...does not exist...");
if (configs.Length > 1) throw new NotFoundException("...Multiple...rename them.");
return configs.First();
// ✅
return configs.Length switch {
    0 => throw new NotFoundException("...does not exist..."),
    1 => configs.Single(),
    _ => throw new NotFoundException("...Multiple...rename them.")
};
```
One exhaustive expression beats three sequential statements; `Single()` asserts the invariant `First()` hid. — f3973f0

**Replace null-check-then-throw with the `??` throw expression.** `[C#] [learning-proxy]`
```csharp
// ❌
Resource? resource = service.GetResourceForWrite(id, user);
if (resource == null) throw new SessionExpiredException();
// ✅
Resource resource = service.GetResourceForWrite(id, user) ?? throw new SessionExpiredException();
```
Drops the nullable local and the branch; the variable lands non-null. — 9702c83

**Fix the hidden bug, don't just tidy it — equality must check both directions.** `[C#] [learning-proxy]`
```csharp
// ❌  passes when settings1 ⊆ settings2 but counts differ
return settings1.Settings.All(s => AreEqual(s, settings2.Settings.FirstOrDefault(t => t.Key == s.Key)));
// ✅
return settings1.Settings.Count == settings2.Settings.Count
    && settings1.Settings.All(s => AreEqual(s, settings2.Settings.FirstOrDefault(t => t.Key == s.Key)));
```
Guard the count first. (A reviewer *adding* a line — a genuine correctness fix, still surgical.) — dccbe7d

**Use `is not (A or B)` instead of negating an `is (A or B)`; drop bespoke `IfIs<T>` helpers.** `[C#] [learning-proxy]`
```csharp
// ❌
qp => !(qp is ZapNamedSetInstance or ZapCalculatedMemberInstance),
// ✅
qp => qp is not (ZapNamedSetInstance or ZapCalculatedMemberInstance),
```
Also `qp is X || qp is Y` → `qp is X or Y`, and `IfIs<T>(x => …)` → `qp is T t && …`. Lean on built-in pattern syntax. — 1854223, a526a0f

**In a method group conversion, pass the group, don't wrap it in a lambda.** `[C#] [learning-proxy]`
```csharp
// ❌
runCache.GetOrAdd(packageId, (id, ct) => GetLatestWithRetries(id, ct))
// ✅
runCache.GetOrAdd(packageId, GetLatestWithRetries)
```
When params line up, the method group *is* the delegate. (`action.Invoke(x)` → `action(x)` too.) — d8bdfec

**Keep filters deferred; materialize read-only results with `.ToArray()`, not `.ToList()`.** `[C#] [learning-proxy]`
```csharp
// ❌
IEnumerable<Tenant> Filter(IEnumerable<Tenant> tenants) => tenants.Where(CanAccess).ToList();
return Filter(tenants).Select(ToClientModel).ToList();
// ✅
IEnumerable<Tenant> Filter(IEnumerable<Tenant> tenants) => tenants.Where(CanAccess);
return Filter(tenants).Select(ToClientModel).ToArray();
```
Materialize once, at the boundary that owns the snapshot. Keep `.ToList()` only when mutation or a list-specific contract requires it. — 98baa78a0, ffbc75d6c

**Use the established Zap helper family instead of spelling the same operation with the BCL.** `[C#] [learning-proxy]`
```csharp
// ❌
if (!string.IsNullOrEmpty(publicBaseUrl)) { ... }
return string.Join(" ", content.Select(c => c.Text));
// ✅
if (!publicBaseUrl.IsNullOrEmpty()) { ... }
return content.Join(" ", c => c.Text);
```
Search `Zap.Text`, `Zap.Collections`, `Zap.Reflection`, and nearby code before introducing a raw equivalent. — 98baa78a0, ffbc75d6c

**Await independent service calls together with tuple `.WhenAll()`.** `[C#] [learning-proxy]`
```csharp
// ❌
var tenantsTask = tenantService.IndexAsync();
var hostsTask = hostService.IndexAsync();
var tenants = await tenantsTask;
var hosts = await hostsTask;
// ✅
var (tenants, hosts) = await (tenantService.IndexAsync(), hostService.IndexAsync()).WhenAll();
```
The tuple keeps the concurrent intent in one expression and observes both tasks if either fails. — 6c99d8dbf, ffbc75d6c

**Materialize EF queries with `.AsEnumerable()`, not `.ToArray()`, before a client-side `.Select`.** `[C#] [learning-proxy]`
```csharp
// ❌
context.TeamsAiNotifications.Where(n => n.UserId == userId).ToArray().Select(MapToEntity).ToArray();
// ✅
context.TeamsAiNotifications.Where(n => n.UserId == userId).AsEnumerable().Select(MapToEntity).ToArray();
```
`AsEnumerable()` switches to LINQ-to-Objects without an extra allocation; the trailing `ToArray()` does the one materialization. — 3e0dfbb

---

## Nullability & null-flow

**`?.Field is Constant` treats missing and non-matching identically — null never matches.** `[C#]`
```csharp
if (templateInstance.Fields.Get(keyValuePair.Key)?.FieldType is FieldType.ReferenceSetComponent && queryPart.IsSystemTemplate(SystemResources.NullMember))
    templateInstance.ClearInstanceValue(keyValuePair.Key);
```
— 80944704429

**`Enum.TryParse(out _) ? x : null` → null-propagating `?.ToEnumOrNull<T>()`, then `??` fallback.** `[C#]`
```csharp
// ❌
Position = Enum.TryParse(theme[Chart.LegendPlacement], out LegendPlacement placement) ? placement : LegendPlacement.None,
// ✅
Position = theme[Chart.LegendPlacement]?.ToEnumOrNull<LegendPlacement>() ?? LegendPlacement.None,
```
Applied uniformly across every site. — 53d58ae17be

**Build-or-null, then drop the gaps with `.NotNull()` — no per-field branching downstream.** `[C#]`
```csharp
.Select(member =>
    {
        var uniqueName = member.Element("UniqueName");
        var calculatedMemberId = member.Element("CalculatedMember")?.AttributeValue("Id");
        return uniqueName != null && calculatedMemberId != null
            ? new CalculatedMemberInstanceReference(uniqueName, sourceQueryPartId, calculatedMemberId)
            : null;
    })
.NotNull();
```
— 87c732caaeb

**`[NotNullWhen(true)]` lets callers skip the re-check / `!` after the guard.** `[C#]`
```csharp
// ❌ EvaluationResult result (non-null) forces every caller to null-check before calling
// ✅ nullable + [NotNullWhen(true)]: a true return proves the value non-null downstream
bool ShouldNotify(AINotificationConfig config, [NotNullWhen(true)] EvaluationResult? result)
{
    return result?.ConditionMet == true && (config.ConditionType == AINotificationConditionType.State || ...);
}
```
— d01c921777f

**Combined relational/constant pattern folds the nullable in: `is not null and not X and < Y`.** `[C#]`
```csharp
// ❌  return calculationOrder != (decimal)PresetCalculationOrder.None && calculationOrder < (decimal)PresetCalculationOrder.Aggregation;
// ✅
return calculationOrder is not null
    and not (uint)PresetCalculationOrder.None
    and < (uint)PresetCalculationOrder.Aggregation;
```
— 660c9374a7c

---

## Pattern matching & switch dispatch

**Validate-and-narrow in ONE switch: null, negation, nested property, declaration arm.** `[C#]`
```csharp
return resourceService.GetResourceForRead(resourceId, user) switch
    {
        null => throw new InvalidAnalysisException(InvalidAnalysisReason.NotFound, "..."),
        not Analysis => throw new InvalidAnalysisException(InvalidAnalysisReason.NotAnAnalysis, "..."),
        Analysis { Properties.CopilotEnabled: false } => throw new InvalidAnalysisException(InvalidAnalysisReason.NotCopilotEnabled, "..."),
        Analysis analysis => analysis
    };
```
— 5563ac7c640

**Type-dispatch switch returning collection-expression results, ending in `_ => throw`.** `[C#]`
```csharp
private static QueryPart[] FieldValue(this object fieldValue, QuerySerializationOptions options) => fieldValue switch
    {
        string value => [new QueryPart { Text = value }],
        int value => [new QueryPart { Number = value }],
        bool value => [new QueryPart { Boolean = value }],
        IQueryPart queryPart => [queryPart.ToSerializedQueryPart(options)],
        _ => throw new ArgumentOutOfRangeException(nameof(fieldValue), GetTypeNameSafe(fieldValue), "Unexpected field value type")
    };
```
— 9225dbc987c

**Each switch arm resolves-or-throws (`?? throw`, is-pattern + ternary throw, `_ => throw`).** `[C#]`
```csharp
return connectionIdentifier switch
    {
        { AnalysisViewStateKey: string viewStateKey } =>
            analysisTools.GetCellSetResourceViewState(context, viewStateKey).Resource.Connection
            ?? throw new InvalidToolUsageException("Current analysis doesn't have a connection configured."),
        { SpecificConnection: ConnectionAddress address } =>
            serverConnectionResolver.GetServerConnectionInfo(address.ModelServerResourceId) is IServerConnectionInfo serverConnection
                ? new CubeConnectionInfo(serverConnection, address.Database, address.Database)
                : throw new InvalidToolUsageException("Could not find a valid model server for the provided ID."),
        _ => throw new InvalidToolUsageException($"Invalid connection identifier...")
    };
```
— 4d2ac1ac983

**`is T x && … is { Prop: true }` chains replace `IfIs<T>(lambda)` — flatter, no closures.** `[C#]`
```csharp
// ❌  return queryPart.IfIs<Measure>(m => { var mi = CubeDef.GetMeasure(m.UniqueName); return mi != null && mi.IsCalculated; }) || queryPart.IfIs<NumericTemplateInstance>(...) ;
// ✅
return queryPart is Measure measure && CubeDef.GetMeasure(measure.UniqueName) is { IsCalculated: true }
    || queryPart is NumericTemplateInstance numeric && IsPreAggregate(numeric.ResultingCalculationOrder);
```
— 660c9374a7c

**Group enum cases with `or`, then match the payload with list+property patterns.** `[C#]`
```csharp
FieldType.None or FieldType.UserInput or FieldType.CheckBox or FieldType.DropDown or FieldType.MdxExpression =>
    serializedFieldValues switch
        {
            [{ Text: string text }] => text,
            [{ Number: decimal number }] => (double)number,
            _ => throw new ArgumentOutOfRangeException(nameof(serializedFieldValues), serializedFieldValues, "Unexpected field value structure for single-value field.")
        },
```
— 82bcef4baa5

**`is X or Y` as an inline boolean test, paired with `is not X`.** `[C#]`
```csharp
AllowsMultiSelect = g is CheckBoxSliceGroup or CheckBoxDropDownTreeSliceGroup or DateRangeSliceGroup,
CanBePutOnRows = g is not TimeContextSliceGroup && !analysisResource.Query.Axes.Dimensionality().IntersectsWith(g.DimensionalityUniqueNames),
```
— 005b551ebf9

**Declaration pattern + a downstream type pattern as a `.Any()` membership test.** `[C#]`
```csharp
return node.NodeValue.Address.Tuple.Members
    .Concat(otherAxisNode.NodeValue.Address.Tuple.Members)
    .Any(m => m is QueryMember qm && ResourceInstance.Query.FindQueryPart(qm.SourceQueryPartId) is MemberProperty);
```
— a088ead0125

**Positional `is (T x, U y)` unpacks a nullable tuple and null-checks it in one step.** `[C#]`
```csharp
// ❌  var rt = UnwrapReturnType(method.ReturnType); if (rt == null) return null; var schema = BuildSchema(rt, IsNullableParameter(method.ReturnParameter!)); return ToJsonElement(schema);
// ✅
if (UnwrapReturnType(method.ReturnParameter!) is (Type returnType, NullabilityInfo nullability))
{
    var schema = BuildSchema(returnType, nullability);
    return ToJsonElement(schema);
}
return null;
```
— 29b3250d3ab

**Special-case first via ternary, then fall through a `??` chain — reads as a priority list.** `[C#]`
```csharp
return PartialTupleTreeNodeInfo.IsBlankMember(uniqueName) ? PartialTupleTreeNodeInfo.CreateBlankNodeMember(uniqueName)
    : QueryMember.ConvertResultMemberIfPossible(uniqueName, query, cubeDef.RealCubeDefinition)
    ?? cubeDef.GetMember(uniqueName);
```
— fa4a50ac47b

**When the type set is open/extensible, dispatch dynamically (open-generic + `MakeGenericType` + `IsRegistered` guard + `(dynamic)`) rather than a closed hand-written switch.** `[C#]`
```csharp
public interface ISolutionInfoCleaner<in T> where T : Resource { Task<bool> Clean(T resource, CancellationToken cancellationToken); }

public async Task<bool> Clean(Resource resource, CancellationToken cancellationToken)
{
    var cleanerType = typeof(ISolutionInfoCleaner<>).MakeGenericType(resource.GetType());
    if (container.IsRegistered(cleanerType))
        return await ((dynamic)container.Resolve(cleanerType)).Clean((dynamic)resource, cancellationToken).ConfigureAwait(false);
    return false;
}
```
The escape hatch when a closed switch would have to grow forever. — ac54528b1a2

**Tri-state dispatch: cast the whole switch to the nullable base so supported arms return subtypes, an `or`-grouped block of known-unsupported cases collapses to `null`, and `_ => throw` still catches the genuinely new.** `[C#]`
```csharp
var node = (DimensionTreeNodeSpecifier?)(request.NodeType switch
    {
        NodeType.Cube => new CubeNodeSpecifier(),
        NodeType.Dimension => new DimensionNodeSpecifier(request.Identifier),
        NodeType.Level => new LevelNodeSpecifier(request.Identifier, 0), // Level number isn't important here
        NodeType.Root or NodeType.Search or NodeType.NamedSet or NodeType.MemberProperty or NodeType.TimeFunction => null,
        _ => throw new ArgumentOutOfRangeException(nameof(request.NodeType), $"Unexpected node type: {request.NodeType}")
    });
```
— 325f3590dca

---

## Fail-loudly & exceptions

**`?? throw` inside a `Select` via a shared expression-bodied factory.** `[C#]`
```csharp
var memberChain = new MemberChain(expandedPath.Select(un => cubeDef.GetMember(un) ?? throw MissingMemberException(un)));
static InvalidToolUsageException MissingMemberException(string un) => new($"Member {un} doesn't exist in the model.");
```
— 005b551ebf9

**`as X ?? throw` asserts each projected element mid-pipeline rather than failing later.** `[C#]`
```csharp
.Select(v => v as ISetComponent ?? throw new ArgumentOutOfRangeException(nameof(serializedFieldValues), v, "Tuple field value must be a set component."))
```
— be9b626eac1

**Trust-boundary filter: rethrow safe typed exceptions, swallow the rest behind a generic one.** `[C#]`
```csharp
static async ValueTask<object?> InvokeWithExceptionFiltering(FunctionInvocationContext context, CancellationToken cancellationToken)
{
    try { return await context.Function.InvokeAsync(context.Arguments, cancellationToken).ConfigureAwait(false); }
    catch (InvalidToolUsageException) { throw; }
    catch (LocalizedException) { throw; }
    catch { throw new InvalidToolUsageException("Unknown"); }
}
```
— 42e807a3429

**Most-specific catch first; map an expected error to a domain result or rethrow.** `[C#]`
```csharp
catch (CurrentPeriodException e)
{
    return new QueryPartDiagnosis(responsibleQueryPart, e.LogMessage, e.AdvancedErrorMessage, e.ErrorCode, e.SimpleErrorMessage, e.HelpLink); // Always an expected error
}
catch (OlapException e)
{
    if (CheckForExpectedErrors(typedTemplateInstance, e) is QueryPartDiagnosis expectedError)
        return expectedError;
    throw;
}
```
— 33f1d98e8a5

**For AI-consumed messages, surface raw `e.Message` — localization is the wrong layer.** `[C#]`
```csharp
// ❌ Value = e.GetLocalizedMessage().NullIfEmpty() ?? PageText.Message_UnknownError
// ✅ Value = e.Message
```
— 48977c5089b

---

## Types & immutability (records)

**Mutable class with setters → abstract record with `init`-only props.** `[C#]`
```csharp
// ❌ public abstract class Schema { public bool HasDefaultValue { get; set; } protected bool IsNullable { get; private set; } }
// ✅ public abstract record Schema { public bool HasDefaultValue { get; init; } public bool IsNullable { get; init; } }
```
— f94a5864e24

**Self-mutating method → `with` expression returning a fresh record.** `[C#]`
```csharp
// ❌ public Schema MakeNullable() { IsNullable = true; return this; }
// ✅ public Schema AsNullable() => this with { IsNullable = true };
```
— f94a5864e24

**Empty-bodied marker interfaces + one-line positional records = a tidy closed hierarchy.** `[C#]`
```csharp
public interface IQueryPart;
public interface IFieldValue;
public interface ISet : IQueryPart, IFieldValue;

public record MeasureGroup(string Mdx, string Caption) : ISet;
public record Measure(string Mdx, string Caption) : ISet, IFormulaComponent;
```
— b0a26993fe6

**Abstract record base + empty derived records models a discriminated union of singletons.** `[C#]`
```csharp
public abstract record Operator : IFormulaComponent;
public record And : Operator;
public record Else : Operator;
```
— b0a26993fe6

**Small immutable value bundle → `readonly record struct` with positional params.** `[C#]`
```csharp
readonly record struct SchemaContext(string[] Path, JsonTypeInfo TypeInfo);
```
— 58a77a08cc3

**`required init` for mandatory fields; expression-bodied derived props off the raw ones.** `[C#]`
```csharp
public required JsonPatchOp Op { get; init; }
public required string Path { get; init; }
public string[] PathSegments => JsonPatchHelpers.ParseJsonPointer(Path);
public string[]? FromSegments => From is null ? null : JsonPatchHelpers.ParseJsonPointer(From);
```
— 5ee120088fc

**Serialization records: never null — default every collection `[]`, every child `new()`.** `[C#]`
```csharp
public RichText Title { get; init; } = new();
public SortedMember[] SortedColumns { get; init; } = [];
public HashSet<AxisSummary> ColumnSummaries { get; init; } = [];
```
The `HashSet` here is superseded by later practice: for serialization models prefer `SortedSet`/`SortedDictionary` (see fc8460aa42f below) — this example is about the never-null defaults. — b0a26993fe6

**Target-typed `new()` drops the repeated generic type name.** `[C#]`
```csharp
// ❌ public SerializableUnknownTypeList<IZapMember> SerializableMembers { get; set; } = new SerializableUnknownTypeList<IZapMember>();
// ✅ public SerializableUnknownTypeList<IZapMember> SerializableMembers { get; set; } = new();
```
— 1c0d8b25822

**Domain↔serialization mapping lives in a `static` converter as `ToSerializedX` extensions.** `[C#]`
```csharp
public static ResourceReference ToSerializedReference(this IResourcePointer reference) => new()
    {
        ResourceId = reference.Id,
        Name = reference.Name.DefaultCultureValue
    };
```
— b0a26993fe6

**Generic `with` on a constrained `T` layers props while preserving the concrete type.** `[C#]`
```csharp
public static T WithSerializedOlapResourceProperties<T>(this T serializedOlapResource, Zap.Query.OlapResource domain) where T : OlapResource =>
    (serializedOlapResource with { Connection = new CubeConnection() }).WithSerializedResourceProperties(domain);
```
— b0a26993fe6

**Record wrapping a list overrides `Equals` to compare by sequence.** `[C#]`
```csharp
public virtual bool Equals(JsonArrayValue<T>? other) =>
    other is not null && other.GetType() == GetType() && other.SequenceEqual(this);
```
— b0a26993fe6

**Model the "one of two" input as a record of nullable props; the switch enforces the choice.** `[C#]`
```csharp
public record AnalysisIdentifier
{
    public Guid? ResourceId { get; init; }
    public string? ViewStateKey { get; init; }
}

CellSetResource ResolveAnalysisResource(CopilotContext context, AnalysisIdentifier analysis) =>
    analysis switch
        {
            { ViewStateKey: string viewStateKey } => GetCellSetResourceViewState(context, viewStateKey).Resource,
            { ResourceId: Guid resourceId } => GetCopilotEnabledAnalysis(resourceId, context.User),
            _ => throw new InvalidToolUsageException("No analysis specified. ...")
        };
```
— 005b551ebf9

**Pick `SortedSet`/`SortedDictionary` for serialization models — deterministic, diff- & patch-safe.** `[C#]`
```csharp
// ❌ public HashSet<string> SignFlips { get; init; } = [];
// ✅ public SortedSet<string> SignFlips { get; init; } = [];   // better diffs, and avoid AI patching the wrong indexes
```
— fc8460aa42f

**`bool?` set to `true` or `null` (never `false`) so the negative serialises away.** `[C#]`
```csharp
Selected = checkBoxSliceGroup.IsSelected(new ZapTuple(m)) ? true : null,
Children = []
```
— 005b551ebf9

**A `[JsonIgnore]` get/init facade reads whichever backing store is populated and fans out on init.** `[C#]`
```csharp
[JsonIgnore]
public IReadOnlyDictionary<string, string> Cultures
{
    get => CopilotCultures.Any() ? CopilotCultures : PersistedCultures.ToDictionary(c => c.Key, c => c.Value.GetString() ?? string.Empty);
    init
    {
        CopilotCultures = value.ToDictionary();
        PersistedCultures = value.ToDictionary(c => c.Key, c => JsonElement.Parse($"\"{c.Value}\""));
    }
}
```
— 84bff3a299f

---

## Behaviour: LINQ, laziness, intent-named methods

**Defer expensive work with `Lazy<T>` so branches share it without forcing it.** `[C#]`
```csharp
var modifiedResource = new Lazy<Analysis>(() =>
    {
        var clone = (Analysis)originalResource.Clone(); // Don't want to modify the actual resource if there are issues with the proposed design
        analysisConverter.ApplyAnalysisDomainProperties(clone, modifiedModel);
        return clone;
    });

if (DiagnoseQueryErrors(originalResource).None()
    && DiagnoseQueryErrors(modifiedResource.Value) is { Length: > 0 } newQueryDiagnoses)
{
    throw new InvalidToolUsageException($"...{JsonSerializer.Serialize(QueryIssues(newQueryDiagnoses))}");
}
```
Collapses a nested `.None()`/`.Any()` guard into one `&&` with an `is { Length: > 0 }` capture. — 52c89aa53ea

**Duplicated apply-logic → one intent-named `ApplyChangesToDomain` shared by both call sites.** `[C#]`
```csharp
// ❌ the same apply+populate sequence inlined twice (once live, once on the clone)
analysisConverter.ApplyAnalysisDomainProperties(analysisResource, modified);
if (analysisResource.Connection?.CanQuery() == true)
    analysisResource.PopulateDimensionality(analysisResource.Connection, new DimensionalityCalculationService());
// ✅ named once, called from both (the clone path stays behind the Lazy<T> above)
void ApplyChangesToDomain(Analysis domain, SerializationModel.Analysis modified)
{
    analysisConverter.ApplyAnalysisDomainProperties(domain, modified);
    if (domain.Connection?.CanQuery() == true)
        domain.PopulateDimensionality(domain.Connection, new DimensionalityCalculationService());
}
```
— 04725ddc88a

**Make per-instance expensive work `Lazy<T>` fields, not eager ctor work.** `[C#]`
```csharp
readonly Lazy<JsonElement> _inputSchema = new(() => new CopilotAIFunctionSchemaGenerator(serializerOptions, shouldExcludeParameterFromSchema).CreateInputSchema(method));
public override JsonElement JsonSchema => _inputSchema.Value;
```
— 59221bbd4b8

**Hoist a compound condition into named predicates so the `if` reads as a sentence.** `[C#]`
```csharp
// ❌ if (config.Schedule.IsNotEmpty() && IsDueToEvaluate(config) && config.ConditionType switch { ... })
// ✅
if (ShouldRetryAfterFailure(config)
    || ((IsDueToEvaluate(config) || config.Schedule.IsNullOrEmpty())
        && ShouldRunQueryForConditionType(config, resource)))
```
— d01c921777f

**`.Intersect(other)` states the set intent — replaces `.Where(other.Contains)`.** `[C#]`
```csharp
// ❌ suppressedHierarchyUniqueNames.Where(axis.Dimensionality.Contains).Select(cubeDef.GetHierarchy)
// ✅ suppressedHierarchyUniqueNames.Intersect(axis.Dimensionality).Select(cubeDef.GetHierarchy)
```
The eager `ToHashSet` is dropped too, now that membership isn't probed. — 82bcef4baa5

**`.Init([...])` says "the contents are exactly this" — replaces Clear()+Add()/AddRange().** `[C#]`
```csharp
// ❌ Sets.Clear(); Sets.AddRange([Location.CountryLevel, new SetComponentContainerSet(Measures.GrossAmtMeasure)]);
// ✅ Sets.Init([Location.CountryLevel, new SetComponentContainerSet(Measures.GrossAmtMeasure)]);
```
— ef472a90479

**Detect duplicates with `GroupBy(...).Any(g => !g.IsSingular())`; walk a tree with `.PreOrder().Where(...)`.** `[C#]`
```csharp
if (currentIdNodes.StringValues().GroupBy(id => id, StringComparer.OrdinalIgnoreCase).Any(g => !g.IsSingular()))
    throw new InvalidToolUsageException("Patch cannot be applied because it would result in duplicate object IDs.");
var referenceNodes = root.PreOrder().Where(n => typeof(TypedGuid).IsAssignableFrom(n.Type));
```
— 904a3c3710f

**Build a set inline; pass the comparer straight into `ToHashSet`.** `[C#]`
```csharp
var originalIds = IdNodes(json.GetTypedJson<TModel>()).StringValues().ToHashSet(StringComparer.OrdinalIgnoreCase);
```
— 904a3c3710f

**Intent-revealing rename + `IReadOnlyCollection<string>` input; `??=` keeps the first fallback.** `[C#]`
```csharp
public static IZapMeasure GetDefaultRelatedMeasure(this ICubeDefinition cubeDef, IReadOnlyCollection<string> dimensionality)
{
    var allRelatedMeasures = cubeDef.GetMeasureGroupRelationships()
        .Where(kv => kv.Value.Intersect(cubeDef.GetDimensionNamesForHierarchies(dimensionality)).Any())
        .SelectMany(mg => cubeDef.GetMeasureGroupMeasures(mg.Key));
    IZapMeasure fallbackMeasure = null;
    foreach (var measure in allRelatedMeasures)
    {
        // Prefer to avoid calculated measures ... and First/Last child aggregations, which may require a time dimension
        if (measure.IsCalculated || measure.AggregatorType is MeasureAggregator.FirstChild or MeasureAggregator.LastChild)
            fallbackMeasure ??= measure;
        else
            return measure;
    }
    return fallbackMeasure;
}
```
— 80e46424b0c

**Stream the normal parts, then a guarded `!= null && != X && != Y` ladder appends an error.** `[C#]`
```csharp
foreach (var part in update.Contents.Select(c => ResponsePart(c, context)).NotNull())
    yield return part;

if (update.FinishReason != null && update.FinishReason != ChatFinishReason.ToolCalls && update.FinishReason != ChatFinishReason.Stop)
{
    logger.Warning("Chat finished with unexpected reason {FinishReason}", update.FinishReason);
    yield return new CopilotResponsePart { Type = ResponsePartType.Error, Value = CopilotText.Error_UnexpectedFinishReason };
}
```
— 5c71e934235

**Fetch N+1, return N, derive `HasMore` from the overflow — one pass, no second count query.** `[C#]`
```csharp
var results = resourceService.SearchAllResourcesOfType<ExpressionTemplate>(...).Take(MaxFunctionResults + 1).Select(t => new FunctionInfo { ... }).ToArray();
return new FunctionSearchResults
    {
        Functions = results.Take(MaxFunctionResults).ToArray(),
        HasMore = results.Length > MaxFunctionResults
    };
```
— e589beb4e5f

**Thread context as an explicit parameter — delete the ambient `*ContextProvider` indirection.** `[C#]`
```csharp
// ❌ yield return AIFunctionFactory.Create(GetOpenAnalysisResults);  // builder: .UseContext(...) sets AmbientCopilotToolContextProvider.SetCurrent
//    string GetOpenAnalysisResults(string viewStateKey) { var vs = toolContext.Current.GetViewState(viewStateKey) ...; }
// ✅
yield return context.CreateAIFunction(GetOpenAnalysisResults);
string GetOpenAnalysisResults(string viewStateKey, CopilotContext context) { var vs = context.GetViewState(viewStateKey) ...; }
```
— abce44b2120

**Clone behind an `is not null` guard before mutating a shared object; terse why-comment.** `[C#]`
```csharp
if (explore is not null)
    resource = (CellSetResource)resource.Clone(); // Don't modify cached copy
if (explore?.SlicerSelections is { Count: > 0 })
    ApplySlicerSelections(resource, explore.SlicerSelections);
```
— 0b43a3f828c

**Express a capability hierarchy as a ladder of getters, each ANDing onto the level below — stated once, not re-checked per level.** `[C#]`
```csharp
public bool CanUseCopilot => CoreSwitch.CopilotChat.Enabled
    && (isInCloud.Value || CoreSwitch.CopilotChatOnPrem.Enabled)
    && licensedFeatures.IsLicensed(LicenseFeatures.FullCopilotAccess);
public bool CanAccessData => CanUseCopilot && allowCopilotToAccessData;
public bool CanDesign => CanAccessData && licensedFeatures.IsLicensed(LicenseFeatures.CopilotDesign);
public bool HasFullAccess => CanDesign && licensedFeatures.IsLicensed(LicenseFeatures.FullCopilotAccess);
```
— 38085ea7b36
