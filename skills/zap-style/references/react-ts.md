# zap-style · React & TypeScript idioms (concern shard)

Cross-cutting frontend house idioms. Loaded for any TS/React change. Score against the rubric in `../SKILL.md`.

**Contents:** ⭐ Learning (TS) · React component structure · React rendering & dispatch · React infra (controllers, models, event-reduce) · TypeScript types & narrowing

## ⭐ LEARNING (TS) — a senior rewrote skill-authored code (highest signal)

**Don't wrap a model in a pass-through @Injectable controller — pass the model as a prop and inline the lone verb.** ⭐ `[TS] [learning]`
```tsx
// ❌ wrapper controller that only re-exposes the model and forwards one call
export const ColorSchemePicker = reactive(function ColorSchemePicker() {
    let controller = useController(ColorSchemeController);
    return <Row …>{… c == controller.currentScheme … onClick={() => controller.changeColorScheme(c)}}</Row>
});
@Injectable
class ColorSchemeController {
    constructor(private _model: ColorSchemeModel) { }
    get currentScheme() { return this._model.scheme; }
    changeColorScheme(scheme: PaletteName) { this._model.events.colorSchemeSelected(scheme); applyColorScheme(scheme); }
}
// ✅ model passed as a prop; the lone verb is a local function after the JSX
export const ColorSchemePicker = reactive(function ColorSchemePicker({ model }: { model: ColorSchemeModel }) {
    return <Row …>{… c == model.scheme … onClick={() => changeColorScheme(c as PaletteName)}}</Row>

    function changeColorScheme(scheme: PaletteName) {
        model.events.colorSchemeSelected(scheme);
        applyColorScheme(scheme);
    }
});
```
Same lesson as the helper-reuse learning, in TS: delete needless indirection. — 32c133ab406

---

## ⭐ LEARNING-PROXY (TS) — corrections a reviewer demanded in PR review

Changes made *because a reviewer asked*. Follow these over plain examples if they conflict.

**Use the intent-revealing collection idiom.** `[TS] [learning-proxy]`
```ts
// ❌
return this._pipelineOutputSchema.columns.filter(c => c.IsPrimaryKey).length > 0;
// ✅
return this._pipelineOutputSchema.columns.some(c => c.IsPrimaryKey);
```
`.some()` says "any?" and short-circuits; `.filter().length > 0` allocates and reads as a count. — 6062b8d

**Use arrow functions with lexical `this`; drop the captured-alias dance.** `[TS] [learning-proxy]`
```ts
// ❌
let fullyQualifiedTableNameGenerator = this._fullyQualifiedTableNameGenerator;
return tables.map(function (table) { return new SchemaTableViewModel(table, conciseGen, fullyQualifiedTableNameGenerator); });
// ✅
return tables.map(table => new SchemaTableViewModel(table, conciseGen, this._fullyQualifiedTableNameGenerator));
```
Arrow closes over `this`, so the temp alias and the verbose `function` wrapper both vanish. (Also: flatten `.catch().then()` nesting; drop superfluous parens.) — 210b842, 4846af3, b672fc7

---

## React: component structure

**Canonical shape: `reactive(named fn)` + separate `IXProps`; hooks top, single JSX tree, handlers after the return.** `[TS]`
```tsx
export const QuestionsPopup = reactive(function QuestionsPopup({ model, onResponsesSubmitted }: IQuestionsPopupProps) {
    let [requestIndex, setRequestIndex] = useState(0);
    let currentRequest = model.pendingRequests[requestIndex];

    return <FocusScope wrap autoFocus restoreFocus>…</FocusScope>

    function previousQuestion(): boolean | void {
        return setRequestIndex(Math.max(requestIndex - 1, 0));
    }
});
```
— ebfae0230f4

**When a component reads observable state, wrap it in `reactive(function Name(){…})` — keep the named form for stack traces.** `[TS]`
```tsx
const ChatContext = reactive(function ChatContext({ model, controller }: { model: CopilotChatModel, controller: CopilotChatController }) {
    return <>…reads model.mode, controller.activeContextTypes…</>
});
```
— 42bf6a52432

