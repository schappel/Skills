# zap-style · Tests (concern shard)

Test idioms. Loaded when a change has test surface. Score against the rubric in `../SKILL.md` (axis 7).

## ⭐ LEARNING-PROXY (tests) — corrections a reviewer demanded in PR review

Highest-signal test-style evidence: changes made *because a reviewer asked*. Follow these over the plain examples if they conflict.

**Don't fabricate test data — reuse the named fixture constant.** `[C#] [learning-proxy]`
```csharp
// ❌
sut.SetMeasureColumn(step, InputColumnId, new UnpivotMeasureColumnSettings(new DisplayName("foo"), MeasureId));
// ✅
sut.SetMeasureColumn(step, InputColumnId, new UnpivotMeasureColumnSettings(MeasureName, MeasureId));
```
An ad-hoc `"foo"` paired with the real `MeasureId` is misleading; use the shared `MeasureName`. — 1c9e3fa

**Use the real fake / null-object, not a mock framework, for collaborators.** `[C#] [learning-proxy]`
```csharp
// ❌
new OrganizationProfileChecker(new SpecifiedUserProvider(_user), Mock.Of<IOrganizationSource>(s => s.Organization == Organization.Current));
// ✅
new OrganizationProfileChecker(new SpecifiedUserProvider(_user), new SpecifiedOrganizationSource(Organization.Current!));
```
Reach for the existing `Specified…` fake / `Logger.None` before `Mock.Of` / `Substitute.For<ILogger>`. — 3ed9f59, 997f430, d498014

**Don't expand assertions by hand when a tailored EasyAssertion exists.** `[C#] [learning-proxy]`
```csharp
// ❌
result.GetChildResources().ItemsSatisfy(c => c.ShouldBeA<Folder>())
    .And.Single().GetChildResources().ItemsSatisfy(gc => gc.ShouldBeA<Blob>());
// ✅
result.GetChildResources().ShouldBeASingular<Folder>()
    .And.GetChildResources().ShouldBeASingular<Blob>();
```
`ShouldBeASingular<T>()` checks count + type + returns the item in one call. — d498014

