# zap-style · Modeller domain idioms (domain shard)

Area-specific conventions for the data-prep / ETL backend under `Modeller/**` — pipelines, steps, updaters, transforms, jobs, loaders, warehouse, state objects, DI. Distilled from Robert Macdonald's commits. Load this in addition to the C# concern shards (`csharp-core.md`, `csharp-diff.md`) for any change under `Modeller/`. Score against the rubric in `../SKILL.md`.

**Contents:** Match over closed unions (the signature idiom) · Pipeline DI, hydraters & factories · Domain types & records · Column updates & batching · SQL / Q-graph construction · Surgical edits & boundaries

---

## Match over closed unions — the signature modeller idiom

The dominant pattern. A closed type family (steps, loaders, calculations, transforms, source values) exposes a static `Match` extension taking one `Func` per case. It replaces `IVisitor`/`Accept` ceremony *and* one-method `IXxxJudge` services. Because `Match` is exhaustive, adding a new case is a compile-time forcing function — there is no `_ => default` catch-all.

**Replace a visitor with a `Match` extension over a switch expression.** `[C#]` [domain:modeller]
```csharp
// ❌ before — accept/visit ceremony on every node type
public interface ISourceValue { TOut Accept<TOut>(ISourceValueVisitor<TOut> visitor); }
public sealed class FloatSourceValue(float value) : ISourceValue {
    public float Value { get; } = value;
    public TOut Accept<TOut>(ISourceValueVisitor<TOut> v) => v.Visit(this);
}
// ✅ after — marker interface + static Match; subtypes are bodyless
public interface ISourceValue;
public static class SourceValueExtensions {
    [Pure] public static T Match<T>(this ISourceValue value,
        [InstantHandle] Func<StringSourceValue, T> @string,
        [InstantHandle] Func<FloatSourceValue, T> @float, /* ...all cases... */) =>
        value switch { StringSourceValue v => @string(v), FloatSourceValue v => @float(v),
                       _ => throw new ArgumentOutOfRangeException(nameof(value)) };
}
public sealed class FloatSourceValue(float value) : ISourceValue { public float Value { get; } = value; }
```
The `Func`-per-case `Match` keeps exhaustiveness without the `IVisitor`/`Accept` boilerplate. — 24be6e5f98a

**Call `Match` at the consumer instead of a one-off inner visitor class.** `[C#]` [domain:modeller]
```csharp
// ❌ before — a private one-off IVisitor implementation
public SourceValue ToSerializationModel(ISourceValue v) => v.Accept(new SourceValueConverterVisitor());
private class SourceValueConverterVisitor : ISourceValueVisitor<SourceValue> {
    public SourceValue Visit(StringSourceValue value) => new() { String = value.Value }; /* ...11 overloads... */
}
// ✅ after — inline lambdas, no class
public SourceValue ToSerializationModel(ISourceValue v) => v.Match(
    @string  => new SourceValue { String = @string.Value },
    dateTime => new SourceValue { DateTime = dateTime.Value }, /* ... */
    integer  => new SourceValue { Integer = integer.Value });
```
— 24be6e5f98a

**Collapse a single-method `IXxxJudge` service into a private static `Match`.** `[C#]` [domain:modeller]
```csharp
// ❌ before — DI'd IIsConfigurableStepJudge whose whole body is one Match
private bool IsConfigurableNextStep([NotNullWhen(true)] IStep? next)
    => next != null && configurableStepJudge.IsConfigurableStep(next);
// ✅ after — judge deleted, logic inlined as a static
private static bool IsConfigurableStep(IStep step) => step.Match(
    initialMapping: _ => true, mapping: _ => true, sql: _ => true,
    union: _ => true, aggregate: _ => true, unpivot: _ => true, history: _ => true,
    cdm: _ => false, classification: _ => false, rest: _ => false, forecasting: _ => false);
```
A stateless "Judge" that's one exhaustive `Match` doesn't earn an interface or a DI registration. — fb9bfd0b615