**Event handlers are hoisted function declarations after the JSX, so render reads top-down and `onClick={verb}`.** `[TS]`
```tsx
export function CubeLink({ url, children }: ILinkComponentProps) {
    let controller = useController(CubeLinkController);
    let iconClass = iconClasses[type];

    return <Row inline spacing="none" styles={[link, cubeNodeStyle]} onClick={findInDimensionTree}>
        {iconClass && <Styled.span styles={linkIcon} className={iconClass} />}
        <Styled.span styles={linkText}>{children}</Styled.span>
    </Row>

    function findInDimensionTree() {…}
}
```
— 79154642925

**Lift a fat inline JSX block into a named sub-component — pure verbatim extraction, behaviour-preserving.** `[TS]`
```tsx
// ❌ <Row …><Tooltip …><OverflowMenu …>{/* ~40 inline lines */}</OverflowMenu></Tooltip></Row>
// ✅
<Row spacing="none" sidePadding styles={copilotMessageButtons}>
    <ChatContext model={model} controller={controller} />
</Row>

function ChatContext({ model, controller }: { model: CopilotChatModel, controller: CopilotChatController }) {
    return <>{/* the same JSX, moved verbatim */}</>
}
```
— 1430549f301

**Non-trivial prop sets get a named `I…Props` that extends `IChildren`, grouped with blank lines by concern.** `[TS]`
```tsx
export interface IMessageInputProps extends IChildren {
    value: string;
    onValueChange: (value: string) => void;

    onKeyDown?: (event: React.KeyboardEvent<HTMLInputElement>) => void;

    autoFocus?: boolean | "recurring";
    styles?: StyleCollection;
}
```
— e86be705b23

**Tiny presentational components keep props typed inline (no interface); `ref?: Ref<T>` is just another prop.** `[TS]`
```tsx
function FolderItem({ name, onClick }: { name: string; onClick: () => void; }) {
    return <Row spacing="subGrid" hideOverflow shrink>
        <Icon name="folder" size="small" />
        <Styled.span onClick={onClick} styles={[resourceName, truncateText]}>{name}</Styled.span>
    </Row>
}
```
— a7589bba83b

**React 19 ref-as-prop: accept `ref?: Ref<T>` and forward it — `forwardRef` is gone.** `[TS]`
```tsx
function AssistantTypingPlaceholder({ show, ref }: { show: boolean; ref?: Ref<HTMLDivElement>; }) {
    return <ChatBubble ref={ref} type="assistant">…</ChatBubble>
}
```
— 881835a124b

**Co-locate the `style()` blocks just below the small component that uses them.** `[TS]`
```tsx
function ContextChip({ context, onRemove }: { context: ICopilotContext; onRemove: () => void; }) {
    return <Tooltip content={context.label}>
        <IconButton size="medium" onClick={onRemove} styles={contextChip}>
            <IconWithOverlay primary={{ name: context.icon, size: 'small', styles: contextChipIcon }} overlay={{ name: 'closeOverlay', size: 'small', styles: contextChipOverlay }} />
        </IconButton>
    </Tooltip>
}
```
— 93e8d4266de

---

## React: rendering & dispatch

**Polymorphic render via a `Record<string, Component>` lookup + ternary fallback — no switch, no if-ladder.** `[TS]`
```tsx
export function CopilotLink({ href, children }: { href: string; } & IChildren) {
    let LinkComponent = linkComponents[new URL(href).protocol];
    return LinkComponent ? <LinkComponent url={new URL(href)}>{children}</LinkComponent> : <span>{children}</span>;
}
type LinkComponent = (props: ILinkComponentProps) => ReactNode;
export const linkComponents: Record<string, LinkComponent> = { 'https:': WebLink, 'zap:': ResourceLink, 'qp:': QueryPart };
```
— 9f6940e2bcd

