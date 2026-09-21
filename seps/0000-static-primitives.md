# SEP-0000: Static Primitives Extension for MCP Server Cards

- **Status**: Draft
- **Type**: Extensions Track
- **Created**: 2026-09-21
- **Author(s)**: Jin-Dong Kim (@jdkim)
- **Sponsor**: None (seeking sponsor)
- **Extension Identifier**: `io.modelcontextprotocol/static-primitives`
- **Working Group**: Server Card Working Group (associated)
- **Depends on**: [SEP-2127 (MCP Server Cards)](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2127), [SEP-2133 (Extensions)](./2133-extensions.md)

## Abstract

This SEP proposes a follow-on extension to [SEP-2127 (MCP Server Cards)](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2127). The extension lets a server include an optional inline list of its primitives — tools, prompts, and resources — inside its Server Card, for servers whose primitive set is the same for every caller. This lets consumers that read the Server Card before connecting (registries, developer portals, agent orchestrators, browser-embedded chat widgets) know what the server offers without opening a live MCP connection and without needing a caller identity. Resource entries carry three optional fields (`sizeBytes`, `volatility`, and `autoAttach`) that let clients make size-budget and freshness decisions before any content is transferred. A reference implementation is deployed at `pubdictionaries.org`, with a browser-widget client that reads and uses the extension in production.

## Motivation

SEP-2127's §"Why Exclude Primitives?" deliberately defers primitive advertisement to a follow-on SEP. The reason given: MCP servers are "inherently dynamic" and a static document cannot reliably represent primitives that vary by user, session, or configuration. The same rationale names the next step directly: _"A follow-on SEP should address the prerequisites, such as variant enumeration and clear consumer contracts, before primitive advertisement is added."_ This SEP is that follow-on.

The dynamism argument is a real constraint, but it does not apply to every server. For a large class of MCP servers — public biocuration APIs, documentation servers, static tool catalogs, most anonymous read-only services — the primitive set does not vary per caller. Every consumer sees the same tools, prompts, and resources. For that class of server, primitive advertisement is not only safe; it is what registries and orchestrators need in order to work.