**A total `Match` is the registration point — new types opt in/out explicitly, never via a fallback.** `[C#]` [domain:modeller]
```csharp
// ❌ before — every step type silently "supported"
classification: _ => true, rest: _ => true, forecasting: _ => true);
// ✅ after — new step types opt out of column mapping explicitly
classification: _ => false, rest: _ => false, forecasting: _ => false);
```
Adding a step type forces a compile-time decision at every `Match` site (cost, query-build, timestamp); unsupported behaviour is a deliberate `false`, not a default. Introduce a calculation type → touch each `Match`, no `_ => default`. — be28e99dc16, 6cef6a8a04a

**Hydrater dispatches loader-state to concrete loaders via one `state.Match`.** `[C#]` [domain:modeller]
```csharp
public ILoader Hydrate(ILoaderState state, IPipelineReferenceFactory pipelineReferenceFactory, IDataSourceReferenceFactory dataSourceReferenceFactory) =>
    state.Match<ILoader>(
        dataSourceTableLoader => new FromDataSourceTableLoader(dataSourceTableLoader, dataSourceReferenceFactory, derivedSourceLoadSettingsProvider),
        dataSourceQueryLoader => new FromDataSourceQueryLoader(dataSourceQueryLoader, dataSourceReferenceFactory, derivedSourceLoadSettingsProvider),
        dateSettingsLoader   => new FromDateSettingsLoader(dateSettingsLoader, dateSettingAnalyticsProvider),
        pipelineLoader       => new FromPipelineLoader(pipelineLoader, pipelineReferenceFactory, loaderSystemColumnsProvider),
        warehouseQueryLoader => new FromWarehouseQueryLoader(warehouseQueryLoader, pipelineReferenceFactory));
```
One `Match` over loader-state is the single hydration point; factories receive what each loader needs, nothing more. Use method-group refs for real handlers and `name: _ => const` for constant cases. — bcf4331db52, f43136dff5b

---

## Pipeline DI, hydraters & factories

**Make a domain service injectable: `[SingleInstance(As = [typeof(I)])]` and push a niche dependency from ctor to method param.** `[C#]` [domain:modeller]
```csharp
// ❌ before — provider injected into ctor just to compute a derived value
public class PipelineHydrater(... IOptimizeForWarehouseReportingProvider modelMaterializationProvider, ...)
public WarehouseMaterializationMode DerivedMaterializationMode { get { ... optimizeForWarehouseReportingProvider.OptimizeForWarehouseReporting.Value ... } }
// ✅ after — registered explicitly against the interface; the niche dependency is a method parameter
[SingleInstance(As = [typeof(IPipelineHydrater)])]
public class PipelineHydrater(ILoaderHydrater loaderHydrater, IStepHydrater stepHydrater, ...) : IPipelineHydrater
public WarehouseMaterializationMode DerivedMaterializationMode(IOptimizeForWarehouseReportingProvider optimizeForWarehouseReportingProvider) =>
    state.MaterializationMode switch { PipelineMaterializationMode.Auto => ..., _ => throw new ArgumentOutOfRangeException(nameof(MaterializationMode)) };
```
A dependency used in one method belongs in that method's signature, not the constructor. Register against the interface (`As = [typeof(I)]`), not the concrete type. — f43136dff5b