**Render data is a module-level `Record` keyed by a string-union — TS enforces every case has an entry.** `[TS]`
```tsx
const Suggestions: Record<SuggestionCategory, string[]> = {
    default: [CopilotText.Suggestion_Default1, CopilotText.Suggestion_Default2, CopilotText.Suggestion_Default3],
    model: [CopilotText.Suggestion_Model1, CopilotText.Suggestion_Model2, CopilotText.Suggestion_Model3],
    analytics: […]
};
```
— a0d26f1505c

**Three-way render as a nested ternary passed into a prop, ending in `: null` — no wrapper component.** `[TS]`
```tsx
bodyOverride={
    model.validating.hasEverLoaded && model.resources.length == 0
        ? <Row topBottomPadding sidePadding spacing="half"><Icon name="check" size="large" customColor={successColor} /><span>{allValidMessage}</span></Row>
        : model.validating.isLoading ? <Center topBottomPadding><Loading … /></Center> : null
}
```
— a7589bba83b

**Guarded JSX with chained `&&`; once the guard proves non-null, a trailing `!` assertion is fine.** `[TS]`
```tsx
{message.requestId && message.approved == null &&
    <Column sidePadding topBottomPadding="half" styles={[card, toolUseRequest]}>
        <ButtonList>
            <SplitButton contained onClick={() => onSubmitApproval(message.requestId!, true)} menuChildren={<>
                <MenuItem text="Allow for rest of conversation" onClick={() => onSubmitApproval(message.requestId!, true, true)} />
            </>}>Allow</SplitButton>
            <Button onClick={() => onSubmitApproval(message.requestId!, false)}>Skip</Button>
        </ButtonList>
    </Column>
}
```
— 881835a124b

**Push the gnarly check into a named model getter; key a `useEffect` on a model value instead of manual subscribe.** `[TS]`
```tsx
// ❌ useEffect(() => model.events.partsReceived.item.subscribe(() => { setIsReceivingParts(true); setTimeout(() => setIsReceivingParts(false), defaultLoadingDelay); }), []);
//    let showPlaceholder = model.status == 'pending' && !isReceivingParts && (!lastResponse || !(lastResponse instanceof ToolUseModel && lastResponse.status == 'pending'));
// ✅
useEffect(() => {
    setIsCurrentlyUpdatingResponse(true);
    setTimeout(() => setIsCurrentlyUpdatingResponse(false), defaultLoadingDelay);
}, [messageGroup.lastPartReceivedTime]);

let showPlaceholder = messageGroup.status == 'pending'
    && !messageGroup.lastPartIsPendingToolUse // loading spinner already covers pending tool uses
    && !isCurrentlyUpdatingResponse;
```
— bd7ed87bb2e

**When a parent prop should seed but not lock local state, use `useSemiControlledState(initial, preferred)`.** `[TS]`
```tsx
// ❌ let [category, setCategory] = useState<SuggestionCategory>('default');  // + a syncing useEffect
// ✅ let [category, setCategory] = useSemiControlledState<SuggestionCategory>('default', preferredCategory);
```
— 42bf6a52432

---

## React infra: controllers, models, event-reduce

**Controllers are `@Injectable`; every dep is a `private _camelCase` ctor param, no field bodies.** `[TS]`
```ts
@Injectable
export class CopilotChatController {
    constructor(
        private _model: CopilotChatModel,
        private _service: Services.CopilotChatService,
        private _copilotAccess: CopilotAccessLevels,
        private _snacks: SnackBarModel
    ) { }
```
— 4ed78557aca

**Controllers that own side-effects implement `IDisposable` and hold an optional `_subscription`.** `[TS]`
```ts
@Injectable
class ValidateResourcesTreeController implements IDisposable {
    private _resourceSavedSubscription?: Rx.IDisposable;
    constructor(private _serverEvents: Events, private _scheduler: Rx.Scheduler, private _service: Services.ResourceManipulationService, private _model: ValidateResourcesTreeModel) { }
```
— a7589bba83b

