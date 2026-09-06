# Review: Claude content serialization regression

## Finding — shared content type emits fields for the wrong block variant

**Defective file:** `core/relay/model/claude.go`, lines 55–67, specifically the `Text` and `Thinking` JSON tags at lines 57–58. The reviewed tree contains `json:"text"` and `json:"thinking"` rather than their former `omitempty` forms.

Removing `omitempty` globally forces both string fields into every serialized `ClaudeContent`, regardless of its `Type`. This fixes the missing empty `text` field in a streaming text-block start, but also adds an empty `thinking` field to ordinary text blocks and both empty fields to image, tool-use, and tool-result blocks. The shared structure represents multiple block variants and is used for upstream requests as well as downstream streaming responses; a streaming-specific requirement was incorrectly applied to all of them.

For example, a tool-use block constructed with `Type: "tool_use"`, `ID: "tool_1"`, `Name: "lookup"`, and an empty input map now has this JSON shape (key order is immaterial):

```json
{"type":"tool_use","text":"","thinking":"","id":"tool_1","name":"lookup","input":{}}
```

The `text` and `thinking` members are unrelated to the tool-use variant. Likewise, a text block containing `hello` acquires `"thinking":""`. These payloads can fail strict block-type validation upstream or in compatible clients. This is a protocol-compatibility regression, not just larger JSON. Exact provider acceptance and error messages were not tested against a live service.

## Evidence in the reviewed tree

- **Shared wire contract:** `core/relay/model/claude.go:55–67` declares the unconditional fields. `ClaudeMessage.Content` uses this type at lines 74–77, and `ClaudeRequest.System` / `Messages` use it at lines 144–158. There is no `ClaudeContent.MarshalJSON` override to filter fields by block type.
- **Ordinary upstream requests are affected:** `core/relay/adaptor/anthropic/openai.go:204–210` constructs system text blocks; lines 222–231 construct text/tool-result blocks and explicitly clear `Text` for tool results; lines 248–259 construct image blocks; lines 271–280 construct tool-use blocks without setting either string. The new tags defeat that intentional absence of variant-inapplicable fields.
- **Serialization does not sanitize the structure:** `core/relay/adaptor/anthropic/gemini.go:21–43` marshals the converted Claude request with `sonic.Marshal` and places those bytes directly in the upstream request body. The shared model therefore affects real request payloads, not only intermediate data.
- **Why the original fix appeared useful:** `core/relay/adaptor/openai/claude.go:449–456` starts a thinking block with an empty `Thinking`, and lines 480–487 start a text block with an empty `Text`. `core/relay/render/claude.go:37–52` serializes these objects directly. Preserving the appropriate empty field in these start events is necessary, but changing the shared tags also emits the *other* variant's field. Tool-use start events use the same model at `core/relay/adaptor/openai/claude.go:522–527`.
- **Coverage gap:** `core/relay/model/claude_test.go:1–66` tests usage conversion and cache-control behavior, not the serialized shapes of the different `ClaudeContent` variants.

The adaptor and renderer paths above demonstrate the blast radius; they are not independently defective edits in the introducing change. The defective change is confined to the two tags in `core/relay/model/claude.go`.

## Introducing change and later correction

The mirror's work branch starts from a single snapshot commit, so its local history does not itself establish the upstream merge chronology. Read-only GitHub API inspection supplies that evidence:

1. [Upstream PR #604](https://github.com/labring/aiproxy/pull/604), merged June 20, 2026, became [commit `449859f`](https://github.com/labring/aiproxy/commit/449859f6d013ad63f463b8e75eb9d4386246ef47). Its description reports the original OpenCode error, `Invalid input: expected string, received undefined`, at `content_block.text`. Its one-file patch removes `omitempty` from **both** `Text` and `Thinking`. The reviewed snapshot contains those exact changed tags.
2. [Upstream PR #609](https://github.com/labring/aiproxy/pull/609), merged June 29, 2026, became [commit `aebeb5e`](https://github.com/labring/aiproxy/commit/aebeb5ec60415093db37c49dfda3fb6aa7ffd01e). Its description explicitly says `Reverts labring/aiproxy#604`; its one-file patch restores `omitempty` on those same two fields. This confirms that the merged change was subsequently backed out. The PR description does not supply a production error trace, so the concrete failure mechanism above is derived from the current serialization contract and its callers, not attributed to an unreported incident.

Read-only evidence commands used: `gh api repos/labring/aiproxy/pulls/604`, the corresponding `/files` endpoint, and the same two endpoints for PR #609; local numbered source inspection and a `MarshalJSON` symbol search. No fix was implemented and no live provider request was made. Only this review document is changed.
