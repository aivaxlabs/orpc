# Changelog

All notable changes to ORPC are documented in this file.

## ORPC/1 Draft 2 — 2026-09-19

### Changed

- Requests now support multipart delivery using `<part> <final>`.
- Request frames now use:
  `<length> ORPC/1 REQ<id> <method> <part> <final>`.
- Response frames now use:
  `<length> ORPC/1 RES<id> <part> <final>`.
- Removed `<execution-id>` from response frames.
- Correlation and operation state are now concentrated in the shared `<id>`.
- New operations and retries must use a base ID that is not already active; fresh collision-resistant IDs are recommended.
- Whole-transfer recovery always starts a new operation with a fresh base ID.
- Request and response reconstruction now use the same part/final semantics.

### Added

- Reserved identifier grammar:
  - `base-id = [0-9a-zA-Z_.@]+`
  - `control = #[0-9A-Z]+`
  - `id = <base-id> | <base-id><control> | <control>`
- Operation-scoped reserved controls:
  - `<base-id>#CANCEL`
  - `<base-id>#CANCELACK`
  - `<base-id>#CHECKSEND`
  - `<base-id>#CHECKOK`
  - `<base-id>#CHECKFAIL`
  - `<base-id>#RESEND`
  - `<base-id>#LOCKED`
- Global reserved controls:
  - `#PING`
  - `#PONG`
  - `#EXIT`
  - `#BYE`
- Whole-content integrity verification through `#CHECKSEND`.
- Multiple integrity hashes in the form `<algorithm>:<hash>;<algorithm>:<hash>;...`.
- Whole-operation cancellation with acknowledgement.
- Whole-content resend requests.
- Active-ID collision handling through `#LOCKED`.
- Heartbeat support through `#PING/#PONG`.
- Graceful transport shutdown through `#EXIT/#BYE`.

### Semantics

- `#CHECKSEND` hashes the complete reconstructed body only:
  `body_part_1 || body_part_2 || ... || body_part_N`.
- ORPC headers, length prefixes, part metadata, and the `#CHECKSEND` frame itself are excluded from the integrity hash.
- `#CHECKFAIL` invalidates the affected transfer and requires recovery with a fresh base ID.
- `#RESEND` requests the complete logical content again; selective part retransmission is not defined.
- A request received on a base ID that is still active is discarded and answered with `#LOCKED`.
- Reserved-control requests use method `0`, part `1`, and final `1`.
- Application-generated request IDs may contain only the base-ID character set and may not contain `#`.

## ORPC/1 Draft 1

Initial experimental specification.

- Length-prefixed binary framing with ASCII headers.
- Single-frame requests.
- Multipart responses.
- Request IDs plus server-generated execution IDs.
- Ordered reconstruction.
- Whole-request recovery.
- Bidirectional acknowledged application calls.