**Derived state is an expression-bodied `get` on one line, reaching through deps with `?.` — not a stored field.** `[TS]`
```ts
get activePaneContext() { return this._paneModel.activePane?.copilotContext; }
```
— 93e8d4266de

**Coerce a possibly-undefined object to bool with `!!` so a `get` returns a clean boolean.** `[TS]`
```ts
get canUseActivePaneContext() {
    return this._copilotAccess.hasDataAccess && !!this.activePaneContext;
}
```
— 93e8d4266de

**Public intent-named method wraps the raw lookup; `?.x ?? fallback` supplies the default. Raw getter goes private.** `[TS]`
```ts
isEnabled(key: string) {
    return this.getOption(key)?.enabled ?? FeatureSwitch.optionEnabled(key);
}
private getOption(key: string) { return this.optionMap.get(key.toLowerCase()); }
```
— 75c564bef61

**Name the boolean condition as a `@derived` getter so components read intent, not raw checks.** `[TS]`
```ts
@derived
get lastPartIsPendingToolUse() {
    let lastPart = this.agentResponses.lastOrNull();
    return lastPart instanceof ToolUseModel && lastPart.status == "pending";
}
```
— bd7ed87bb2e

**A two-way branch with no return value is a bare ternary statement, not if/else.** `[TS]`
```ts
toggleDataAccess() {
    this._copilotAccess.hasDataAccess ? this._copilotAccess.optOut() : this._copilotAccess.optIn();
}
```
— 75c564bef61

**Single-statement if/else-if/else drops the braces; bodies stay one-line — the house default.** `[TS]`
```ts
if (FeatureSwitch.optionEnabled('CopilotRedux'))
    await this._model.events.responsePartsReceived(this.getResponse(latestMessage, priorConversation, signal), turnContext);
else if (FeatureSwitch.optionEnabled('CopilotResponseStreaming'))
    await this._model.events.responseStreamed(this.streamFromAssistant(latestMessage, priorConversation, signal), turnContext);
else
    await this._model.events.assistantResponseGenerated(this.promptAssistant(latestMessage, priorConversation, signal), turnContext);
```
— 4abb7190fd5

**event-reduce model shape: `@events` companion class, public `events` field, ctor-injected deps, `@state` fields.** `[TS]`
```ts
@events
class ApplicationSettingsEvents {
    copilotDataAccessChanged = asyncEvent<void, { allowed: boolean }>();
}

@model
export class ApplicationSettingsModel {
    events = new ApplicationSettingsEvents();
    constructor(private _features: FeatureSwitchesModel) { }

    @state
    fileUploadMaximumMegabytes = ZapVariables.ApplicationSettings.FileUploadMaximumMegabytes;
```
— 75c564bef61

**Optimistic UI: apply the requested value on `.started`, revert on `.rejected` — the asyncEvent lifecycle drives the reducer.** `[TS]`
```ts
@reduced
copilotDataAccess = reduce(ZapVariables.ApplicationSettings.CopilotDataAccess, this.events)
    .on(e => e.copilotDataAccessChanged.started, (_, { context }) => context.allowed)
    .on(e => e.copilotDataAccessChanged.rejected, (_, { context }) => !context.allowed)
    .value;
```
— 75c564bef61

**Initial-then-latest-from-one-event → `setFrom(initial, event.map(...))`, not a full `reduce().on().value`.** `[TS]`
```ts
@reduced
name = setFrom(this._initial.Name, this.events.thisResourceValidated.map(r => r.Name));
```
— a7589bba83b

**Reducer returns a spread-updated copy via nested ternaries, falling through to `current` to no-op — never mutates.** `[TS]`
```ts
@reduced
private _state = reduce({ text: this._initialState.text ?? '', status: this._initialState.status }, this.events)
    .on(e => e.partReceived.item, (current, { item }) => current.status == 'streaming'
        ? item.Type == ResponsePartType.Text ? { ...current, text: current.text + item.Value } : { ...current, status: 'completed' }
        : current)
    .on(e => e.streamClosed, current => current.status == 'streaming' ? { ...current, status: 'cancelled' } : current)
    .value;
```
— 4abb7190fd5