**Inject a shared provider instead of `new`-ing a dependency graph in a field.** `[C#]` [domain:modeller]
```csharp
// ❌ before — loader hand-builds its collaborator (and its collaborator's collaborator)
public class FromDateSettingsLoader(IFromDateSettingsLoaderState state) : Loader(state), IFromDateSettingsLoader {
    private readonly DateSettingAnalyticsProvider _analyticsProvider = new(new CalendarToHierarchyIdConverter());
    public AnalyticsOverrides AnalyticsOverrides => _analyticsProvider.SourceAnalytics(state.DateSettings);
}
// ✅ after — provider injected, threaded through the factory & hydrater that construct the loader
public class FromDateSettingsLoader(IFromDateSettingsLoaderState state, IDateSettingAnalyticsProvider dateSettingAnalyticsProvider)
    : Loader(state), IFromDateSettingsLoader {
    public AnalyticsOverrides AnalyticsOverrides => dateSettingAnalyticsProvider.SourceAnalytics(state.DateSettings);
}
```
A `new X(new Y())` field is a missed registration — inject the interface and thread it through the factory/hydrater. — bcf4331db52

**Shape a pipeline Job as per-run state on the ctor + a `[SingleInstance]` factory holding the singleton deps.** `[C#]` [domain:modeller]
```csharp
// ✅ Job ctor = per-run state; factory = injected singletons
public class CleanWarehouseRelationshipsJob(
    IReadOnlyModel model, IDataMigrationRetryableOperationRunner retryableOperationRunner,
    IWarehouseOperationContext warehouseOperationContext, IPublishedWarehouseRelationshipDropper publishedWarehouseRelationshipDropper)
    : Job, ICleanWarehouseRelationshipsJob { /* ... */ }

[SingleInstance(As = [typeof(ICleanWarehouseRelationshipsJobFactory)])]
public class CleanWarehouseRelationshipsJobFactory(
    IDataMigrationRetryableOperationRunner runner, IPublishedWarehouseRelationshipDropper dropper)
    : ICleanWarehouseRelationshipsJobFactory
{
    public ICleanWarehouseRelationshipsJob CreateJob(IReadOnlyModel model, IWarehouseOperationContext ctx)
        => new CleanWarehouseRelationshipsJob(model, runner, ctx, dropper);
}
```
Per-run inputs (`model`, `WarehouseOperationContext`) belong on the Job; the factory holds the service deps and injects them in `CreateJob`. — 73b9f0daea4

---

## Domain types & records

**Type a domain reference with a purpose-built record, not a borrowed general interface.** `[C#]` [domain:modeller]
```csharp
// ❌ before — reused ILocalColumnReference; ColumnId/Name nullable "just in case"
public class UnpivotColumnSettings(ColumnId inputColumnId, ILocalColumnReference measureColumn, PivotKeyValue pivotKeyValue)
{
    public ILocalColumnReference MeasureColumn { get; set; } = measureColumn;
}
// ✅ after — a record that says exactly what an unpivot measure column is
public class UnpivotColumnSettings(ColumnId inputColumnId, UnpivotMeasureColumnReference measureColumn, PivotKeyValue pivotKeyValue)
{
    public UnpivotMeasureColumnReference MeasureColumn { get; set; } = measureColumn;
}
public record UnpivotMeasureColumnReference(ColumnId ColumnId, string Name);
```
The dedicated record drops the optional `?` on `ColumnId`/`Name` — the unpivot case always has both. — b35be03849d

**Replace a hand-rolled `OneOfBase` + manual equality with a sealed record union.** `[C#]` [domain:modeller]
```csharp
// ❌ before — OneOf with a hashcode hack for value equality
public abstract class CalculationStatus : OneOfBase<...>, IEquatable<CalculationStatus>
{
    public override bool Equals(object obj) => obj?.GetType() == GetType();
    public override int GetHashCode() => 0; // OneOf issue #28
    public class ValidStatus : CalculationStatus;
}
// ✅ after — records give structural equality for free
public abstract record CalculationStatus
{
    public record ValidStatus : CalculationStatus;
    public record UntestedStatus : CalculationStatus;
}
```
A closed status hierarchy is a record union; let the compiler supply `Equals`/`GetHashCode` instead of a `GetHashCode() => 0` workaround. — 2f0d1b7cc2d

