# ORPC — Ordered Remote Procedure Call

A stateless, binary RPC protocol for exchanging potentially large finite requests and responses over a shared channel, with ASCII headers, byte-length framing, multipart reconstruction, integrity checks, cancellation, recovery controls, and connection heartbeats.

**Status: experimental — ORPC/1 Draft 2.** The repository currently contains the protocol design and specification, not a production-ready client or server implementation. Draft revisions may change the wire format.

[Read the specification](SPEC.md) · [Changelog](CHANGELOG.md) · [GitHub repository](https://github.com/aivaxlabs/orpc) · [MIT license](LICENSE)

## Why ORPC exists

ORPC grew out of remote access work on Avi Desktop and Avi Workspace. Large application results could exceed a relay or transport's per-message limit even though the connection itself was healthy.

ORPC makes segmentation part of the protocol. Both requests and responses may be split into numbered parts, unrelated operations can share the same channel, and a receiver only completes a transfer after every part through the final part has arrived.

The design targets large **finite payloads**, not an unlimited stream. It keeps application data opaque and avoids requiring schemas, generated clients, HTTP/2, persistent transfer sessions, or a server-side replay cache.

This makes ORPC suitable for transports and relays that can carry complete binary messages or byte streams, including WebSocket relays such as Cloudflare Workers.

## Wire format

Every frame is:

~~~text
<length> SP <payload>
<payload> = <header> LF <content>
~~~

The decimal length counts every payload byte, including the header and LF, but excludes the length digits and the separating space.

A request is:

~~~text
<length> ORPC/1 REQ<id> <method> <part> <final>
<content>
~~~

A response is:

~~~text
<length> ORPC/1 RES<id> <part> <final>
<content>
~~~

For example:

~~~text
37 ORPC/1 REQ125 archive.read 1 1
foobar
~~~

A multipart response can be:

~~~text
23 ORPC/1 RES125 1 0
Hello
~~~

~~~text
24 ORPC/1 RES125 2 1
 world
~~~

The reconstructed response is exactly Hello world.

The same part/final mechanism applies to multipart requests. Part numbering starts at 1. final=1 marks the final part number, but receiving the final part does not complete a transfer until all preceding parts are present.

## IDs and multiplexing

Application-generated base IDs use:

~~~text
base-id = [0-9a-zA-Z_.@]+
control = #[0-9A-Z]+

id =
    <base-id>
  | <base-id><control>
  | <control>
~~~

A base-id MUST NOT contain #. Application-generated request IDs MUST use only a base-id. IDs containing a control suffix and IDs consisting only of a control are reserved by ORPC.

One base-id identifies one active logical operation and correlates its request, response, and operation-scoped controls. New operations MUST use an ID that is not currently active. Implementations SHOULD generate fresh, collision-resistant IDs and SHOULD avoid historical ID reuse because delayed frames can otherwise become ambiguous.

If a peer receives a new request on an ID that is still processing or responding, it discards that request and returns #LOCKED.

## Reserved operation controls

Controls scoped to an operation append a reserved suffix to its base-id:

- base-id#CANCEL — cancel the whole active operation. Requires #CANCELACK.
- base-id#CANCELACK — confirms cancellation. No response is required.
- base-id#CHECKSEND — sends one or more integrity hashes for the complete reconstructed content, excluding ORPC headers. The body format is algorithm:hash;algorithm:hash;....
- base-id#CHECKOK — confirms that #CHECKSEND validated. No response is required.
- base-id#CHECKFAIL — reports that #CHECKSEND did not validate. The failed attempt is discarded and recovery uses a fresh base-id.
- base-id#RESEND — requests retransmission of the complete logical content, never selected parts. Recovery uses a fresh base-id.
- base-id#LOCKED — reports that the base-id is still occupied by an active operation. The colliding request is discarded and must be retried with a fresh base-id.

#CHECKSEND hashes the concatenation of the complete body in logical part order:

~~~text
hash(body_part_1 || body_part_2 || ... || body_part_N)
~~~

It does not hash ORPC headers, part metadata, or the #CHECKSEND frame itself.

## Reserved global controls

Global controls use an ID consisting only of a reserved control:

- #PING — heartbeat request. The peer MUST answer with #PONG.
- #PONG — heartbeat response. No response is required.
- #EXIT — request graceful transport shutdown. The initiator waits for #BYE before intentionally closing the transport.
- #BYE — indicates that transport shutdown is imminent.

A REQ using a reserved control ID MUST use method 0, part 1, and final 1.

Example heartbeat:

~~~text
22 ORPC/1 REQ#PING 0 1 1

20 ORPC/1 RES#PONG 1 1
~~~

Reserved control frames are protocol messages, not application methods.

## Cancellation and recovery

Cancellation is operation-wide:

~~~text
27 ORPC/1 REQ125#CANCEL 0 1 1

28 ORPC/1 RES125#CANCELACK 1 1
~~~

Cancellation acknowledgement means the peer accepted the cancellation and will stop delivering further content when possible. It does not imply that every underlying application action was physically interruptible.

ORPC does not selectively request a missing part. #RESEND and integrity failure abandon the affected transfer and recover using a new base-id. A response is never resent as an unrelated orphan RES frame: when a response must be regenerated, the original requester reissues the operation under a new base-id.

ORPC still does not guarantee exactly-once execution. Methods exposed to automatic recovery must be read-only, idempotent, or protected by application-level idempotency.

## Notifications and bidirectional calls

Either peer can initiate a call. An event notification is an ordinary request in the opposite direction and receives an ordinary response:

~~~text
97 ORPC/1 REQevt.123 conversation.changed 1 1
{"eventId":"change-456","conversationId":"thread-789"}
~~~

~~~text
24 ORPC/1 RESevt.123 1 1
OK
~~~

OK is an application convention, not an ORPC status. An empty response is also valid when the application defines it as acknowledgement.

## Transport requirements

ORPC runs over either a reliable ordered byte stream or a binding that delivers complete frames with integrity.

TCP, STDIO pipes, and stream-oriented Unix sockets can carry the length-prefixed frames directly. WebSocket and message-queue bindings carry one complete prefixed ORPC frame per message. Direct WebSocket bindings use binary messages.

Complete-frame bindings may reorder, duplicate, or lose frames. ORPC reconstructs request and response parts by part number, detects incomplete transfers, and provides whole-transfer recovery controls. It does not reorder bytes inside a frame.

Bindings remain responsible for authentication, authorization context, confidentiality, backpressure, routing, frame-size limits, and transport lifecycle.

## Current limits and tradeoffs

ORPC intentionally does not define schemas, application error envelopes, generated clients, exactly-once execution, resumable partial transfers, selective part retransmission, compression negotiation, or flow-control windows.

Checksums provide content-integrity validation according to the selected hash algorithms; they are not authentication unless a cryptographic keyed mechanism is separately defined.

The protocol remains experimental. See [SPEC.md](SPEC.md) for normative requirements and [CHANGELOG.md](CHANGELOG.md) for wire-format changes.

## Contributing

Protocol feedback and proposed bindings are welcome through issues and pull requests. For specification changes, describe the problem, provide concrete frame examples, explain retry and state implications, and identify wire-compatibility effects.

## License

This repository is published under the [MIT License](LICENSE).
