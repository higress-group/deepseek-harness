# Agent Note: Preserve non-empty streamed tool-call identities

Status: implemented

English | [中文](2026-08-18-deepseek-tool-call-id-preservation.zh.md)

## Problem

DeepSeek-compatible streaming responses can identify a tool call in its first delta and send later argument deltas with an explicitly empty `tool_calls[].id`. The DeepSeek adapter assigned every defined wire value to the accumulated block, so an empty continuation replaced the usable identity. The emitted tool-call block then carried an empty `CallId`, preventing downstream tool-result correlation and causing a valid streamed continuation to fail with an unresolved tool lineage.

## Decision

The DeepSeek translator updates an accumulated tool-call identity only when the wire value is non-empty. The first non-empty ID therefore survives later argument-only deltas, while a later non-empty wire value remains able to correct or replace the accumulated identity. Argument fragments, function names, finish reasons, usage, and the emitted wire vocabulary remain unchanged. The package's English and Chinese READMEs document this provider behavior beside the other streaming rules.

## Alternatives considered

**Reject a stream after an empty tool-call ID.** Rejected because an empty ID on an argument continuation is valid provider-compatible input when an earlier delta already established the call identity; rejecting it turns recoverable wire variation into a failed request.

**Repair the empty identity downstream in tool assembly or continuation handling.** Rejected because the adapter owns translation from provider wire data and downstream consumers should receive one stable call identity rather than carry provider-specific repair logic.

**Accept only the first non-empty ID and ignore every later non-empty ID.** Rejected because a later non-empty value can be the provider's corrected identity; the minimal invariant is to ignore empty replacements, not to freeze all future non-empty values.

## Consequences

Fragmented DeepSeek tool calls preserve their established identity through empty argument deltas, so tool-result correlation and agent continuations remain usable. Streams that never provide a non-empty ID still emit the existing empty identity and retain the downstream failure behavior for an actually unidentifiable call. No request or response protocol fields change.

## Testing

The adapter translation test replays the live capture pattern with a non-empty first ID followed by empty-ID argument fragments and requires the stable ID on every emitted delta and the final block. The focused `llm-deepseek` Vitest file passes 30 tests, the package TypeScript build passes, and the bilingual README pairing check passes.