**Expose a strongly-typed covariant `State` so callers stop casting.** `[C#]` [domain:modeller]
```csharp
// ❌ before — every caller casts the base State
var loaderState = (IFromWarehouseQueryLoaderState)pipeline.State.Loader;
new FromWarehouseQueryLoaderSqlStatement(loaderState).Update(query, new List<IPipelineReferenceState>());
// ✅ after — `new` covariant State on the interface; callers get the right type for free
public interface IFromWarehouseQueryLoader : IFromQueryLoader { new IFromWarehouseQueryLoaderState State { get; } }
public new IFromWarehouseQueryLoaderState State => state;
new FromWarehouseQueryLoaderSqlStatement(pipeline.Loader.State).Update(query, []);
```
A `new`-typed property on the derived interface kills the cast at every call site. — 607dd32d34d

**Use `required` init properties instead of nullable-to-be-set-later option bags.** `[C#]` [domain:modeller]
```csharp
// ❌ before — settable, implicitly nullable, must remember to set
public string SqlStatement { get; set; }
public IEnumerable<PipelineReferenceViewModel> Tokens { get; set; }
// ✅ after — compiler enforces they're provided
public required string SqlStatement { get; set; }
public required IEnumerable<PipelineReferenceViewModel> Tokens { get; set; }
```
`required` moves "did you forget to set this?" from runtime to compile time. — 607dd32d34d

---

## Column updates & batching

**Add a batched overload taking `IReadOnlyList<ColumnId>`, make the singular one delegate, then delete the singular once every caller is migrated.** `[C#]` [domain:modeller]
```csharp
// step 1 — batch path; single-column path is a thin wrapper (839c3b4ff60)
public void CascadeColumnNameChangesInPipeline(IPipeline pipeline, ColumnId columnId, StepSnapshot stepSnapshot)
    => CascadeColumnNameChangesInPipeline(pipeline, [columnId], stepSnapshot);
public void CascadeColumnNameChangesInPipeline(IPipeline pipeline, IReadOnlyList<ColumnId> columnIds, StepSnapshot stepSnapshot)
{
    CascadeColumnDatabaseNameChangesInPipeline(pipeline, columnIds, stepSnapshot);
    pipelineColumnReferencesUpdater.Update(pipeline);
    columnTranslationsUpdater.Update(pipeline);
}
// step 2 — once every caller passes a list, the singular overload is deleted (e66e6411ab4)
void CascadeColumnDatabaseNameChanges(IPipeline pipeline, Guid stepId, IReadOnlyList<ColumnId> columnIds, StepSnapshot stepSnapshot);
```
Cascading one column at a time re-walks the pipeline per column; batch the `ColumnId`s so one cascade handles them all, then remove the migration scaffolding so the API has one shape. — 839c3b4ff60, e66e6411ab4

---

## SQL / Q-graph construction

**Build SQL through the Q-graph with a named local function for the per-column expression.** `[C#]` [domain:modeller]
```csharp
var qColumns = tableSchema.Columns.Select(c => CreateQColumnForMaterializeWithNewTimestamps(c, NewTimestampExpression, innerSelectTable));
query.AddColumns(qColumns);
return query;

QCase NewTimestampExpression(WarehouseTableColumn column) {
    var originalTimestamp = new QColumn(column.Name) { Table = innerSelectTable };
    return new QCase(
        cases: [ new QCaseWhenThenStatement(
                   new QTwoExpressionConstraint(originalTimestamp, ConstraintOperator.GreaterThan, publishTimestamp),
                   originalTimestamp) ],
        elseCase: publishTimestamp);
}
```
Q-graph constructors take named args (`cases:`, `elseCase:`); a local function names the intent and is reused by the constant-timestamp overload (`_ => newTimestampExpression`). — 87a88d22f36