Specific problems documented in [ext-server-card #30](https://github.com/modelcontextprotocol/ext-server-card/issues/30) and reported by production operators:

- **Registries** cannot describe what a server does without connecting to it under some identity, calling `tools/list`, and hoping the list they collect matches what real users see.
- **Browser-embedded chat widgets** that start when a public page loads do not have time for a pre-render connection and authentication; they need to know the primitive set from a static document.
- **Agent orchestrators** that plan tool routing across many servers face the same problem at scale: every server whose catalog cannot be published statically becomes a runtime discovery cost.
- **Gateway operators** report that consumers build unofficial catalogs anyway, collected under whatever identity happened to be available — and those catalogs are _less_ accurate than one the server's own publisher would produce.

**How to think about this extension:** it is not a replacement for runtime primitive access. It is an _optimization_. When the declared list is small enough to be useful before connecting, clients save a round trip and the model has the information from the first turn. When the declared list is too large, the client falls back to the runtime path, which is unchanged. The extension never removes any capability — it just adds a cheap way to skip a step that already existed. The reference implementation shows this working: the production catalog (about 30 KB) is skipped by the client because its declared size exceeds the client's budget, and the model falls back to the pre-existing `list_dictionaries` tool with no loss of function.

## Specification

### Extension Identifier

The extension identifier is `io.modelcontextprotocol/static-primitives`, using the reserved `io.modelcontextprotocol/` prefix for Official Extensions per SEP-2133.

### Placement

The extension is carried in the Server Card `_meta` object, under its extension identifier:

```json
{
  "$schema": "https://static.modelcontextprotocol.io/schemas/v1/server-card.schema.json",
  "name": "org.pubannotation/pubdictionaries",
  "version": "1.2.3",
  "description": "Biocuration dictionary lookup and annotation",
  "remotes": [
    { "type": "streamable-http", "url": "https://pubdictionaries.org/mcp" }
  ],
  "_meta": {
    "io.modelcontextprotocol/static-primitives": {
      "tools": [/* ... */],
      "prompts": [/* ... */],
      "resources": [/* ... */]
    }
  }
}
```

When the top-level `extensions` field described in SEP-2133 becomes available on Server Card, this extension SHOULD migrate to `extensions["io.modelcontextprotocol/static-primitives"]`. `_meta` remains a valid placement for backward compatibility.

Implementations MAY also include the per-entry fields (see Resources below) on the runtime `resources/list` response, under the same `_meta` key. This gives a consistent shape to clients that reach the server through runtime discovery rather than through a Server Card. The reference implementation does this today, because Server Card discovery is still being finalized.

### Semantic Contract

For each primitive-type array (`tools`, `prompts`, `resources`) that a server includes in the extension, the declared entries MUST be the same entries the server's `tools/list`, `prompts/list`, or `resources/list` operation returns for **any** caller — including an unauthenticated one. If the server's primitive set for a given type varies by caller identity, session, feature flag, or other runtime state, the server MUST NOT include that primitive-type array in the extension.

A server MAY declare only a subset of the three primitive types. For example, a server whose tools are the same for every caller but whose resources vary per caller may include `tools` in the extension and omit `resources`.

If the extension is not present on a Server Card, this carries no negative implication about the server. A server that does not declare the extension may have static primitives, dynamic primitives, or no primitives at all; clients MUST fall back to the standard runtime discovery operations in that case. A way for a server to positively signal _"I have primitives, but they cannot be declared statically"_ is left to future work.

### Tools

Each entry in the `tools` array is an MCP `Tool` object using the wire format defined in the current MCP specification: `name`, `description`, `inputSchema`, and optionally `outputSchema`, `annotations`, `title`. The declared list MUST equal what `tools/list` returns.

### Prompts

Each entry in the `prompts` array is an MCP `Prompt` object using the wire format from the current MCP specification: `name`, `description`, and `arguments` (each with `name`, `description`, `required`). The declared list MUST equal what `prompts/list` returns.

The extension advertises the prompt _metadata_ (what `prompts/list` returns), not the filled-in `messages[]` payload (what `prompts/get` returns). Clients still call `prompts/get` with the user's arguments to get the ready-to-send `messages[]`.

### Resources

Each entry in the `resources` array is an MCP `Resource` object (`uri`, `name`, `mimeType`, `description`) with three additional optional fields:

```typescript
interface StaticPrimitiveResource extends Resource {
  /**
   * The exact byte count of the payload that resources/read would return.
   * When present, this value MUST equal the byte length of the read
   * response body. A server that cannot compute the size cheaply SHOULD
   * omit the field rather than guess.
   */
  sizeBytes?: number;

  /**
   * How the resource's content changes across turns of a client's session.
   * - "stable":   the content does not change between turns; a client MAY
   *               read the resource once and re-use the cached payload on
   *               later turns.
   * - "volatile": the content may change between turns; a client that
   *               attaches this resource SHOULD re-read it from the server
   *               each time it attaches, rather than use a cached copy.
   * Default when absent: "stable".
   */
  volatility?: "stable" | "volatile";

  /**
   * A hint to the client about whether to attach this resource without an
   * explicit user action. Default when absent: true.
   */
  autoAttach?: boolean;
}
```

**Notes on the meaning of each field:**

- `sizeBytes` is exact when it is present, so that clients can make budget decisions without guessing. Servers that cannot compute the exact size cheaply omit the field, and clients fall back to trimming or capping the payload after they fetch it.
- `volatility` controls _when the client re-reads the resource from the server_. It does not control whether the resource is included in a given turn's context. A `"stable"` resource MAY be read once and the payload kept in a cache for later turns. A `"volatile"` resource SHOULD NOT be read when the client starts (a cached copy from start-up is likely already out of date by the time the first turn is sent) and SHOULD be re-read from the server before each turn that includes it. Whether the payload (fetched or cached) is _added to a given turn's system prompt_ is a separate decision the client makes based on its own budget.
- **Whether to include the payload in each turn is a separate decision from volatility.** A client that has cached a stable resource, and has budget for it, SHOULD include the payload in every turn's context — not only in the first turn. A client that reads a stable resource once and then includes it only in turn 1 will find the model spending later turns rediscovering the same resource through runtime tools, since the model does not see the cached payload if the client did not attach it to that turn. A server that declares `volatility: "stable"` should expect clients to keep the payload available on every turn, not to include it only once.
- For `"volatile"` resources, clients SHOULD re-check the budget on every re-read, because a declared `sizeBytes` may have changed between turns.
- `autoAttach` is decided by two independent sources: the server's declaration (this field), and the client's own budget or policy. A client MAY choose not to include a resource even when the server declared `autoAttach: true`, if the resource exceeds a local size or count budget. A client SHOULD NOT auto-attach a resource for which the server declared `autoAttach: false`.

### Migration to Server Card `extensions` field

When the `extensions` field described in SEP-2133 becomes available on Server Card (a map from extension identifiers to per-extension settings), this extension SHOULD move from `_meta["io.modelcontextprotocol/static-primitives"]` to `extensions["io.modelcontextprotocol/static-primitives"]` with the same payload shape. The `_meta` placement remains valid indefinitely, but `extensions` is preferred where it is available: a client must recognise the extension identifier in order to see the payload at all, which is safer for behaviour that depends on the client being extension-aware (see Security Implications).

## Rationale

### Why this scope, and why now

SEP-2127 defers primitive advertisement until a follow-on can address "variant enumeration and clear consumer contracts." This SEP does not solve variant enumeration in general. Instead, it identifies the sub-class of servers where variant enumeration is trivial — the primitive set does not vary — and specifies a consumer contract for that sub-class only. Narrowing the scope this way is deliberate: it lets us release a usable extension now, without having to wait for the harder general case to be worked out.

### Why three primitive types in v1

The reference implementation exercises all three. Limiting the extension to tools would be a narrowing without a good reason: the same contract ("the extension advertises what `<primitive>/list` returns") applies to tools, prompts, and resources in the same way. The community requests documented on [ext-server-card #30](https://github.com/modelcontextprotocol/ext-server-card/issues/30) also cover all three.

### Why "static" in the name

The name deliberately echoes SEP-2127's own rationale text ("primitives are inherently dynamic"). Naming the extension after the sub-class it addresses ("static primitives") is more direct than choosing a neutral term. Reviewers of this SEP will see the name as a direct answer to the objection in SEP-2127, not as an attempt to avoid it.

### Why URI-only for resources (Option X), not content-inlined (Option Y)

Two placements were considered for the resource payload:

- **X.** Advertise the URI and metadata only; content is still fetched using `resources/read`.
- **Y.** Include the content directly in the extension.

Option Y can save more round trips, but it has no upper limit on cost: a client fetching a Server Card cannot know how large that Server Card will be, and a server's declared resource payloads could make it grow to any size. Option X has a bounded cost — the extension itself only carries metadata — and it delays content transfer to the point where the client actually uses the content, at which point the client can apply its own budget. The reference implementation confirms Option X works well: the production catalog's declared size (about 30 KB) is nearly four times the client's 8,000-byte budget, so the client skips including it. If the bytes had been in the Server Card itself, the client would have downloaded them just to make that decision.

### Why `sizeBytes` is exact when it is present

Approximate size hints make budget decisions unclear: a resource declared "about 8 KB" is not obviously affordable against an 8 KB budget. An exact `sizeBytes` gives clients a value they can act on directly. The cost is real: a server that serves `resources/list` from a cached, static hash must serialise the payload to compute an exact byte count. The reference implementation does this and confirms the cost is manageable in practice. Servers for which exact computation is expensive SHOULD omit the field rather than guess. Clients treat an absent `sizeBytes` as "unknown" and apply their own trim after fetching.

### Why two separate fields (`volatility` and `autoAttach`)

The two fields answer two independent questions the client has to decide separately:

- `volatility` decides whether the client should re-read the resource from the server between turns.
- `autoAttach` decides whether the client should include the resource in a turn's context without an explicit user action.

Combining them into a single field would force the server to say something about both dimensions at the same time, even when only one is relevant. It would also make it hard to express `autoAttach` cleanly. `autoAttach` in practice has two inputs — the server's declaration (this field) and the client's own budget — and any combined field could only ever express the server's half of that decision, leaving the client's half implicit.

The wording is deliberate on a related point: neither field carries a turn count. Turn counts are a client-side matter (they depend on the client's budget and session state); the server describes only a property of the resource. In particular, the Notes above say explicitly that a stable resource that fits within budget SHOULD be included in every turn's context, not only in the first turn — so that implementers do not read `volatility: "stable"` as "attach on turn one and then never again."

### Why `_meta` in v1, `extensions` later

The `extensions` field described in SEP-2133 does not yet exist on Server Card. `_meta` is the correct extension slot today. When `extensions` becomes available, this extension SHOULD move there. The distinction matters for declarations that touch security:

- Under `_meta`, a client that does not recognise the extension identifier simply ignores the vendor-namespaced payload — no error, no warning. This is "fail-open": ignoring the extension is safe as long as ignoring it does not have consequences.
- Under `extensions`, a client is expected to recognise the extension identifier in order to see the payload at all. This is "fail-closed": a client that does not know about the extension does not see its declarations, which is safer when disregarding them could cause harm.

The fields in this v1 are optimization hints, so fail-open under `_meta` is safe. Future extensions whose declarations matter for security should target `extensions` from the start.

## Backward Compatibility

- **Fully additive.** Servers that do not implement the extension continue to work with all MCP clients. Clients that do not implement the extension treat `_meta` entries as unknown data and fall back to runtime `tools/list` / `prompts/list` / `resources/list` as they did before.
- **Client behaviour when the extension fields are absent.** The defaults for `volatility` (`"stable"`) and `autoAttach` (`true`) are chosen to match how clients unaware of the extension already treat static reference resources: fetch the resource once, cache the payload, and include it in each turn's context. A server that sends only the plain MCP `Resource` shape (with none of the extension fields) will get the same client behaviour it got before the extension existed.
- **Compatibility between extension-aware and extension-unaware clients.** A resource declared with `autoAttach: false` will not be auto-attached by extension-aware clients, but it MAY still be auto-attached by extension-unaware clients that reach the resource through runtime `resources/list`. A server for which not attaching a resource is security-critical SHOULD NOT rely on this extension under the `_meta` placement; see Security Implications.

## Security Implications

- **The extension payload does not carry credentials or session-specific data.** By contract, all declared primitives are what any caller — including an unauthenticated one — would see through runtime `*/list`. There is no way for the extension to grant elevated access.
- **Fail-open behaviour under `_meta`.** A client that does not recognise the extension identifier will not act on `autoAttach: false` or on the other declarations. This is acceptable when the fields are only optimization hints (which is the case for every field in v1) — the fallback is the behaviour clients had before the extension existed. If a future field's meaning is that ignoring it could have security or privacy consequences, that field SHOULD live under the `extensions` slot (once available) rather than under `_meta`.
- **`sizeBytes` disclosure is negligible.** The field reveals the byte count of a resource whose URI and MIME type are already public through `resources/list`. An attacker who fetches the resource learns the same information.
- **Cache freshness.** A client that respects `volatility: "stable"` may serve cached content that has become stale if the server has changed the resource. This is a freshness question, not a security one; the client MAY subscribe through MCP's own `resources/subscribe` to receive change notifications, which this extension does not affect.

## Reference Implementation

Live in production:

- **Server**: `pubdictionaries.org` (repository: `pubdictionaries` on `master`, extension implemented in `app/controllers/mcp_controller.rb`) publishes six MCP tools, one prompt (`annotate`), and one resource (`pubdictionaries://dictionaries`) under `_meta["io.modelcontextprotocol/static-primitives"]` on its runtime `resources/list` response. Verified live: `curl -X POST https://pubdictionaries.org/mcp -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" -d '{"jsonrpc":"2.0","id":1,"method":"resources/list","params":{}}'` returns the dictionary catalog entry with `sizeBytes`, `volatility: "stable"`, and `autoAttach: true` under the extension key.
- **Client**: `llm_meta_widget` 0.4.0, published on RubyGems and running in the annotation-page widget at `pubdictionaries.org`. Consumes all three primitive types; makes budget decisions from `sizeBytes` before payload transfer; honours `volatility` and `autoAttach`; includes stable resources within budget on every turn.
- **Verification**: the server RSpec test suite, the widget's test suite, and a browser end-to-end run against the published gem. The end-to-end test reads the actual outgoing `fetch` request bodies from the browser, rather than checking what the model wrote in its reply. The evidence is therefore the actual request payload, not the model's own report of what it did.
- **Production evidence for the size gate**: development declares roughly 1.3 KB for the dictionary catalog and includes it in every turn's context; production declares roughly 30 KB against the client's 8,000-byte budget and skips inclusion, falling back to the pre-existing `list_dictionaries` tool. Same code path, opposite behaviour, decided by a single declared number.

## Working Group and Maintainers

- **Working Group**: Server Card Working Group (see [Discussion #2563](https://github.com/modelcontextprotocol/modelcontextprotocol/discussions/2563)). This extension is scoped as a follow-on to SEP-2127 and is proposed for association with the existing WG rather than a new one.
- **Extension Repository**: To be created as `experimental-ext-static-primitives` under the incubation route per SEP-2133. On acceptance, the Working Group will decide the final location — either within the existing `ext-server-card` repository (as a follow-on to Server Cards) or in a dedicated `ext-static-primitives` repository.
- **Author**: Jin-Dong Kim (@jdkim).

## Future Work

- **A signal for servers that have primitives but cannot declare them statically.** Some registries want a way for such servers to say "I have primitives, but you must fetch them at runtime," instead of being silent. A future revision or a companion SEP could add a `dynamic: true` marker, either at the extension level or per primitive type.
- **Inline content for small resources, with a size cap.** Option Y is still attractive for small resources (short lookup tables, small prompt examples). A future revision could allow inline content when `sizeBytes` is below a declared cap.
- **Alignment with SEP-2934's DNS-based discovery.** If DNS-based discovery is finalized, the extension should describe how a Server Card found through DNS carries its `static-primitives` payload.
- **Guidance on prompt arguments in embedded contexts.** Experience with the reference implementation gave a general observation about `prompts/get` in embedded chat widgets: an in-page assistant exists to help people who do not know the interface, so declaring any prompt argument as _required_ creates a paradox — users who could fill the argument in do not need the assistant, and users who cannot are refused. The valuable part of `prompts/get` in these contexts is the server-authored _instruction to the model_ (numbered steps, references to tool names, the shape of the task), which is closer to a fragment of a system prompt than to a template with slots for user input. Prompt authors targeting embedded contexts SHOULD treat all arguments as optional hints, and place the task itself in the prompt payload. This is guidance for prompt authors, not a schema change; a separate best-practice document could capture it.
- **Extension negotiation for fields that touch security.** If a future field's meaning is that ignoring it could have security consequences, that field should target the `extensions` slot on Server Card (fail-closed), not `_meta` (fail-open).

## References

- [SEP-2127: MCP Server Cards](https://github.com/modelcontextprotocol/modelcontextprotocol/pull/2127)
- [SEP-2133: Extensions](./2133-extensions.md)
- [ext-server-card #30: Add optional tool metadata to Server Card for offline discovery](https://github.com/modelcontextprotocol/ext-server-card/issues/30)
- [MCP Specification](https://modelcontextprotocol.io/specification)
- [RFC 8615: Well-Known URIs](https://datatracker.ietf.org/doc/html/rfc8615)
- Reference implementation: `pubdictionaries` server (branch `master`) and `llm_meta_widget` 0.4.0 on RubyGems