**Assert outcomes through EasyAssertions; keep asynchronous operations as tasks.** `[C#] [learning-proxy]`
```csharp
// ✅ successful Task<T>: assert completion, then continue with its result
client.ListToolsAsync()
    .ShouldComplete()
    .And.Select(tool => tool.Name)
    .ShouldOnlyContain(expectedToolNames);

// ✅ expected synchronous exception
Should.Throw<NotSupportedException>(() => CreateInputSchema(methodName))
    .And.Message.ShouldStartWith("Type is not supported");

// ✅ expected asynchronous exception
client.CallToolAsync(toolName, arguments)
    .ShouldFail()
    .And.Message.ShouldContain("authorization");
```
Use `Should.Throw<T>` for a synchronous `Action`/`Func`; use `ShouldFailWith<T>` or `ShouldFail().And` for `Task`/`ValueTask`. The [EasyAssertions assertion reference](https://github.com/soxtoby/EasyAssertions/blob/master/doc/Assertions.md) documents the split. — 98baa78a0, 830e88658

**Configure a task-returning substitute with `.ThrowsAsync(...)`.** `[C#] [learning-proxy]`
```csharp
// ❌ tenantClient.GetByAliasAsync(alias).Throws(exception);
// ✅
tenantClient.GetByAliasAsync(alias).ThrowsAsync(exception);
```
Match the substitute's asynchronous contract; reserve `.Throws(...)` for synchronous members. — 830e88658

**Keep nullable analysis on in fixtures; initialize setup-owned fields with `null!`.** `[C#] [learning-proxy]`
```csharp
// ❌
#nullable disable
[TestFixture]
public class TenantTests { private ITenantClient _tenantClient; }
// ✅
public class TenantTests { private ITenantClient _tenantClient = null!; }
```
Ordinary NUnit classes are discovered without `[TestFixture]`; use `null!` only when setup assigns the field before every test. — 28148d174, 4f76a15fd

---

## Tests

**Canonical shape: `Method_Condition_ExpectedResult`; AAA split by blank lines; locals `sut`/`result`; one behaviour per test.** `[C#]`
```csharp
[Test]
public void IsNullable_WhenParameterIsExplicitlyNullableReferenceType_ReturnsTrue()
{
    var sut = GetParameterInfo(typeof(NullableEnabledMethods), nameof(NullableEnabledMethods.NullableReference));

    var result = NullableMetadata.IsNullable(sut);

    result.ShouldBe(true);
}
```
— 310635c0796

**Prefer hand-written fakes over `Substitute.For` + `Arg.Is(...).Returns(...)`; drive behaviour via plain properties.** `[C#]`
```csharp
// ❌ _copilotAgent = Substitute.For<ICopilotAgent>(); _copilotAgent.TakeTurn(Arg.Any<...>(), capabilities, Arg.Any<...>()).Returns(response);
// ✅ _copilotAgent = new FakeCopilotAgent(); _copilotAgent.NextResponse = new AgentResponse(…);
```
— 76361fc3f04

**Idiomatic fake: settable props drive behaviour; `NextResponse!` for configured value; `NotImplementedException` for unused members.** `[C#]`
```csharp
internal class FakeCopilotAgent : ICopilotAgent
{
    public AgentResponse? NextResponse { get; set; }
    public Exception? ThrowOnTakeTurn { get; set; }

    public Task<AgentResponse> TakeTurn(IReadOnlyList<Message> conversation, AgentCapabilities capabilities, CancellationToken cancellationToken)
    {
        if (ThrowOnTakeTurn is not null)
            throw ThrowOnTakeTurn;
        return Task.FromResult(NextResponse!);
    }

    public IAsyncEnumerable<MessagePart> StreamTurn(…) => throw new NotImplementedException();
}
```
— 76361fc3f04

**A real `SpecifiedUserProvider` fake replaces `Substitute.For<IUserProvider>() + .Returns()`.** `[C#]`
```csharp
// ❌ _userProvider = Substitute.For<IUserProvider>(); _userProvider.User.Returns(_testUser); _sut = new CopilotBot(…, _userProvider, …);
// ✅ _sut = new CopilotBot(…, new SpecifiedUserProvider(_testUser), …);
```
— 76361fc3f04

**Fake records each interaction into a `List` exposed as `IReadOnlyList` (as a record) — assert over it like any result.** `[C#]`
```csharp
class FakeTeamsNotificationSender : ITeamsAiNotificationSender
{
    readonly List<TeamsNotification> _sentNotifications = [];
    public IReadOnlyList<TeamsNotification> SentNotifications => _sentNotifications;

    public async Task SendNotification(TeamsBotEnvironment botEnvironment, string conversationId, string userId, IReadOnlyList<string> messages, IReadOnlyList<CopilotImageAttachment> imageAttachments, CancellationToken cancellationToken) =>
        _sentNotifications.Add(new TeamsNotification(botEnvironment, conversationId, userId, messages, imageAttachments));
}
record TeamsNotification(TeamsBotEnvironment BotEnvironment, string ConversationId, string UserId, IReadOnlyList<string> Messages, IReadOnlyList<CopilotImageAttachment> ImageAttachments);
```
— 5563ac7c640

**Stateful fake: `ConfigureResponse(Regex, string)` seeds replies, `RequestCount` records calls — stub + verify, no framework.** `[C#]`
```csharp
class FakeChatClient : IChatClient
{
    readonly List<(Regex Pattern, string Response)> _responses = [];
    public int RequestCount { get; private set; }
    public void ConfigureResponse(Regex messagePattern, string response) => _responses.Add((messagePattern, response));

    public Task<ChatResponse> GetResponseAsync(IEnumerable<ChatMessage> messages, ChatOptions? options = null, CancellationToken cancellationToken = default)
    {
        RequestCount++;
        var userPrompt = messages.Last(m => m.Role == ChatRole.User).Text;
        var response = _responses.Select(r => r.Pattern.IsMatch(userPrompt) ? r.Response : null);
        return Task.FromResult(new ChatResponse(new ChatMessage(ChatRole.Assistant, response.FirstNotNull())));
    }
}
```
— 5563ac7c640

**Fake repositories use backing `Dictionary`s with real interface semantics (ordering, `?? []`).** `[C#]`
```csharp
class FakeNotificationRepository : IAINotificationRepository
{
    readonly Dictionary<Guid, List<AINotificationEvaluation>> _evaluations = [];
    public void AddConfig(AINotificationConfig config) { _configs[config.Id] = config; if (!_evaluations.ContainsKey(config.Id)) _evaluations[config.Id] = []; }
    public IReadOnlyList<AINotificationEvaluation> GetEvaluationsByConfigId(Guid configId) => _evaluations.Get(configId)?.OrderByDescending(e => e.RunAt).ToArray() ?? [];
}
```
— 5563ac7c640

**NSubstitute is still fine for pure read-only stubs that just return canned data (no interaction verification) — but only when no hand-written `Specified…`/fake already exists for the interface.** If one exists, use it — that's the ⭐ learning-proxy above (3ed9f59): its ❌ was a read-only stub too; the sin was ignoring the existing fake. `[C#]`
```csharp
var cubeDefinitionFactory = Substitute.For<ICubeDefinitionFactory>();
cubeDefinitionFactory.GetCubeDef(Arg.Any<ICubeConnectionInfo>()).Returns(MockZapBITraining.CubeDef);
_sut = new AnalysisConverter(new CubeConnectionConverter(connectionProvider, connectionFactory, cubeDefinitionFactory));
```
— f53c2bfb1c6

**Use EasyAssertions' collection vocabulary, not LINQ-into-`ShouldBe`.** `[C#]`
```csharp
// ❌ evaluations.Count.ShouldBe(5); evaluations.All(e => e.Error != string.Empty).ShouldBe(true);
// ✅
_notificationRepository.GetEvaluationsByConfigId(config.Id).ShouldBeLength(5).And.AllItemsSatisfy(e => e.Error.ShouldNotBeEmpty());
```
— f5ff00a0808

**Assert the intent, not the integer: `.ShouldBeEmpty()` over `.Count.ShouldBe(0)`.** `[C#]`
```csharp
// ❌ _notificationSender.SentNotifications.Count.ShouldBe(0);
// ✅ _notificationSender.SentNotifications.ShouldBeEmpty();
```
— f5ff00a0808

**`.ShouldNotBeNull()` / `.ShouldBeASingular<T>()` narrow AND return the value, so `.And.` keeps drilling in one chain.** `[C#]`
```csharp
result.ObjectArray.ShouldBeASingular<ReferencableObject>()
    .And.Id.ShouldNotBeNull()
    .And.ToString().ShouldNotBe(fakeRefId);

JsonSerializer.Deserialize<TestBool>(value ? "true" : "false", _testOptions)
    .ShouldNotBeNull()
    .And.Value.ShouldBe(value);
```
— eefe26b2a0b, 9fea45712b1

**Assert structured output whole with `.ShouldMatchJson("""…""")`, not field-by-field.** `[C#]`
```csharp
var result = CreateInputSchema(nameof(TestMethods.ScalarEnumAndArray));
result.ShouldMatchJson(
    """
    {
        "type": "object",
        "properties": {
            "requiredText": { "type": "string", "description": "Required text" },
            "choice": { "type": "string", "enum": ["the_first", "the_second"] }
        },
        "required": ["requiredText", "choice"],
        "additionalProperties": false
    }
    """);
```
— fc8460aa42f

**Parameterise with `[TestCase]`, expected JSON inline as a raw string literal — the truth table reads in the attributes.** `[C#]`
```csharp
[TestCase(typeof(TestBool), """{"type":"boolean"}""")]
[TestCase(typeof(TestGuid), """{"type":"string","format":"uuid"}""")]
public void JsonSchema_UsesPrimitiveType(Type typedType, string expectedSchema)
{
    var result = GetSchema(typedType).ToJsonString();

    result.ShouldBe(expectedSchema);
}
```
— 9fea45712b1

**NUnit `[Theory]` with a `bool` param auto-runs both values; derive the expectation inline from the input.** `[C#]`
```csharp
[Theory]
public void TypedBool_SerializedAsBoolean(bool value)
{
    JsonSerializer.Serialize(new TestBool(value), _testOptions).ShouldBe(value ? "true" : "false");
}
```
— 9fea45712b1

**Round-trip tests: act = `ToSerialized` then `ToDomain`; `.ShouldMatch([...])` for collections, `.ShouldBe` for scalars. AAA is split by blank lines — no `// Act`/`// Assert` banners.** `[C#]`
```csharp
var serialized = (SerializedAnalysis)_sut.ToSerialized(original);
var result = (Analysis)_sut.ToDomain(serialized);

result.Query.ColumnsAxis.Sets.Sets.GetUniqueNamesForAssertions().ShouldMatch([Location.CountryUn, Measures.GrossAmtUn]);
result.PivotTableAppearance.RowAndColumnStyles[rowOrColumnTuple.ToCellAddress()][StyleProperty.BackgroundColor].ShouldBe("#112233");
```
— ef472a90479

**Test fixtures live at the bottom: a `file static` class of sample methods + one-line primary-ctor subclasses as typed doubles.** `[C#]`
```csharp
file static class TestMethods
{
    public static void ScalarEnumAndArray(…) { }
    public static void DictionaryInput(SortedDictionary<string, int> entries) { }
}
class TestBool(bool value) : TypedBool(value);
class TestGuid(Guid value) : TypedGuid(value);
```
— 9fea45712b1

**Private static expression-bodied helpers live below the tests; lookup uses `?? throw` so a typo fails loudly.** `[C#]`
```csharp
static MethodInfo GetMethod(string methodName) => typeof(TestMethods).GetMethod(methodName, BindingFlags.Static | BindingFlags.Public | BindingFlags.NonPublic)
    ?? throw new InvalidOperationException($"Missing test method '{methodName}'.");
```
— fc8460aa42f

**To exercise nullable-context behaviour, declare fixtures under `#nullable disable` / `#nullable restore`.** `[C#]`
```csharp
class NullableEnabledMethods
{
    public void NullableReference(string? value) { }
}

#nullable disable
class NullableDisabledMethods
{
    public void Reference(string value) { }
}
#nullable restore
```
— 310635c0796
