# ORPC — Ordered Remote Procedure Call

A stateless, binary protocol for exchanging potentially large responses over a shared channel, with ASCII headers and byte-length framing.

ORPC splits a response into numbered parts, lets unrelated calls share the channel, reconstructs each response in order, and requires a new request when delivery is incomplete. The application decides what the content means; ORPC does not distinguish JSON, plain text, raw files, or application errors.

**Status: experimental — ORPC/1 Draft 1.** This repository currently contains the protocol design and specification, not a client library, server implementation, or production-readiness claim. Draft revisions may change the wire format.

[Read the specification](SPEC.md) · [GitHub repository](https://github.com/aivaxlabs/orpc) · [MIT license](LICENSE)

## Contents

- [Why ORPC exists](#why-orpc-exists)
- [How it works](#how-it-works)
- [What ORPC guarantees](#what-orpc-guarantees)
- [What the transport must provide](#what-the-transport-must-provide)
- [Notifications and bidirectional calls](#notifications-and-bidirectional-calls)
- [How it compares](#how-it-compares)
- [When to use it](#when-to-use-it)
- [Current limits and tradeoffs](#current-limits-and-tradeoffs)
- [Implementing ORPC](#implementing-orpc)
- [Contributing](#contributing)
- [License](#license)

## Why ORPC exists

ORPC grew out of remote access work on Avi Desktop and Avi Workspace. A conversation-context response could contain several megabytes of text and exceed the remote path's per-message limit. A connection could open successfully and answer a small discovery request, then close while delivering a larger response.

The problem was not a size limit in JSON-RPC itself. It was sending an entire application result as one message through a bounded transport. Raising that bound can accommodate today's response while leaving the same failure mode for the next larger one.

ORPC makes response segmentation part of the protocol. Each part is a complete transport message, so senders can alternate between a large response, a small response, and an event call. Receivers know which parts belong together and when the result is complete. If a part never arrives, the client requests the resource again instead of accepting truncated content.

This is a design for large **finite responses**, not an unlimited stream or a replacement for every RPC system. It keeps framing compact and leaves application data models outside the protocol.

## How it works

Every frame is `<length> SP <payload>`, where the payload is an ASCII header, one LF byte, and arbitrary binary content. The decimal length counts every payload byte, including the header and LF, but excludes its own digits and separating space. ORPC defines its own boundaries; no escaping or separator after the body is needed.

For example, `33 ORPC/1 REQ125 archive.read\nfoobar` has a 33-byte payload. `31 ORPC/1 RES125 exec-A 1 1\nfoobar` has a 31-byte payload. Here `\n` denotes the actual LF byte, not two literal characters. In the frame blocks below, the Markdown line break before a closing fence is not part of the transmitted body.

A request names a method and carries its application input:

```text
33 ORPC/1 REQ125 archive.read
foobar
```

The server returns numbered response parts. Here, the second part begins with a space that belongs to the content:

```text
30 ORPC/1 RES125 exec-A 1 0
Hello
```

```text
31 ORPC/1 RES125 exec-A 2 1
 world
```

The receiver returns exactly `Hello world` after it has both parts.

The response header is:

```text
ORPC/1 RES<request-id> <execution-id> <part> <final>
```

- **Request ID** correlates a response with one request attempt.
- **Execution ID** separates different server executions of the same request, including executions caused by duplicate delivery.
- **Part** starts at 1 and specifies reconstruction order, regardless of arrival order.
- **Final** is `1` for the last part and `0` otherwise.

Receiving the final part does not mean receiving the complete response. If part 2 arrives before part 1, it waits. Parts from other request IDs may arrive between them without being mixed into this result.

The receiver selects the execution identified by its first valid response frame and ignores other executions for that attempt. This prevents constructing a result from part 1 of one execution and part 2 of another.

Only the framing and header are ASCII. Bodies may contain any bytes, including NUL or invalid UTF-8. Text examples here use UTF-8 bodies for readability, not as a protocol requirement. Raw files need no Base64, and ORPC adds no `encoding` or `content-type` field: each method defines its content contract.

Parts may split a text character across byte boundaries. Applications concatenate bytes before decoding text, or use a decoder that retains state across parts. See the [binary example](SPEC.md#144-binary-application-data).

The short IDs above are illustrative. Real deployments should use collision-resistant identifiers as described in [the specification](SPEC.md#7-correlation-and-multiplexing).

## What ORPC guarantees

### Ordering within one response

Parts are concatenated in numerical order, with no added separators or modified content. Identical duplicate parts have no effect; conflicting parts invalidate the attempt.

“Ordered” does **not** mean that separate calls execute in order. Sending an update before a read does not guarantee that the read sees the update. Applications coordinate dependencies between calls.

### Complete delivery or explicit failure

The receiver completes a call only after receiving every part through the final part. A finite deadline detects missing content, including a lost final marker or a request that never received any response.

On incomplete delivery, the caller reissues the same method and application content using a fresh request ID. It discards the old attempt and never combines content across retries. Backoff, retry budgets, and overall deadlines bound recovery. If recovery cannot finish, the caller receives a failure—not a successful partial result.

This is **not exactly-once execution**. A lost response can cause the same operation to execute again. Exposed methods must be safe to repeat: read-only, idempotent, or protected by application-level idempotency handling.

### No persistent protocol session

Requests are self-contained. ORPC does not require a server-side transfer cache, resume token, session handshake, or replay history. Another server instance can handle a retry.

Active calls still need temporary execution and reconstruction state. Stateless does not mean zero memory, and it does not prohibit application state such as a database or an idempotency record.

## What the transport must provide

**ORPC requires a reliable ordered byte stream, or a transport delivering complete frames with integrity.** Its length prefix supplies framing on TCP, STDIO pipes, and stream-oriented Unix sockets. A parser validates and bounds the prefix before allocation, accumulates exactly the declared payload bytes, and repeats. Reads may split a frame or contain several frames.

WebSocket and message-queue bindings deliver one complete prefixed frame per message and check that its declared length matches the delivered payload. Direct WebSocket bindings use binary messages. Text-only transports require an agreed lossless adapter, such as Base64 encoding of the entire frame, decoded before ORPC parsing; this is not automatic content conversion. Complete frames may arrive out of order; ORPC does not reorder bytes or fragments inside a frame. UDP needs a complete frame per datagram within supported limits or an external adaptation layer; IP alone does not guarantee delivery.

For STDIO, stdout carries only protocol frames and stderr carries logs. Writers serialize entire frames, handle short writes, and respect backpressure. Readers must not apply newline translation; EOF inside a prefix or payload is incomplete delivery.

Bindings define authentication, routing, integrity, backpressure, and accepted frame sizes. ORPC itself supplies neither encryption nor authorization.

Native WebSocket fragmentation is a different mechanism: frames are reconstructed into a WebSocket message before delivery to the application. ORPC makes each part a separate message so responses can be interleaved at that level. This permits multiplexing; fair queueing and backpressure are still necessary for small calls to remain responsive.

See [the required transport contract](SPEC.md#3-required-transport-contract) for normative requirements.

## Notifications and bidirectional calls

Either peer can initiate a call. An event notification is an ordinary request in the opposite direction, with a short response acknowledging it:

```text
93 ORPC/1 REQevt-123 conversation.changed
{"eventId":"change-456","conversationId":"thread-789"}
```

```text
31 ORPC/1 RESevt-123 exec-C 1 1
OK
```

`OK` is an application convention, not a special ORPC status. An empty final response can also serve as acknowledgment. The application defines whether acknowledgment means acceptance, durable storage, or completed processing; acceptance is the recommended default, not a guarantee of durable delivery.

If the acknowledgment is lost, the event is requested again. Event handlers must tolerate repetition or deduplicate using a stable application event ID. The response itself is not acknowledged, avoiding an acknowledgment loop.

See [acknowledged notifications](SPEC.md#71-acknowledged-notifications).

## How it compares

These protocols solve overlapping but different problems. ORPC is a narrow experimental contract, not a claim that established alternatives cannot carry large data.

| Protocol | Core approach | Difference from ORPC |
| --- | --- | --- |
| **JSON-RPC 2.0** | Transport-independent JSON requests, results, errors, notifications, and batches. | It does not standardize segmented response reconstruction. An application can add streaming or pagination conventions; ORPC makes numbered parts and whole-request recovery part of its core contract, without prescribing JSON content. |
| **gRPC** | Typically Protocol Buffers with an HTTP/2-based transport; supports unary and streaming calls, status, deadlines, and flow control. | It already addresses streaming through a mature framework. ORPC uses length-prefixed binary payloads with ASCII headers over byte streams or complete-frame bindings and leaves schemas and application status opaque. It currently offers no comparable implementation ecosystem. |
| **SOAP** | XML envelopes with messaging semantics, faults, and transport bindings; can be paired with other specifications. | ORPC does not define an application envelope, fault model, service-description language, or XML processing rules. SOAP-related extensions may address reliability and attachments outside the base messaging framework. |
| **WebSocket** | A full-duplex transport with message boundaries and lower-level frame fragmentation. | It is a possible carrier, not an RPC contract. ORPC adds method calls, response correlation, ordered part reconstruction, and recovery rules above it. |

ORPC is not wire-compatible with JSON-RPC, gRPC, or SOAP. Its `OK` acknowledgment is not a standardized status equivalent to a gRPC status or SOAP fault.

Background references: [JSON-RPC 2.0 specification](https://www.jsonrpc.org/specification), [gRPC core concepts](https://grpc.io/docs/what-is-grpc/core-concepts/), [SOAP 1.2 messaging framework](https://www.w3.org/TR/soap12-part1/), and [WebSocket RFC 6455](https://www.rfc-editor.org/rfc/rfc6455.html).

## When to use it

Consider ORPC when you control both peers and need to explore:

- Large finite results, including raw files, crossing a frame-size-limited path.
- Several concurrent calls sharing a channel without serializing entire responses behind one another.
- Bidirectional event delivery with application acknowledgment.
- Response reconstruction independent of transport delivery order.
- Whole-request recovery without storing resumable transfer sessions on the server.

Prefer an existing protocol or a dedicated transfer mechanism when you need production SDKs, typed service generation, resumable uploads, or an established interoperability ecosystem. ORPC is still a draft and has no implementation in this repository.

## Current limits and tradeoffs

- **Only responses are segmented.** A request, including a notification, must fit in one bounded frame and any transport message-size limit. Draft 1 does not solve segmented uploads by itself.
- **Content is opaque binary.** Applications define text encodings, schemas, and file formats. Text-only transport adapters may add Base64 overhead, but byte-preserving bindings need no such conversion.
- **Recovery repeats the entire operation and transfer.** It does not request just the missing parts. A changing resource may return different content unless the application pins a version or snapshot.
- **No exactly-once execution.** Repeat safety is an application requirement, including for acknowledged events.
- **No fixed total-response limit does not mean unlimited resources.** Implementations must bound accepted response sizes, concurrent calls, queues, storage, retries, and duration.
- **No wire-level `ABORT` or `CANCEL`.** The current draft relies on completion deadlines and local cancellation behavior. It does not require an abandoned server execution to stop immediately.
- **No universal part size or latency guarantee.** Implementations choose frame sizes and scheduling appropriate to their transport and workload.
- **No promise of eventual delivery during permanent failure.** Recovery ends with an explicit failure when its policy is exhausted.

## Implementing ORPC

There is no package to install yet. [SPEC.md](SPEC.md) is the normative source; this README is an explanatory overview.

An implementation needs a transport binding, a byte-oriented frame parser and ASCII header encoder, a method dispatcher, a paced response segmenter, an execution-isolated receiver, and bounded recovery logic. Application handlers also need repeat-safety and acknowledgment contracts before automatic recovery is enabled.

Use the [conformance requirements](SPEC.md#15-conformance-requirements) as the starting acceptance checklist. Test reordered parts, missing final markers, duplicate requests producing different executions, lost event acknowledgments, and small calls competing with large responses—not only successful in-order delivery.

This is Draft 1, still under development—not a second protocol release. Peers must agree on the same specification snapshot outside the protocol. The experimental `ORPC/1` marker alone does not guarantee compatibility, and earlier unprefixed working examples are not the current wire format.

## Contributing

Protocol feedback and proposed bindings are welcome through [issues](https://github.com/aivaxlabs/orpc/issues) and [pull requests](https://github.com/aivaxlabs/orpc/pulls).

For a specification change, describe the problem, show concrete frames or a failure sequence, explain the effect on statelessness and retry safety, and identify wire-compatibility implications. Distinguish normative requirements from implementation recommendations. Update examples and conformance criteria together.

Do not describe a proposed behavior as implemented or interoperable without evidence. No performance claims or conformance certification are implied by the draft.

## License

This repository is published under the [MIT License](LICENSE).