**Reducer reuses an existing child model via `find(...) ?? create` — preserves identity/state across re-runs.** `[TS]`
```ts
@reduced
resources = reduce([] as ValidatedResourceModel[], this.events)
    .on(e => e.resourcesValidated.resolved, (resources, { result }) => result.ResourceTreeNodes
        .filter(n => !n.ParentId)
        .map(n => resources.find(r => r.id == n.Id) ?? this.resourceModel(n, result.ResourceTreeNodes)))
    .value;
```
— a7589bba83b

**Per-entity event scope: inject parent events + an id, expose `get`s that filter/map the parent's stream to this id.** `[TS]`
```ts
class ValidatedResourceEvents {
    constructor(private _tree: ValidateResourcesTreeEvents, private _id: Uuid) { }
    get thisResourceValidated() { return this._tree.resourcesValidated.resolved.filterMap(v => v.result.ResourceTreeNodes.find(n => n.Id == this._id)); }
    childEvents(id: Uuid) { return new ValidatedResourceEvents(this._tree, id); }
```
— a7589bba83b

**Non-dep ctor params (raw JSON) are parsed into a private field in the body; `??` picks the populated cast, `!!` normalizes flags.** `[TS]`
```ts
constructor(private _statePocketKey: string, resourceProviderJson: string, private _options: IValidateResourcesTreeOptions) {
    this._resourceProvider = JSON.parse(resourceProviderJson);
}
validationRequest(): server.ValidateResourcesRequest {
    return {
        ResourceIds: (this._resourceProvider as server.DirectValidationResourceProvider).ResourceIds
            ?? (this._resourceProvider as server.PromptOnImportDirectResourceProvider).PromptOnImportResources,
        OnlyReviewableResources: !!this._options.onlyReviewableResources,
```
— a7589bba83b

**React 19: read a context with `use(Context)`, not `useContext(Context)`.** `[TS]`
```tsx
// ❌ import { createContext, useContext } from "react";  … useContext(RichTooltipContext)
// ✅
import { createContext, use } from "react";
let tooltipContext = use(RichTooltipContext);
```
— d2a4aa6fa2d

**React 19: a context is its own provider — `<Ctx value={…}>`, not `<Ctx.Provider value={…}>`.** `[TS]`
```tsx
return <RichTooltipContext value={richTooltipController}>
    {model.groups.map(group => <RibbonSection key={group.caption} caption={group.caption} items={group.items} />)}
</RichTooltipContext>
```
— 676bd21fc11

---

## TypeScript types & narrowing

**Replace a TS `enum` with an `as const` array → a union (`typeof arr[number]`) plus a runtime `Record<U,U>` constant; stays iterable.** `[TS]`
```tsx
export const heartbeatResults = ['Ok', 'SessionExpired', 'Error'] as const;
export type HeartbeatResult = typeof heartbeatResults[number];
export const HeartbeatResult = heartbeatResults.toObject(v => v) as Record<HeartbeatResult, HeartbeatResult>;
```
— 2c71723fec3

**User-defined type guard returning an intersection (`x is A & B` via `'prop' in x`) narrows with no cast.** `[TS]`
```tsx
function isFeatureSwitchedHeading(heading: TOCTreeNode): heading is TOCTreeNode & TOCItem {
    return 'features' in heading;
}
// caller: if (isFeatureSwitchedHeading(heading) && heading.features.length) { … heading.features now typed … }
```
— a36de104817

**`} satisfies IMenuItem)` validates an inline object literal without widening it or hoisting a typed local.** `[TS]`
```tsx
.map(rt => ({
    text: UserControlText.Label_ConnectToMyData,
    icon: 'connection' as IconName,
    action: () => { onSelect(rt); }
} satisfies IMenuItem))
```
— d80c839953d