**Split an ambiguous overloaded `GenerateSql` into intent-named methods by context.** `[C#]` [domain:modeller]
```csharp
// ❌ before — same name for "this is the whole query" vs "this is a select source"
$"FROM ({fromSql.GenerateSql(sqlGenerator)}) AS SOURCE"   // which behaviour?
// ✅ after — the name says where the SQL is going
public interface IQuerySql : ISelectSource { string GenerateQuerySql(IQuerySqlGenerator generator); /* ... */ }
$"FROM ({fromSql.GenerateSelectSourceSql(sqlGenerator)}) AS SOURCE"   // subquery form
return select.GenerateQuerySql(sqlGenerator);                          // top-level form
```
When one method does two jobs depending on context, give each context its own named method. — a90528276b6

---

## Surgical edits & boundaries

**Take `IReadOnlyList<T>` at the boundary so the body stops re-buffering with `.ToList()`.** `[C#]` [domain:modeller]
```csharp
// ❌ before — IEnumerable param, defensively copied, multiply enumerated
IEnumerable<IHydratedLookupSettings> lookupColumns, ...
private readonly IReadOnlyList<IHydratedLookupSettings> _lookups = lookupColumns.ToList();
int depth = _lookups.Max(c => c.LookupPath.Count);
// ✅ after — already a list; use the parameter directly
IReadOnlyList<IHydratedLookupSettings> lookupColumns, ...
int depth = lookupColumns.Max(c => c.LookupPath.Count);
```
When the body iterates a sequence several times, demand a list at the signature and the internal copy disappears. — defd272d76c

**Early-return the no-op case and build the changed value with a `with` expression.** `[C#]` [domain:modeller]
```csharp
// ❌ before — nested if, mutate qColumn in place
if (input.DataType != setting.DataType) {
    if (!IsSafeImplicitCast(input, setting)) {
        qColumn.DataType = setting.DataType.ToSqlDataType();
        qColumn.ExplicitCast = true;
    }
}
return qColumn;
// ✅ after — guard, then a non-mutating `with`
if (input.DataType != setting.DataType && !IsSafeImplicitCast(input, setting))
    return qColumn with { DataType = setting.DataType.ToSqlDataType(), ExplicitCast = true };
return qColumn;
```
— 51b3fe61487

**Drop a parameter the whole call chain ignores — interface, base, impls, tests, in one commit.** `[C#]` [domain:modeller]
```csharp
// ❌ before — CancellationToken accepted but never observed
string? GetWarehouseInitializationScript(Collation collation, CancellationToken cancellationToken);
// ✅ after — removed everywhere in one surgical pass
string? GetWarehouseInitializationScript(Collation collation);
```
Unused parameters are noise; remove them across the entire chain, tests included, in one commit. — 50d9f17472a

**Mark an enumerator passthrough that hands out a disposable enumerator `[MustDisposeResource]`.** `[C#]` [domain:modeller]
```csharp
[MustDisposeResource]
public IEnumerator<IHydratedCalculationTemplateSetting> GetEnumerator() => settings.GetEnumerator();
[MustDisposeResource]
IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
```
A wrapper that returns the underlying list's enumerator declares it so callers don't leak it. — a203d385a76

---

## Notes on style divergence (read before scoring)

Robert's modeller code is overwhelmingly consistent with the house rubric (surgical minimum-diff, CLEAN intent-revealing names, modern idiom: records, `with`, collection expressions, primary constructors, named `Match` arms). Two pragmatic divergences appear in the history and should **not** be copied as ideals, nor scored as house style:

- **Transitional double overloads.** A batch overload sometimes lands alongside the singular one, with the singular removed in a *later* commit. That's a staged migration, not the target state — the end state (one API shape) is what the rubric rewards.
- **A few domain "why" comments and feature-switch guards** survive in step/calculation code (e.g. `ModellerCoreSwitch.LessSqlCasting.Enabled`). The comments axis stays on the strict zero-comments line; a genuine non-obvious-constraint why-line is allowed, decorative dividers (`//Options`) are not.
