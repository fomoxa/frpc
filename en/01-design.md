# fRPC: the RPC layer for Fomoxa

English · [Tiếng Việt](../vi/01-design.md)

Status: design document. It is neither a specification nor an implementation guide yet.

This document describes an RPC layer placed on top of Fomoxa, at the conceptual level and at the byte level. It is language independent and references no API or source code of any implementation.

The capitalised normative keywords (MUST, MUST NOT, SHOULD, SHOULD NOT, MAY) carry their usual meaning in specification text.

Two prerequisite documents:

| Document | What is used here |
|---|---|
| `specification` RFC-0002 | A Model is a contiguous list of fields, no metadata; §9.1 version skew; Enum is always 4 bytes |
| `implementation-guide` `01-overview.md`, `02-flows.md` | The three layers, the DATA frame, the fingerprint handshake, the tick cycle, the single pending frame slot, event data lifetime |

---

## 0. Scope

fRPC is a separate repository. It is not part of fomoxa-net and is not on the fomoxa-net roadmap. It sits on top of the specification and consumes a net implementation the same way an application consumes it.

This boundary has a test: if fRPC requires a change inside fomoxa-net, the logic is being placed in the wrong layer. `implementation-guide` §11 step 5 states the same condition for transport authors. Everything fRPC needs lives in the payload of the DATA frame, which `02-flows.md` §2.2 defines as opaque bytes to the frame layer.

The goal of fRPC is an RPC layer integrated with the properties Fomoxa already provides. The evaluation criterion for every decision in this document is: which existing property of the specification or of the implementations does this decision exploit.

Other RPC systems are mentioned in §3.4 and §17 to explain why a common design choice does not apply under Fomoxa's constraints.

---

## 1. Position in the architecture

```
   APPLICATION         service implementation · client stub
       │
   ────┼──────────────────────────────────────────────────────
       │
   fRPC                call id · dispatch · deadline
                       interceptor · identity · stream
       │
       ├── calls CODEC       model ↔ bytes         (RFC-0002)
       ├── compression (opt) body only              §11
       │
   ────┼──────────────────────────────────────────────────────
       │
   FOMOXA NET          frame · handshake · heartbeat · tick
       │
   ────┼──────────────────────────────────────────────────────
       │
   TRANSPORT           TCP · UDP · WebSocket · QUIC
```

Dependency direction: fRPC depends on net and on the codec. Net does not depend on fRPC. The codec does not depend on fRPC. The transport depends on no layer above it.

Division of responsibility: net determines which message type a byte sequence belongs to. fRPC determines which call that byte sequence belongs to and which component handles it.

### 1.1 Inherited properties

The table lists the properties fRPC receives from the specification and from net implementations. fRPC creates none of them; the requirement on fRPC is not to lose them.

| Source | Property |
|---|---|
| Per-message fingerprint comparison at field level in the handshake (`02-flows` §3.3) | If the two schemas disagree on their intersection, the connection is rejected with a reason code, at connect time. There is no state in which two sides deploy divergent schemas and then decode different values at runtime |
| Version skew, RFC-0002 §9.1 | Appending a field to a request model is a valid change, and valid independently per method (§3.3). No endpoint versioning is required |
| Deterministic wire format (RFC-0001 §6.1) | Payload signing, idempotency keys derived by hashing content, comparable audit records, replay of a call sequence. These operations require two implementations to produce identical bytes |
| Net's transport boundary (`01-overview` §3) | Runs over every transport net supports, with no dependency on a protocol stack. Adding a transport does not change fRPC |
| Heartbeat and dead-peer detection (`02-flows` §4) | fRPC implements no liveness mechanism. The processing time of a call does not affect dead-peer detection |
| `fomoxa-inspect` | Reads the body of an RPC frame after skipping the 5-byte header and the ctx block when present (§4.4). No separate decoder is needed for the RPC layer |
| `fomoxac` and the existing schema | A method's request, response and item types are models in the same schema already used for plain messages. There is no second interface definition language |
| Time injected from outside (B8) | Deadline, timeout and cancellation cycles run to completion in tests without waiting in real time |

The first two rows follow from two decisions in the specification: *position is the only identifier* and *prefix fingerprint chains*. They are not features of the RPC layer.

The same two decisions produce the limits: no free-form metadata map (§3.4), no optional fields, no removal of a field in the middle.

### 1.2 Operational assumption: long-lived connections

fRPC is designed for long-lived connections:

```
   client  ──────── connection lives for the application session ────> backend
   service ──────── connection lives for the duration of a task ────> service
```

This assumption follows from the first row of §1.1. A schema-verifying handshake means every connection sends a greeting listing the whole schema, 14 bytes per message:

```
   ~1,100 messages  ×  14 B  ≈  15 KB, paid before READY
```

That cost is amortised over the number of calls carried by the connection.

Three consequences:

- Per-call cost takes priority over per-connection cost. The byte trade-offs in this document follow that ordering; for example, dropping `method id` (§3.3) makes the greeting larger and every frame smaller.
- Do not optimise the greeting without measurements. This is an architectural assumption, not a protocol defect. Fingerprint-keyed schema caching remains an open option, to be considered only against data.
- Mass reconnection. The assumption does not hold at deploy time: ten thousand peers reconnecting simultaneously, 15 KB each, produce a 150 MB burst. The reconnecting side MUST spread reconnections with jitter. This is an operational requirement, not a protocol requirement.

---

## 2. Constraints inherited from Fomoxa

Every design decision in the following sections traces back to one row of this table.

| # | Fomoxa constraint | Source | Required consequence for fRPC |
|---|---|---|---|
| C1 | The DATA frame carries one control field: the message id. No flags, no sequence number | `02-flows` §2.2 | All RPC information MUST live in the payload. fRPC defines a fixed header §4 |
| C2 | The payload is opaque to net | `02-flows` §2.2 | fRPC may place its own header there without breaking compatibility |
| C3 | The handshake has already verified that the two schemas agree on every shared message | `02-flows` §3.3 | No type negotiation at runtime. An unknown type is not an fRPC runtime error |
| C4 | The handshake is not a security mechanism | `02-flows` §3.3 | Authentication belongs to fRPC, separate from the handshake §10 |
| C5 | The core holds one pending send slot. Sending again while it is occupied returns a congestion error | `01-overview` §5 | fRPC MUST own its send queues, with ceilings, and MUST surface congestion to the application §7 |
| C6 | No background threads, no async, everything runs inside `tick` | `01-overview` §2, §8 | All processing in the fRPC core runs inside a tick; one call spans many ticks. A handler running on the driving thread MUST NOT block; long work is split across ticks §8, §19 |
| C7 | Event data is valid only until the next tick | `02-flows` §6.2 | fRPC MUST decode or copy within the receiving tick. It MUST NOT retain a borrowed reference in a pending call |
| C8 | Timestamps are passed in; the clock is monotonic | `01-overview` B8, B9 | Deadlines use the tick's `now`. The fRPC core MUST NOT read a clock itself; the shell reads the clock and passes `now` into the core §19 |
| C9 | Fomoxa does not retransmit, reorder or deduplicate | `01-overview` §7 | Neither does fRPC. On packet transports, RPC semantics degrade §5.7 |
| C10 | 16 MiB ceiling per message | `02-flows` §2.7 | A body above the ceiling MUST be split into a stream. fRPC checks before sending rather than letting the transport return ⊘ |
| C11 | Exactly one termination event per session | `01-overview` B6 | All pending calls are cancelled exactly once §5.8 |
| C12 | No metadata on the wire; position is the only identifier | RFC-0001 §4.1 | No per-call key/value metadata map §3.4 |
| C13 | Enum is always 4 bytes, and a value outside the defined set is an invalid byte stream | RFC-0002 §7 | Error codes are declared `UInt32`, not Enum, so the code space stays extensible §4.6 |
| C14 | Every table MUST have a ceiling, and the ceiling MUST be a concrete number | `02-flows` §8 | fRPC publishes its ceiling table §4.8 |

---

## 3. Conceptual model

### 3.1 Vocabulary

Each concept has exactly one name, following the convention of `02-flows.md`:

```
call        one invocation with its own lifecycle, identified by a call id
method      a callable procedure, identified on the wire by the message id
            of its request model §4.4
kind        the role of a frame within a call: REQUEST, RESPONSE,
            ERROR, ITEM, END, CANCEL, CREDIT
body        one encoded Fomoxa model, or empty
status      the result code of a completed call
deadline    the point in time after which the caller cancels the call
pending     the set of unfinished calls of a session
dispatcher  table from request message id to handler
```

Service does not appear on the wire, and a method has no identifier of its own. See §3.3.

### 3.2 Four call shapes

```
   UNARY              REQUEST ──────────────>
                      <────────────── RESPONSE | ERROR

   SERVER STREAM      REQUEST ──────────────>
                      <────────────────── ITEM
                      <────────────────── ITEM
                      <─────────────── END | ERROR

   CLIENT STREAM      REQUEST ──────────────>
                      ITEM ─────────────────>
                      END ──────────────────>
                      <────────────── RESPONSE | ERROR

   BIDI STREAM        REQUEST ──────────────>
                      ITEM ─────────────────>
                      <────────────────── ITEM
                      END ──────────────────>
                      <─────────────── END | ERROR
```

A call is terminated by exactly one terminating frame: RESPONSE, the callee's END, or ERROR. The caller's END closes only the caller direction (§5.5). CANCEL and CREDIT do not terminate a call: CANCEL asks the other side to send a terminating frame, and CREDIT grants a further item allowance (§5.6).

### 3.3 No service id and no method id

Service is a grouping of methods at declaration and registration time. On the wire the required information is which handler processes the frame, and the service does not participate in that.

Method already has a unique on-wire identifier: the message id of its request model. Fomoxa derives it from the message name, guarantees it is unique within the schema, and the handshake has verified that both sides interpret it identically. A separate `method id` field would spend 4 bytes on every frame repeating information the frame already carries.

This works because only REQUEST requires a dispatch lookup. The other six kinds are matched by `call id`:

```
   REQUEST   → dispatcher lookup by message id → handler
   RESPONSE  ┐
   ERROR     │
   ITEM      │
   END       ├→ pending lookup by call id → the record already knows the method
   CANCEL    │
   CREDIT    ┘
```

The accompanying constraint:

```
   Two methods MUST NOT share a request type.
```

This constraint is independent of the dispatch mechanism. Sharing a request type couples two methods during evolution: version skew under RFC-0002 §9.1 is evaluated per type, not per method, so appending a field for one method appends it for the other as well.

The case to note is the empty request: three methods without parameters MUST declare three distinctly named empty models. The cost is three 14-byte entries in the greeting, paid once per handshake.

Violating this constraint is a registration-time error (§4.10), not a runtime error.

`FRpcVoid`, `FRpcError`, `FRpcCredit` and `FRpcCtx` are ordinary models too. Declaring them in the schema buys three things: their message ids come from `fomoxac` and are guaranteed not to collide, so no reserved id range is needed; the handshake verifies that both ends agree on the shape of `FRpcError` and `FRpcCtx` before the first call is made; and `FRpcVoid` is one empty model shared by every method with no return value, rather than an empty model declared per method.

### 3.4 A declared context block instead of a metadata map

Some information accompanies every call without belonging to the business payload of any call: remaining deadline, trace id, token, tenant. Common RPC systems carry these in a free-form key/value map.

That approach does not apply here. A free-form key/value map is undeclared data: absent from the schema, unverified by the handshake, and in conflict with RFC-0001 §4.1. Accepting it forfeits the property in the first row of §1.1.

fRPC defines a declared context block, optional, separate from the request model:

```
   FRpcCtx
     deadline_ms       UInt32     remaining budget, propagated along a call chain §5.3
     trace_id          Bytes
     span_id           Bytes
     token             String
     tenant            String
     idempotency_key   Bytes      key for a safe retry §20.4
     initial_credit    UInt32     establishes the reply direction's allowance §5.6
```

RFC-0002 has no Optional and no default values, so every field MUST have a defined "unset" value:

| Field | "Unset" is | Meaning |
|---|---|---|
| `deadline_ms` | `0` | No deadline set. It does not mean "already expired" |
| `trace_id`, `span_id` | empty `Bytes` | No trace context |
| `token` | empty `String` | No credential attached to this call §10.4 |
| `tenant` | empty `String` | No tenant declared |
| `idempotency_key` | empty `Bytes` | The call is not declared safe to repeat §20.4 |
| `initial_credit` | `0` | The reply direction's allowance is not established, so it is unlimited. It does not mean "an allowance of zero" §5.6 |

In this revision, "unset" and "set to an empty value" are the same state. A field that needs to distinguish the two MUST carry a companion flag field; no other special value may be used to encode the distinction.

`FRpcCtx` is an ordinary Fomoxa model: the handshake verifies it, version skew under §9.1 applies to it, its bytes are deterministic, and `fomoxa-inspect` reads it. Interceptors read it generically without touching the method's request model (§9). When not enabled it occupies 0 bytes (§4.11).

Unlike a metadata map, `FRpcCtx` has a fixed, declared field set. Adding a field to `FRpcCtx` is a schema change and passes through the same process as any other schema change.

| Information | Location in fRPC |
|---|---|
| Deadline | `FRpcCtx.deadline_ms`, propagated along a call chain §5.3 |
| Trace id / span id | `FRpcCtx` |
| Authentication token | `FRpcCtx.token`, or established once per session §10 |
| Tenant / routing key | `FRpcCtx` |
| Compression flag | Bit 7 of the `kind` byte §11 |
| API version | Handled by the handshake fingerprint §2 C3 |
| Arbitrary caller-defined keys | Not supported |

---

## 4. Wire format

### 4.1 Overview of one fRPC message

```
   ┌────────────── net DATA frame (11 bytes) ──────────────┐
   │ 00 │ 'F' │ 'O' │ message id │ payload length │
   └────┴─────┴─────┴────────────┴────────────────┘
                                                   ┌─ payload ──┐
                                                   │ fRPC hdr 5B│ body │
                                                   └────────────┴──────┘
```

Net processes this frame as an ordinary message. fRPC reads a fixed header and a body.

### 4.2 The fRPC header

```
   ┌──────┬──────────────┐
   │ kind │ call id      │
   │ 1B   │ 4B u32 LE    │
   └──────┴──────────────┘
    ^0     ^1-4

   Exactly 5 bytes, fixed for every kind, including kinds with no body.
```

The method is identified by the frame's message id (§3.3, §4.4).

The first byte is split into bits:

```
   bit 7     body is compressed     §11
   bit 6     ctx block present      §4.11
   bit 5-3   reserved, MUST be 0
   bit 2-0   kind, values 0..6      §4.3
```

The receiver MUST check the three reserved bits. Ignoring them causes a future extension to be misinterpreted with no error signal.

The header has a fixed length so that a reader can locate the start of the body without branching on kind, including when reading a captured packet with no surrounding context.

`kind` occupies 1 byte rather than the 4 bytes of an RFC-0002 Enum because this header is not a Model. It is framing belonging to the fRPC layer, in the same category as net's `'F' 'O'` bytes: fixed by specification, not generated from a schema.

### 4.3 Kind table

| Value | Name | Body | Sender | Terminates the call |
|---|---|---|---|---|
| 0 | REQUEST | request model | caller | no |
| 1 | RESPONSE | response model | callee | yes |
| 2 | ERROR | `FRpcError` | callee | yes |
| 3 | ITEM | item model | the sending side of a stream | no |
| 4 | END | empty | the sending side of a stream | that direction |
| 5 | CANCEL | empty | caller | no |
| 6 | CREDIT | `FRpcCredit` | the receiving side of a stream | no |

ITEM and END travel in both directions; which one depends on the method's shape (§5.5). In server streaming the callee sends them, in client streaming the caller does, and in bidi streaming both do. END closes only the sending direction of whoever sent it, not the call: a client stream ends when the callee returns RESPONSE, a bidi stream ends when the callee sends END for its own direction.

Kind value 7 is unused and invalid. On receiving that value, or a non-zero reserved bit, the receiver returns ERROR with code `INTERNAL` for that call and MUST NOT close the session. This is an application-layer error, not a framing error; the only layer permitted to close a session over invalid bytes is net's frame layer (`02-flows` §2.5).

### 4.4 Semantics of the message id

The message id of a DATA frame always describes the body, never the call.

```
   REQUEST   → id of the method's request model   ← also the dispatch key
   RESPONSE  → id of the method's response model
   ITEM      → id of the item model of the sending direction   §5.5
   ERROR     → id of FRpcError
   END       → id of FRpcVoid
   CANCEL    → id of FRpcVoid
   CREDIT    → id of FRpcCredit
```

For REQUEST, one value serves two purposes: describing the body and selecting the handler. The two do not conflict because the constraint in §3.3 makes the mapping `request type → method` a bijection. The other six kinds only describe the body; their method comes from the pending record.

Consequences of this rule:

- Net's handshake continues to verify every request, response and item type at field level, through the mechanism of `02-flows` §3.3.
- Version skew under RFC-0002 §9.1 applies per type: appending a field to a request model is a valid change and handshake gate ③ accepts it.
- `fomoxa-inspect` reads the body by skipping the first 5 bytes, plus `[length][FRpcCtx]` when bit 6 is set (§4.11). A body with bit 7 set must be decompressed first (§11).

A required check follows: the receiver MUST compare the message id against the type expected for the pair `(method, kind)`, where the method comes from the dispatcher for REQUEST and from pending for the other kinds. On mismatch it returns ERROR `INVALID_ARGUMENT` and does not close the session. The rule applies to every kind, including END, CANCEL and CREDIT.

### 4.5 System models

fRPC declares seven models. `FRpcVoid` and `FRpcError` MUST be present in every schema that uses fRPC. `FRpcCredit` is needed only when the implementation supports flow control (§5.6), `FRpcCtx` only when the ctx block is used (§3.4), and the three reflection models only when the server enables reflection (§21).

Every system model MUST be declared with the codec named `rpc`. A message id derives from the model name plus the codec name (`FRpcVoid.rpc`), so a fixed codec name is what lets every party compute the same id without exchanging anything first. Reflection depends on it: an outside tool has to compute the id and fingerprint of the reflection models before it knows anything about the server. Application models may name their codecs freely.

```
   FRpcVoid
     (no fields)

   FRpcError
     code     UInt32
     message  String

   FRpcCredit
     items    UInt32

   FRpcCtx
     (seven fields, §3.4)
```

`FRpcVoid` has `n = 0`. Per `02-flows` §3.3 branch ⓓ, the empty prefix is a prefix of every chain, so this model never causes a handshake rejection.

`FRpcError` is extensible under RFC-0002 §9.1: appending a field is a valid version skew, and an older peer reads up to the fields it knows and stops. The planned extension is the pair `details_msg_id: UInt32` + `details: Bytes`, which attaches an application-defined model to an error while keeping every value on the wire typed and declared. The extension is not part of the current design (§13.2).

### 4.6 Error codes

Error codes are declared `UInt32`, not Enum. RFC-0002 §7 states that an Enum value outside the defined set is an invalid byte stream; if error codes were an Enum, a code added in a later revision would make older peers unable to decode the error frame at all.

| Code | Name | Meaning |
|---|---|---|
| 1 | CANCELLED | The caller cancelled |
| 2 | DEADLINE_EXCEEDED | Expired before a terminating frame arrived |
| 3 | UNIMPLEMENTED | No handler for this request type |
| 4 | INVALID_ARGUMENT | Wrong body type, or decoding failed |
| 5 | UNAUTHENTICATED | Not authenticated |
| 6 | PERMISSION_DENIED | Authenticated but not permitted |
| 7 | RESOURCE_EXHAUSTED | A ceiling in §4.8 was reached |
| 8 | FAILED_PRECONDITION | State does not permit the operation |
| 9 | UNAVAILABLE | The session ended with the call unfinished |
| 10 | INTERNAL | Unclassified error |

Code `0` is unused: RESPONSE and END already indicate success, so no "OK" code is needed on the wire. A code outside the table is treated as `INTERNAL`, with the original value retained for logging.

### 4.7 Call id

The call id is a 32-bit integer, allocated by the caller, increasing, and scoped to one session.

Both sides may open calls. To identify the allocating side, ids are split by parity:

```
   client allocates even call ids    0, 2, 4, ...
   server allocates odd call ids     1, 3, 5, ...
```

Wrap-around is permitted. Reusing an id currently in `pending` is NOT permitted. The `pending` set is bounded by the ceiling in §4.8, so this condition is always satisfiable.

32 bits rather than 64: the header is 4 bytes shorter on every message, at the cost of one wrap-around rule. The ceiling in §4.8 limits concurrent calls to several orders of magnitude below 2³².

### 4.8 Ceiling table

Per `02-flows` §8: every table MUST have a number.

| Quantity | Proposed ceiling | On reaching it |
|---|---|---|
| Pending calls, per session, per direction | 65,536 | Caller: reject locally. Callee: ERROR `RESOURCE_EXHAUSTED` |
| Frames queued for one call | 64 | The send call returns a congestion error §7.4 |
| Frames queued across the connection | 1,024 | As above. Both numbers are required. CREDIT does not count toward either ceiling §7.5 |
| Commands waiting on the driving thread's inbound queue (self-driven mode) | 1,024 | A send from another thread returns a congestion error §19 |
| Scheduler byte quantum per turn | 64 KiB | Yield to another call §7.2 |
| Time one call's queue stays continuously full | 5 seconds | Terminate that stream with `RESOURCE_EXHAUSTED` §5.4 |
| Registered methods | 65,536 | Registration-time error |
| ctx block length | 64 KiB | Allocation stop §4.11 |
| Body length | 16 MiB − 5 bytes | Local error for the caller; nothing is sent §2 C10 |

Implementation-chosen ceilings MAY change. The body length ceiling derives from net and MUST NOT change.

### 4.9 Byte example

Calling `Player.Get`, call id 4, request model `GetPlayerRequest { id: UInt32 = 7 }`. Assume the id of `GetPlayerRequest` is `0x4A2F8810`.

```
   00 46 4F              DATA frame, 'F' 'O'
   10 88 2F 4A           message id = 0x4A2F8810   (GetPlayerRequest)
   09 00 00 00           payload length = 9
   ──────────────────────────────────────────────── end of net's part
   00                    kind = REQUEST
   04 00 00 00           call id = 4
   ──────────────────────────────────────────────── end of fRPC header
   07 00 00 00           body: GetPlayerRequest.id = 7

   Total 20 bytes. Fixed overhead: 11 from net + 5 from fRPC = 16 bytes.
```

An END frame for the same call:

```
   00 46 4F  <FRpcVoid id>  05 00 00 00  04  04 00 00 00
   Total 16 bytes, empty body.
```

### 4.10 The dispatch table

The key is the message id of the request model. fRPC defines no hash function of its own, creates no second id space, and adds no id derivation step.

Three checks are REQUIRED at registration time:

```
   1. Two methods share a request type      → error, abort the process §3.3
   2. The fRPC id set intersects the set of
      the application's plain message ids   → error, abort the process
   3. A request/response/item type declared
      in a descriptor is absent from the
      schema                                → error, abort the process
```

The fRPC id set consists of every request, response and item type, in both directions, declared in descriptors, plus `FRpcError`, `FRpcVoid`, and `FRpcCredit` when present. `FRpcCtx` is not in the set because it only ever appears inside a body and is never the message id of a frame. The dispatch table is the subset containing the request types.

Check 2 exists because one session may carry both plain messages and RPC messages. An application that uses plain messages on the same session as fRPC MUST declare their ids. Overlapping sets produce an unresolvable state.

Net delivers every DATA frame as a MESSAGE event and does not filter by schema (`02-flows` §6.1). The premise "if the client has a message the server does not know, the server never receives it" (`02-flows` §3.3) holds only between cooperating peers, and a client such as a command-line tool, or a client deployed before its server, can send exactly such a message. The receiver therefore classifies a DATA frame along three branches:

```
   id in the declared plain-message set   → deliver to the application
   id in the fRPC id set                  → an fRPC frame, every rule applies
   id in neither set                      → NEVER delivered to the application
        valid 5-byte header → handled as an fRPC frame:
                              REQUEST → UNIMPLEMENTED (§5.2)
                              a frame of a pending call → INVALID_ARGUMENT (§4.4)
                              a call no longer open → ignored (§5.9)
        broken header       → dropped, no reply
```

The application receives only the messages it declared. An unidentified frame with a broken header is dropped without a reply, because a call id read from such a frame cannot be trusted; an ERROR for an arbitrary call id could end a real call at the other end.

There is no mechanism to detect these three errors at runtime.

### 4.11 The ctx block

When bit 6 of the `kind` byte is set, the body begins with a ctx block:

```
   ┌──────────────┬────────────────┬──────────────────┐
   │ ctx length   │ FRpcCtx        │ request model    │
   │ 4B u32 LE    │ = length bytes │                  │
   └──────────────┴────────────────┴──────────────────┘

   Bit 6 clear → the body contains only the request model; the block costs 0 bytes.
```

Only REQUEST may carry a ctx block. Bit 6 set on any other kind is an `INTERNAL` error for that call.

The length prefix is mandatory. Two concatenated models decode correctly only if the leading model has a fixed size. `FRpcCtx` does not, because it is allowed to evolve. With a sender carrying 6 fields and a receiver that knows 5, RFC-0002 §9.1 has the receiver stop at field 5 and treat the remainder as valid trailing bytes. Without a length prefix, that remainder is the leading bytes of the request model, and the request model decodes to incorrect values with no error signal.

The 5-byte header in §4.2 needs no prefix because it is fixed, specified rather than schema-generated, and does not evolve with the schema.

The ctx block length ceiling is 64 KiB, serving as an allocation stop, in the same category as net's 1 MiB ceiling for the handshake frame.

The frame's message id still describes the request model, not the ctx block (R3). The ctx type is fixed and therefore needs no identifier.

`FRpcCtx` is not declared as the leading field of every request model because that forces every request model to declare an additional field; the fingerprint and field count `n` of every method would then change whenever `FRpcCtx` changes, putting the entire method table into version skew simultaneously. A separate length-prefixed block keeps the two evolution axes independent.

---

## 5. Flows

### 5.1 Unary, success path

```
   CLIENT                                          SERVER
     │                                               │
     │ (session READY - required, §2 C3)             │
     │                                               │
     │ ── DATA[ REQUEST, call=4, msg=GetPlayerReq ] > │
     │                                               │ dispatcher lookup by msg
     │                                               │ interceptors inbound §9
     │                                               │ handler runs
     │                                               │ interceptors outbound
     │ <── DATA[ RESPONSE, call=4, msg=GetPlayerRes ]─│
     │                                               │
     │ remove 4 from pending, complete the call      │
```

On receiving a RESPONSE the receiver MUST check two conditions in order, stopping at the first that fails:

```
   ① is the call id in pending?      no → ignore, no error raised (§5.9)
   ② is the message id the response
      type expected by that call's
      method?                        no → complete the call with
                                          INVALID_ARGUMENT
```

Condition ① ignores rather than reports because a RESPONSE arriving after a call has expired is a normal state; see §5.9.

### 5.2 Unary, error path

The handler returns an error, an interceptor rejects the call, or no handler exists:

```
     │ ── DATA[ REQUEST, call=4, msg=X ] ──────────> │
     │                                               │ X is not in the
     │                                               │ dispatch table
     │ <── DATA[ ERROR, call=4, FRpcError{3,…} ] ─── │
```

A missing handler is not a handshake failure. The handshake guarantees that both sides interpret the same set of types (§2 C3) and accepts the case where one side has a message the other does not know (`02-flows` §3.3). It does not guarantee that the other side implements the method. `UNIMPLEMENTED` therefore exists even though schemas match.

If `X` is absent from the receiver's schema, it belongs to none of the receiver's id sets. Under §4.10 the frame is not delivered to the application but handled as an fRPC REQUEST, and the caller receives `UNIMPLEMENTED` at once. The common case is a client that deploys a new method before its server does.

### 5.3 Deadline, propagation and cancellation

The deadline travels on the wire as remaining time in milliseconds: `FRpcCtx.deadline_ms`.

A duration is used rather than an absolute point in time because the two machines share no clock, and Fomoxa forbids wall-clock reads at every layer (B9). An absolute timestamp is only meaningful if both sides are time-synchronised.

```
   A ──[ deadline_ms = 2000 ]──> B
                                  B spends 300 ms
                                  B ──[ deadline_ms = 1700 ]──> C
```

C knows its remaining budget, and no hop processes a request that A has already abandoned.

The caller keeps a local absolute deadline, computed from the tick's `now` at send time:

```
   tick N     send REQUEST, record pending{ call=4, deadline = now + 2s }

   tick N+k   clock step: now > deadline
                ├── remove 4 from pending
                ├── complete the call: DEADLINE_EXCEEDED
                └── enqueue CANCEL(call=4)

   server     receives CANCEL
                ├── call still running → signal the handler, send ERROR CANCELLED
                └── call already done  → ignore, no error raised
```

Three rules:

1. The caller completes the call first and sends CANCEL afterwards. The caller's outcome does not depend on whether CANCEL arrives.
2. The callee treats `deadline_ms` as an upper bound, not as an instruction. For unary calls it MUST keep its own per-handler limit: a large or zero `deadline_ms` must not hold resources indefinitely, and on a packet transport CANCEL may never arrive (§2 C9). The effective limit is the smaller of the two. Streams follow their own rule in §5.4.
3. Deadlines use the `now` passed into the tick (§2 C8). An implementation MUST NOT read the system clock here; this condition is what makes a full expiry cycle testable.

A middle hop builds the context for the next hop from the context it received. Not every field is copied:

| Field | What the middle hop does | Why |
|---|---|---|
| `deadline_ms` | The budget remaining at the moment of the onward call | The propagation rule above |
| `trace_id`, `tenant` | Copied verbatim | They identify the chain, not one hop |
| `span_id` | Copied verbatim as the parent span | A hop with a tracer overwrites it with its own new span |
| `token` | NOT copied | A credential belongs to exactly one hop §10.4. Forwarding it is an application decision, not a layer default |
| `idempotency_key` | NOT copied | The key derives from this hop's body §20.4 and does not apply to another |
| `initial_credit` | NOT copied | An allowance belongs to one call §5.6 |

Two edge cases:

1. If the budget is exhausted, the middle hop MUST NOT make the onward call. Encoding an exhausted budget as `deadline_ms = 0` is wrong: under §3.4 that value means unlimited, so a spent budget would become an unbounded one at the next hop.
2. If the caller declared no `deadline_ms`, the propagated budget is the middle hop's own handler limit (rule 2 above). The next hop must not be allowed to run longer than the middle hop will wait for it. A handler with no total limit (a stream, §5.4) passes `deadline_ms = 0` to the next hop.

The deadline lives in the ctx block rather than in the header because placing it in the header would cost 4 bytes on every frame, including RESPONSE, ITEM, END and CANCEL, which have no deadline semantics.

### 5.4 Server streaming

```
     │ ── DATA[ REQUEST, call=6, msg=WatchReq ] ───> │ open the stream
     │ <──────────── DATA[ ITEM, call=6 ] ────────── │
     │ <──────────── DATA[ ITEM, call=6 ] ────────── │
     │ ── DATA[ CANCEL, call=6 ] ─────────────────> │ (caller cancels)
     │ <──────────── DATA[ ERROR, call=6, CANCELLED ]│
```

Rules:

- An ITEM arriving after a terminating frame is ignored, with no error raised.
- A non-zero `ctx.deadline_ms` bounds the entire call, not only the time to the first item.
- A stream with `deadline_ms = 0` has no total limit. It lives until a CANCEL, a terminating frame, or the end of the session. The mandatory unary handler limit (§5.3 rule 2) does not apply to streams. Streams run on a stream transport (§5.7), so CANCEL cannot be lost while the session is alive; a dead peer is caught by net's heartbeat, which ends the session, and §5.8 cleans up the call. A stream that is silent in both directions for a long time, such as a watch on rare events, is a valid state.
- The callee MAY declare an idle limit per method: the longest period with no frame of the call received from the other side, CREDIT included. Past it the call terminates with `DEADLINE_EXCEEDED`. The idle limit is implementation policy for catching application bugs, such as a side that lost its handle on a call; it is not an interoperability rule (§15.1).
- The producer is bounded by that call's queue (§7.4) and MUST distinguish two situations:

```
   queue full at this moment       → tell the producer to wait for the next tick
                                     normal condition

   queue full continuously beyond  → terminate the stream, RESOURCE_EXHAUSTED
   the threshold                     the receiver is not consuming
```

  The first situation occurs whenever a handler produces faster than the link drains. The threshold is in §4.8.

  This is local back pressure: it reaches the producer at this end, not the producer at the other end. Back pressure that crosses to the other end is the credit allowance in §5.6.

- Item order is the order in which ITEM frames arrive. fRPC does not number or reorder them (§2 C9).

### 5.5 Client streaming and bidi streaming

The two remaining shapes use exactly the kind set of §4.3; only the direction of ITEM and END changes.

```
   Client streaming                        Bidi streaming

   │ ── REQUEST, call=8 ──────> │          │ ── REQUEST, call=10 ─────> │
   │ ── ITEM, call=8 ─────────> │          │ ── ITEM, call=10 ────────> │
   │ ── ITEM, call=8 ─────────> │          │ <───────── ITEM, call=10 ─ │
   │ ── END, call=8 ──────────> │          │ ── ITEM, call=10 ────────> │
   │ <────── RESPONSE, call=8 ─ │          │ <───────── ITEM, call=10 ─ │
                                           │ ── END, call=10 ─────────> │
                                           │ <────────── END, call=10 ─ │
```

Rules:

- The message id of an ITEM sent by the caller describes the caller-direction item model, which is distinct from the message id of the REQUEST that opened the call. The method declares both. A method that declares no separate id reuses the REQUEST's id for caller items.
- END from the caller closes the caller direction. After it the caller MUST NOT send further ITEM frames on that call.
- An ITEM or END arriving after that direction has closed is ignored, with no error raised, under the same rule as §5.4.
- An ITEM carrying a message id that is not that direction's item model is ERROR `INVALID_ARGUMENT` and terminates the call. This is level two of §5.9.
- The callee terminates the call with RESPONSE (client streaming) or with END for its own direction (bidi streaming). Until then the call stays open even though the caller has closed its direction.
- Call id parity already identifies which side opened the call (§4.7), so no further marker is needed to tell which direction an ITEM belongs to.

### 5.6 Flow control by credit allowance

The back pressure in §5.4 reaches only the local producer. When the producer at the other end generates items faster than the receiver consumes them, the full queue sits at the sending end and the receiver has no way to slow it down. The CREDIT frame (kind 6, §4.3) is that channel.

```
   FRpcCredit
     items   UInt32   how many further items the receiver is prepared to accept
```

The allowance is per direction. Each direction that carries items in a call has its own allowance, controlled by the receiver of that direction, and the allowance is in one of two states:

```
   NOT ESTABLISHED    → the producer sends without limit
        │
        │  the first establishing signal from this direction's receiver, exactly once
        ▼
   ESTABLISHED (n)    → the producer may send at most n ITEM frames
                        each ITEM sent    → n drops by 1
                        n = 0             → the producer stops, the call stays open
                        CREDIT arrives    → n increases, the producer resumes
```

There is no way back from ESTABLISHED to NOT ESTABLISHED. Once established, CREDIT only ever increases the allowance.

Establishing signals:

| Direction | Receiver of that direction | Establishing signal |
|---|---|---|
| Reply direction (server stream, bidi) | The caller | A non-zero `FRpcCtx.initial_credit` on the REQUEST; otherwise the first CREDIT |
| Caller direction (client stream, bidi) | The callee | The first CREDIT |

The reply direction is established through the ctx because the callee runs the handler within the same tick that delivers the REQUEST (§6 step 2); a CREDIT sent after the REQUEST arrives once the handler has already produced items. The caller direction needs no such path: the callee sends its first CREDIT within the tick that delivers the REQUEST.

Rules:

- An endpoint that implements no flow control sends no establishing signal, so every direction it receives stays NOT ESTABLISHED and the other end behaves exactly as it did before the mechanism existed. This is the condition under which adding kind 6 breaks no compatibility.
- `initial_credit = 0` is "unset" under §3.4, so it establishes nothing. Opening a direction with an allowance of zero needs a companion flag field; that is reserved in §13.2.
- A receiver MUST NOT treat items beyond the allowance as an error, and MUST deliver them as ordinary items. Such items have two legitimate sources: items the producer sent before the establishing signal arrived, and items from an endpoint that implements no flow control but still received `initial_credit`. The caller direction can deliver up to roughly one round trip of items before the first CREDIT takes effect.
- CREDIT accumulates, saturating at the `UInt32` ceiling.
- An allowance of zero does NOT terminate the call and does NOT count toward the queue-full threshold of §5.4. A receiver that grants slowly is doing so deliberately; it is not a stalled receiver. The call remains bound by `deadline_ms` and by the idle limit when one is declared (§5.4).
- On a bidi stream an arriving CREDIT increases the allowance of the sending direction of whichever side received that frame.
- A CREDIT frame for a call that is no longer open is ignored, with no error raised. This is level one of §5.9.
- `FRpcCredit` MUST have its own message id, not shared with `FRpcVoid` or `FRpcError`. An implementation that implements no flow control need not declare the model. Such an implementation ignores every CREDIT frame it receives: no error, no terminated call, and the frame never reaches the application (§4.10). The sender of the CREDIT loses nothing, because the other end sends without limit anyway.
- CREDIT is a control frame and follows its own queueing rules in §7.5.

### 5.7 Transport: default and limits

fRPC inherits net's transport independence (§1.1). Different transports do not yield identical semantics.

The default for fRPC is a stream transport: TCP, TLS, WebSocket, QUIC stream. Every semantic statement in this document applies to that mode.

On packet transports, packets may be lost, arrive out of order, or arrive twice. fRPC does not correct any of these, per C9.

| Call shape | Stream transport | Packet transport |
|---|---|---|
| Unary | Correct semantics | REQUEST lost → DEADLINE_EXCEEDED. RESPONSE lost → the same. REQUEST delivered twice → the handler runs twice |
| Server stream, client stream, bidi | Correct semantics | Undefined; out-of-order items are not detectable, and END can arrive before the last ITEM |
| CREDIT | Correct semantics | A lost CREDIT stalls the producer until the call's deadline or idle limit |

Streaming and flow control require a stream transport. Unary over a packet transport is at-most-once from the caller's perspective and may be duplicated at the callee, so methods used in that mode MUST be idempotent. Deduplication through a table of processed call ids belongs to the application; placing it in fRPC would rebuild the reliability layer that RFC-0001 §2 removed.

### 5.8 Session termination

Net emits exactly one termination event (§2 C11). fRPC handles it as follows:

```
   DISCONNECT or HANDSHAKE FAILED
        │
        ├── every pending call (caller side)  → complete with UNAVAILABLE
        ├── every running call (callee side)  → signal the handler, send nothing
        ├── clear the send queues
        └── set the cleaned flag - a second path MUST NOT clean again
```

No ERROR is sent for running calls because the session has ended. Handlers MUST receive a cancellation signal in order to release resources; this is why a handler needs a cancellation signal and not only a return value.

### 5.9 Three levels of abnormality

fRPC distinguishes three levels. The first occurs in normal operation and MUST NOT be treated as an error.

| Level | Examples | Consequence |
|---|---|---|
| Benign race | RESPONSE for a call id no longer pending · ITEM after END · CANCEL or CREDIT for a finished call · REQUEST reusing a running call id | Ignore. No error raised, no error-level log entry, no event |
| RPC error | `UNIMPLEMENTED`, `PERMISSION_DENIED`, `INVALID_ARGUMENT`, handler business errors | ERROR, terminates one call |
| fRPC protocol violation | kind 7 · non-zero reserved bit · wrong call id parity · body fails to decode · ctx above ceiling | ERROR for that call; the session stays alive (R4) |

The first level arises when two frames cross on the wire:

```
   client deadline fires → complete the call, send CANCEL
        ↓ concurrently
   server has already sent RESPONSE
        ↓
   client receives a RESPONSE for a call no longer in pending
```

Neither side violated the protocol. Treating this as a violation turns every timeout that races a response into an incident.

Two REQUIRED rules, because this is where two implementations diverge most easily:

An fRPC protocol violation MUST NOT close the session. R4 states that only net's frame layer closes sessions.

A REQUEST reusing a running call id MUST be ignored, not rejected. Returning `INVALID_ARGUMENT` for that call id would terminate the legitimate running call. Ignoring it also prevents a duplicated REQUEST on a packet transport from invoking the handler a second time while the original call is still running (§5.7).

If the `FRpcError` in an ERROR frame fails to decode, the call completes with `INTERNAL`.

---

## 6. The fRPC tick

fRPC owns the net session and exposes one entry point, `tick(now)`. The following order is REQUIRED; it mirrors §8 of `02-flows.md`:

```
   frpc_tick(now):

       ── 1. net.tick(now) ──
       collect the event list. Net has already flushed its pending slot first.

       ── 2. process events in the order net returned them ──
           READY        → allow calls to be sent
           MESSAGE      → is the id a declared plain message? §4.10
                          yes → deliver to the application
                          no  → split off the 5-byte header, classify kind
                                (unknown id with a broken header → drop, no reply)
                          decode the body here (§2 C7)
                          REQUEST → dispatcher; may enqueue frames
                          RESPONSE/ERROR/ITEM/END → complete or feed the call
                                    on the caller side; caller-direction
                                    ITEM/END go to the handler on the callee side
                          CANCEL  → signal the handler
                          CREDIT  → add to the call's send allowance §5.6
           DISCONNECT   → clean up per §5.8, exactly once
           POLL / REPLY → not processed by fRPC

       ── 3. deadline clock ──
       scan pending; expired calls → complete + enqueue CANCEL

       ── 4. resume unfinished handlers ──
       handlers that declared NOT DONE on a previous tick are called again §8
       results are enqueued

       ── 5. scheduler drains into net ──
       round-robin across calls, byte quantum §7.2
       loop: net.send(...) until the queues are empty or net reports congestion

       ── 6. return results to the application ──
```

Two ordering requirements:

Step 3 after step 2. A RESPONSE arriving in the same tick as the deadline takes precedence. The reverse order reports some calls as expired although the answer was already in the buffer.

Step 5 after step 4. A REQUEST arriving in this tick is answered within the same tick. The reverse order delays every answer by one tick, and the delay accumulates across the hops of a call chain.

---

## 7. Send queues, scheduler and congestion

Net holds one pending slot and refuses to queue (§2 C5). Net also forbids overwriting that slot, because it may hold a handshake verdict. fRPC's queues sit above net and do not replace net's mechanism.

### 7.1 One queue per call

A single FIFO for the whole connection produces head-of-line blocking:

```
   call 7 enqueues 1,000 ITEM frames
   call 8 is a small unary, enqueued afterwards
        → call 8 waits for all 1,000 frames of call 7
```

The required layout:

```
       Call A        Call B        Call C
       queue         queue         queue
          │             │             │
          └─────────────┼─────────────┘
                        ▼
                   SCHEDULER            round-robin, byte quantum
                        │
                        ▼
                    net.send            one frame at a time
```

Three invariants:

```
   1. Order is guaranteed WITHIN each call.
   2. The scheduler MAY interleave BETWEEN calls.
   3. Bytes on the wire are never interleaved within a frame  (net's B7).
```

The prohibition on reordering applies within one call: an ERROR placed ahead of that call's final ITEM would arrive before the data it terminates. It does not apply between two different calls.

### 7.2 Byte quantum

The scheduler yields after a call has sent a defined number of bytes, not a number of frames. Frame sizes vary widely, so counting frames does not reflect the share of the link consumed.

The quantum cannot subdivide a frame. It determines how many frames of one call are sent before yielding. A 16 MiB frame occupies net's pending slot until it has been flushed, and B7 forbids other frames from interleaving.

The scheduler therefore comes with a recommendation:

```
   An RPC message body SHOULD stay below ~64 KiB.
   Larger data SHOULD be split into a stream.
   The 16 MiB ceiling is net's ceiling, not a normal working size.
```

### 7.3 Terminating a call discards its queue

When a call terminates, frames still queued for that call MUST be discarded, after which the terminating frame is enqueued.

Discarding is not reordering: the call has ended. Without this rule a CANCEL takes effect only after the call's entire queue has been drained.

Terminating frames (RESPONSE, the callee's END, ERROR) and CANCEL MUST NOT be refused because of the connection ceiling in §4.8. Each call has at most one such frame, so memory stays bounded. A terminating frame dropped to congestion would leave the other side waiting until its deadline for a call that has already finished. ITEMs are the opposite: when the connection ceiling is reached the producer waits for the next tick, and no item is lost.

Edge case: if the REQUEST has not left the queue, cancellation discards it and sends nothing. The other side never received the call, so a CANCEL would be ignored there (§5.9). The call completes locally with `CANCELLED` and no bytes reach the wire.

### 7.4 Congestion

```
   application call  → fRPC builds the frame, enqueues it on the call, returns immediately
   ceiling reached   → return a congestion error; do NOT queue further
   tick step 5       → the scheduler drains into net until net reports congestion
```

Two rules:

1. Two ceilings (§4.8): one per call, one for the total number of frames queued on the connection. With only a per-call ceiling, the total still grows with the number of calls.
2. Reaching a ceiling is an error returned to the caller, not a session-terminating error. The application decides whether to drop, slow down or disconnect, per `01-overview` §5.

The scheduler's scope is local fairness on the sending side. It has no knowledge of how fast the receiver consumes; that information arrives in CREDIT frames (§5.6). When a call's allowance runs out, that call's producer stops, and the scheduler keeps draining the other calls as usual.

### 7.5 CREDIT is a control frame

If CREDIT followed the same rules as data frames, it could be lost: when a call's queue is full because net is congested, sending a CREDIT returns a congestion error under §7.4, the CREDIT never reaches the wire, and the producer at the other end stops indefinitely. CREDIT therefore follows three rules of its own:

```
   1. Coalesce         each call has at most ONE CREDIT waiting to send per
                       direction. A new CREDIT adds to the waiting one,
                       saturating at the UInt32 ceiling.

   2. Outside ceilings CREDIT does not count toward the per-call or the
                       connection ceiling (§4.8). Because of rule 1 the number
                       of waiting CREDITs cannot exceed the number of open
                       calls, so memory stays bounded.

   3. Goes first       CREDIT is sent ahead of the ITEMs already queued for
                       the same call. CREDIT concerns the opposite direction
                       and has no causal relation to queued ITEMs. Exception:
                       CREDIT MUST NOT overtake the REQUEST that opens its call.
```

The exception in rule 3 is required. A caller can grant CREDIT for the reply direction while its REQUEST is still queued. If the CREDIT reached the wire first, the other side would receive CREDIT for a call that does not exist yet, ignore it under level one of §5.9, and the allowance would be lost.

Granting CREDIT therefore never returns a congestion error. The ordering rule of §7.1 still holds for data frames and terminating frames. When a call terminates, its waiting CREDIT is discarded with the rest of its queue (§7.3).

CREDIT creates no circular wait between the two directions of a bidi stream: CREDIT consumes no allowance, and items held back by an exhausted allowance are not in any queue, because the producer stops generating them (§5.6).

---

## 8. Handlers on the tick loop

There are no background threads (§2 C6). Handlers run inside step 2 of the tick. A long-running handler stops the loop; the heartbeat runs in step 3 of net's tick, so once the tick has been stopped long enough, the peer sends a probe, receives no reply, and terminates the session (`01-overview` §6).

A handler has two result shapes:

```
   DONE        → a return value, or an error. fRPC builds RESPONSE/ERROR immediately.

   NOT DONE    → a resumption point for fRPC to call again in step 4 of later ticks.
                 fRPC keeps the call in pending and sends nothing.
```

The second shape covers long work split across ticks and waiting on another system. The condition: that wait state MUST be checkable with a non-blocking operation on each tick. Work that does not satisfy this condition does not belong on the tick loop; the application runs it elsewhere and submits the result to fRPC as an event.

A handler MUST NOT call back into `frpc_tick`, for the same reason transports must not call back into the core (`01-overview` §12): recursion into partially updated state.

This section describes the pure core. §19 defines a second driving mode in which fRPC owns its own thread and handlers may block. Both modes produce identical bytes.

---

## 9. Interceptors

Interceptors form one chain that runs in two directions.

```
   CALLEE SIDE                        CALLER SIDE
   REQUEST                            call()
     ↓                                  ↓
   [1] log / metrics                  [1] log / metrics
     ↓                                  ↓
   [2] authentication §10             [2] attach token
     ↓                                  ↓
   [3] authorization §10              [3] deadline
     ↓                                  ↓
   handler                            enqueue
     ↓                                  ↓
   chain unwinds 3 → 2 → 1            chain unwinds on completion
```

The context passed along the chain carries:

```
   call id · method (descriptor reference) · kind · session identity
   now (the current tick's timestamp) · deadline if present
   identity §10 · cancellation signal
```

The context contains no metadata map (§3.4). Data written into the context exists only in-process and does not cross the network.

Five constraints:

- Interceptors run inside the tick and are bound by §8: no blocking, no re-entry into fRPC. Unlike handlers, interceptors have NO NOT DONE shape; the chain MUST complete within one tick. Consequently, authentication MUST be a local computation; see §10.1.
- An interceptor MAY terminate a call early with an error. The handler then does not run and the chain unwinds from the link that rejected it.
- Every link whose `outgoing` or `inbound` step ran MUST receive `outbound` exactly once, whatever ends the call: a response, an error, a local failure to enqueue the REQUEST, the deadline, or the end of the session.
- Chain order is configured at startup and fixed for the life of the session. Changing it mid-session sends two calls of the same method through different paths.
- On the caller side, the interceptor chain MUST complete before the ctx block is encoded, because the chain is where the token, deadline and trace id are attached.

---

## 10. Authentication and authorization

Authentication and authorization answer two distinct questions, and `02-flows` §3.3 states that the handshake answers neither:

```
   Authentication   "who is the other side"            → once per session
   Authorization    "may it call this method"          → per call
```

### 10.1 Authentication once per session

fRPC defines no authentication mechanism. It defines where such a mechanism attaches:

```
   session READY
     │
     │ the caller invokes an application-declared authentication method
     │   (token in the request model or in FRpcCtx - §3.4)
     ▼
   the callee verifies and attaches an identity to the session
     │
     ▼
   every later call reads the identity from the context §9
```

Before an identity exists, the authentication interceptor rejects every method not on the exemption list. That list MUST be declared explicitly.

Authentication uses an ordinary call rather than a dedicated kind because that reuses the existing schema, fingerprint, interceptor, deadline and error code paths, and adds no branch to the fRPC header. Changing the authentication algorithm does not change the RPC layer's format.

The authentication check MUST be a local computation. Interceptors have no NOT DONE shape (§9), so an out-of-process call to validate a token blocks the tick loop; once the tick has been stopped long enough, the heartbeat stops and the peer terminates the session (§8).

```
   PERMITTED                              NOT PERMITTED
   ─────────                              ─────────────
   verify a signature locally             remote token introspection
   look up an in-memory identity cache    database queries
   match against a preloaded key set      waiting on any operation
```

Remote data (rotating public keys, revocation lists, quotas) MUST be loaded in the background into a cache and MUST NOT sit on a call's processing path. If the cache is not ready, reject with `UNAVAILABLE`.

This constraint narrows the choice of algorithm: credentials verifiable by local computation (asymmetric signatures, a MAC with a preloaded key) are suitable. Credentials that can only be verified by querying another system are not, unless the query result is already cached.

### 10.2 Per-method authorization

Permissions are descriptor data (§12), not wire data: each method declares its requirement and the authorization interceptor compares it against the identity. Failure returns `PERMISSION_DENIED`. No identity returns `UNAUTHENTICATED`. The two codes are separate because the caller handles them differently: one case requires re-authentication, the other is unaffected by re-authentication.

### 10.3 Boundary with TLS

```
   TLS        protects the channel      → transport, outside Fomoxa's scope
   auth       identifies the caller     → fRPC
```

The two mechanisms are independent. An internal link between two services may omit TLS and still authenticate. TLS does not determine which methods the other side may call.

### 10.4 One connection, one identity

```
   A connection has EXACTLY ONE authenticated identity.
   A per-call credential, if present, MUST NOT replace that identity.
```

Four cases:

| Session state | `ctx.token` | Handling |
|---|---|---|
| Not authenticated | Present | Usable only for methods on the exemption list (§10.1). Other methods: `UNAUTHENTICATED` |
| Authenticated | Empty | Use the session identity |
| Authenticated | Same principal as the session | Accept; no state change |
| Authenticated | Different principal | `PERMISSION_DENIED`. Identity unchanged, handler not run |

The fourth case prevents two outcomes. First, a connection authenticated with a low-privilege identity could escalate per call. Second, log records would show the session identity while the call executed under a different principal, removing the ability to correlate during an investigation.

Delegation, where a service calls on behalf of a user, remains possible in one direction:

```
   session identity     = who opened this connection    ← source of authority
   per-call credential  = on whose behalf               ← NARROWS only
```

The effective permissions of a call are the intersection of the two permission sets, NEVER the union. Both MUST appear in the audit record.

---

## 11. Compression

Compression sits after the codec and before net, which places it inside fRPC, at the boundary between the two calls fRPC makes.

```
   model ──codec──> body bytes ──compress──> compressed body bytes
                                                  │
                      fRPC header (uncompressed) + body
                                                  │
                                                net.send
```

Rules:

- The compression flag is bit 7 of the `kind` byte (§4.2).
- Compression applies only to the model in the body: request, response, item or `FRpcError`. The 5-byte header and the whole `[length][FRpcCtx]` are always uncompressed.

```
   ┌──────────────┬──────────────┬─────────────────────────┐
   │ ctx length   │ FRpcCtx      │ model                   │
   │ uncompressed │ uncompressed │ compressed if bit 7 set │
   └──────────────┴──────────────┴─────────────────────────┘
```

- This boundary follows from the receiver's processing order:

```
   read header → read ctx → interceptors / auth → DECOMPRESS → decode model → handler
```

  The authentication interceptor reads the token before deciding whether to process the call. Compressing the ctx block would force decompression before the token can be read, performing the heaviest work even for calls that will be rejected.

- Compression does not change the message id. The message id describes the type of the body after decompression (§4.4).
- The post-decompression ceiling equals the body ceiling (§4.8). The decompressor MUST check incrementally and MUST NOT allocate according to a declared size and check afterwards. This is an allocation stop of the same kind net applies to the handshake frame.
- Small bodies SHOULD NOT be compressed. The threshold is configuration, not protocol.

The compression algorithm is fixed by configuration on both sides. Net's handshake has a fixed layout (`02-flows` §3.2), and adding a field to it is a protocol version change. The other option is negotiation through an fRPC method invoked after READY, which is reserved in §13.2.

Compression changes determinism at the frame level, not at the model level. The body after decompression MUST be exactly the canonical byte sequence the codec produces. Signatures and hashes MUST be computed over the decompressed body and MUST NOT be computed over the bytes on the wire.

---

## 12. Service registration and descriptors

A descriptor is data. Each method declares:

```
   full name            "Player.Get"          (for reading and logging)
   call shape           unary | server stream | client stream | bidi
   request type         message id + type      ← the method's identifier §3.3
   caller item type     message id + type      ← client stream and bidi only §5.5
   response/item type   message id + type
   permission required  §10.2
   retry safety         unsafe | idempotent | keyed     §20.3
   idle limit           streams only, optional          §5.4
```

The full name does not appear on the wire and does not participate in dispatch. Renaming a method changes no bytes; renaming the request model does, because the id derives from the message name.

Both directions are built from the descriptor: the callee's dispatch table and the caller's stub. The three registration checks in §4.10 run over this data.

A typed stub is a per-language API convenience and belongs to the second row of §15.1: an implementation offering only the byte-level call path interoperates byte for byte with one that generates stubs. It is not part of any phase's completion condition.

`fomoxac` MAY generate descriptors. It MUST NOT be a prerequisite for running fRPC: consistent with the project's positioning, fomoxac is a supporting tool while the protocol must be reconstructible from documentation alone. Descriptors can be written by hand, and the specification must be sufficient for that.

---

## 13. Boundaries

### 13.1 Excluded by principle

The following are on no roadmap. Implementing them would contradict a decision made in a lower layer.

| Excluded | Reason |
|---|---|
| Retransmission, reordering, deduplication | Fomoxa does none of these (§2 C9). Adding them here rebuilds the reliability layer in the wrong place |
| Free-form key/value metadata map | Conflicts with RFC-0001 §4.1. The corresponding need is met by the declared ctx block (§3.4, §4.11) |
| Multiple sessions per call | A call belongs to exactly one session and ends with it (§5.8) |
| Automatic reconnection | `01-overview` §12 forbids it at the transport layer. The reconnection decision belongs to the application |
| Defence against a non-cooperating peer | Handshake gate ③ trusts the list the other side declares; so does fRPC. This is a protocol between cooperating parties |
| Changing fomoxa-net to serve fRPC | §0 |

### 13.2 Not yet built, space reserved

None of the following requires a change to the header layout; the reserved bits in §4.2 account for them.

| Not yet built | Reserved space |
|---|---|
| `details` in `FRpcError` | Append a field; a valid version skew (§4.5) |
| Health check | An ordinary fRPC method; no format change. Method listing has become reflection, §21 |
| Compression algorithm negotiation | Bit 7 exists; the negotiation uses a method invoked after READY (§11) |
| ctx on kinds other than REQUEST | Bit 6 exists; only the rule in §4.11 widens |
| Opening the reply direction with an allowance of zero | Append a flag field to `FRpcCtx` to tell "0" apart from "unset" (§3.4, §5.6). An older peer ignores the flag and sends without limit, which is safe under the rule on items beyond the allowance |

Kind 7 is the only value left free in the three `kind` bits. An eighth kind would take one of the three reserved bits in §4.2 as an extension bit, rather than a layout change.

---

## 14. Invariants

| # | Invariant |
|---|---|
| R1 | Net is unaware of fRPC. No new frame type, no added bytes in a frame, no handshake change |
| R2 | The fRPC header is exactly 5 bytes, fixed for every kind. No method identifier on the wire |
| R3 | A frame's message id always describes the body §4.4 |
| R3b | Two methods do not share a request type, and the fRPC id set is disjoint from the plain message id set §4.10 |
| R4 | An fRPC-layer error terminates one call, not the session. Only net's frame layer closes a session |
| R5 | A call is terminated by exactly one terminating frame, and completed exactly once on the caller side |
| R6 | On session termination, every pending call completes exactly once with `UNAVAILABLE` |
| R7 | Every timestamp in the core comes from the `now` passed into the tick. The core reads no clock; the shell reads the clock and passes it in §19 |
| R8 | The core does not block; an unfinished handler declares NOT DONE. The shell blocks only in the wait step between two ticks. A handler may block only on a worker in self-driven mode §19 |
| R9 | The body is decoded or copied within the receiving tick §2 C7 |
| R10 | Order is guaranteed within each call; the scheduler may interleave between calls; terminating a call discards its queue §7 |
| R11 | The ctx block always carries a length prefix §4.11 |
| R12 | fRPC requires no change in fomoxa-net §0 |
| R13 | A connection has exactly one authenticated identity. A per-call credential only narrows permissions §10.4 |
| R14 | No check on a call's processing path leaves the process. Authentication is a local computation §10.1 |
| R15 | The application receives only the plain messages it declared. A frame in neither id set never reaches the application §4.10 |

---

## 15. Roadmap

Two ordering principles, both derived from Fomoxa's constraints:

- Deadlines and pending cleanup belong in phase 1. Per §2 C9 there is no retransmission layer below, so the deadline is the only mechanism that removes a call from the waiting state. A unary implementation without deadlines retains pending records indefinitely.
- Method models land early. `fomoxac` already derives a message id, a fingerprint and a codec for every model, and refuses to generate a schema whose message ids collide (`SPEC-FINGERPRINT.md` §4). fRPC reuses that mechanism rather than defining an identity layer of its own on top. The remaining information (the call shape, the method name, the permission) is policy rather than byte layout, so it is declared at the registration site instead of generated from the schema.

| Phase | Contents | Completion condition |
|---|---|---|
| 1 | Header, call id, dispatch, unary, `FRpcError`, deadlines, pending cleanup on session termination | Two SDKs in different languages exchange unary calls; a call with no answer still terminates and releases resources |
| 2 | Send queues, scheduler, congestion, two driving modes §19 | Burst sending does not grow memory without bound; a long handler does not lose the connection |
| 3 | CANCEL, server streaming, END | A stream cancelled mid-flight leaves no pending call on either side |
| 4 | Method models and the system models declared in the schema; ids, fingerprints and codecs taken from `fomoxac` | Adding a method takes a model and one registration line; no message id is written by hand |
| 5 | ctx block, deadline propagation, interceptors | A trace id survives a three-hop chain; the last hop receives the remaining budget |
| 6 | Identity, authentication, authorization | Non-exempt methods are rejected before authentication |
| 7 | `RetryPolicy` + `idempotency_key`, compression, client/bidi streaming, credit flow control | §20, §5.5, §5.6 |
| 8 | Reflection | An outside tool with no access to the service's source lists its methods and calls one §21 |

The acceptance criterion for the roadmap as a whole: an incompatible schema change is rejected at connect time, and the bytes produced are identical across two SDKs.

### 15.1 What must be fixed before the second implementation

| Category | Items | Reason |
|---|---|---|
| Normative, affects interoperability | Three levels of abnormality §5.9 · call id rules · compression/ctx boundary §11 · header and ctx block layout · allowance states and the rule on items beyond the allowance §5.6 · CREDIT is never lost to a queue ceiling §7.5 | These determine how two sides interpret the same byte sequence. Leaving them to each SDK loses interoperability |
| Implementation quality, no interoperability constraint | Scheduler §7.1 to §7.4 · two driving modes §19 · `RetryPolicy` §20 · stream idle limit §5.4 | A simple FIFO implementation interoperates with a scheduled one; the bytes are identical. Verified by behavioural tests |

Consequence: §5.9, §5.6 and §11 are normative even for an implementation that enables neither compression nor flow control, because the peer may still send a compressed frame or a CREDIT. §7 can be delivered incrementally.

---

## 16. Minimum test checklist

Following the model of RFC-0003: an implementation is considered correct when it passes all of the following. The tests are language independent.

### Format
- Encode then decode each kind, matching §4.9 byte for byte
- The header is exactly 5 bytes, including for END and CANCEL
- `kind` = 7 → ERROR for the call; the session stays alive
- Non-zero reserved bit → ERROR for the call; the session stays alive
- Wrong message id for the `(method, kind)` pair → `INVALID_ARGUMENT`; the session stays alive

### Registration
- Two methods declaring the same request type → startup error
- A type in the fRPC id set colliding with a plain message id → startup error
- A message id in the declared plain-message set → delivered to the application, first 5 bytes not read as an fRPC header
- A REQUEST whose id is in neither set → `UNIMPLEMENTED` at once; the application receives nothing
- A frame whose id is in neither set and whose header is broken → dropped with no reply; the application receives nothing

### ctx block
- Bit 6 clear → the request model starts at byte 5, with no length prefix
- Bit 6 set, sender's `FRpcCtx` has more fields than the receiver knows → the request model still decodes correctly
- ctx length above 64 KiB → error before allocation
- Bit 6 set on a kind other than REQUEST → `INTERNAL` for the call; the session stays alive
- `deadline_ms` across three hops A→B→C: the value C receives is smaller than the value A sent
- `deadline_ms = 0` → no deadline set §3.4

### Authentication
- Session not authenticated, calling a non-exempt method → `UNAUTHENTICATED`
- Session authenticated, `ctx.token` carrying a different principal → `PERMISSION_DENIED`; the handler does not run; the session identity does not change §10.4
- The effective permissions of a delegated call are the intersection of the two permission sets
- With the network to the token system cut: calls are answered from cache or return `UNAVAILABLE`; the tick loop does not stop §10.1

### Calls
- Call id parity matches the role
- Reusing a pending call id → local error; nothing reaches the wire
- RESPONSE for a call id no longer pending → ignored
- REQUEST carrying a type with no handler → `UNIMPLEMENTED`
- Pending ceiling reached → `RESOURCE_EXHAUSTED`; other calls unaffected
- REQUEST reusing a running call id → ignored; the original call continues
- A call refused locally after the caller chain ran → the chain unwinds exactly once
- Wrong call id parity → `INVALID_ARGUMENT`; the session stays alive
- `FRpcError` fails to decode → the call completes with `INTERNAL`

### Three levels of abnormality
- RESPONSE for a call id no longer pending → ignored, no error raised
- ITEM after END → ignored
- CANCEL for a finished call → ignored
- Every fRPC protocol violation → the session stays alive and other calls continue

### Queues and scheduler
- Call A enqueues 1,000 ITEMs, call B enqueues a REQUEST afterwards → B does not wait for all of A
- A call sending large frames and a call sending small frames share the link by bytes, not by frame count
- A cancelled call → its queued frames are discarded; only the terminating frame is sent
- Cancelling while the REQUEST is still queued → no bytes reach the wire; the call completes with `CANCELLED`
- One call's queue full: below the threshold → the producer waits for the next tick; above it → the stream terminates with `RESOURCE_EXHAUSTED`
- Per-call ceiling reached → only that call is congested
- Connection ceiling reached → the send call returns congestion; the session stays alive

### Bidirectional streaming and credit
- Client stream: caller ITEMs reach the handler in order; RESPONSE follows only after the caller's END
- A caller ITEM carrying the wrong message id → `INVALID_ARGUMENT`, the call terminates
- Sending a further item after the caller direction is closed → a local error, no bytes reach the wire
- Bidi: the two directions interleave and each END is independent
- `initial_credit = 0` → the reply direction is not established; the producer runs unbounded
- `initial_credit = n` → exactly n items are produced, then it stops with the call still open
- CREDIT arrives → exactly the granted number of items follow, numbered continuously with the previous run
- `initial_credit = 0`, then CREDIT n → the reply direction becomes established; the producer stops after n items
- Client stream, the callee sends CREDIT n in the tick that delivers the REQUEST → the caller stops after n caller-direction items
- Items beyond the allowance arrive → delivered as ordinary items, no error raised
- CREDIT for a closed call → ignored
- A sustained allowance of zero → does not trip the `RESOURCE_EXHAUSTED` threshold of §5.4
- CREDIT sent to a peer that does not declare `FRpcCredit` → ignored; the call keeps running unbounded and the application receives nothing
- CREDIT carrying a message id other than `FRpcCredit` → `INVALID_ARGUMENT` for the call; the session stays alive
- A call's queue is full and further CREDIT is granted → no congestion error; two consecutive grants reach the wire as one CREDIT carrying the sum

### Compression
- Bit 7 set together with bit 6: `[length][FRpcCtx]` is readable without decompression
- A body whose declared expansion exceeds the ceiling → error before allocation

### Time
- A full deadline cycle runs by injecting `now`
- RESPONSE and deadline in the same tick → RESPONSE takes precedence (§6)
- Unary, CANCEL never arrives → the callee still terminates the handler by its own limit
- A stream with `deadline_ms = 0`, silent in both directions for longer than the unary handler limit → the call stays open
- The wait step while net is congested → no spinning; the loop wakes when the transport becomes writable or data arrives

### Retry
- An `unsafe` method receives `UNAVAILABLE` → no retry
- A `keyed` method: retries carry the same `idempotency_key`; two operations with different ids carry different keys
- A retry draws from the deadline budget of the first attempt

### Termination
- DISCONNECT with N pending calls → exactly N completions, one per call
- HANDSHAKE FAILED → as above, with no second cleanup
- A handler running when the session ends → receives the cancellation signal; no frame is sent

### Reflection
- The response lists every method the callee serves, with its shape and message ids
- `schema_json` matches the embedded file byte for byte
- An unauthenticated session calling `FRpc.Reflect` → `UNAUTHENTICATED`, unless the operator has added the method to the exemption list
- A greeting holding only the models reflection needs → the handshake accepts it and reflection answers
- A server without reflection, whether or not its schema declares the reflection models → `UNIMPLEMENTED` at once; the application receives nothing

### Interoperability
- Client in language A against server in language B, unary and streaming, matching byte for byte
- One side appends a field to a request model → the handshake accepts and the call still runs (RFC-0002 §9.1)

---

## 17. Alternatives considered and rejected

| Alternative | Reason for rejection |
|---|---|
| Envelope model: one message id for all RPC traffic, with the body as an inner `Bytes` field | The handshake would verify only the envelope, not any request or response model. The envelope has a fixed `n`, so appending a field to the body changes the fingerprint while `n` stays the same; this is branch ⓑ of `02-flows` §3.3, which rejects the session. Version skew would no longer apply |
| The RPC header as the leading fields of every model | Keeps the body a complete canonical model, but requires codegen to emit a wrapper model per method and to duplicate types where two methods share a request. The header is layer framing rather than business data, and net has precedent for framing outside the schema (`'F' 'O'`, length) |
| A 4-byte `method id` in the header | Buys only the ability for two methods to share a request type, at 4 bytes on every frame repeating information the frame already carries. Sharing a request type couples the two methods during evolution (§3.3) |
| The deadline in the header | 4 bytes on every frame, including kinds with no deadline semantics. The callee still needs its own limit (§5.3) |
| A free-form key/value metadata map | Undeclared data, absent from the schema, unverifiable by the handshake; it forfeits the property in the first row of §1.1. The corresponding need is met by the declared ctx block (§3.4) |
| `FRpcCtx` as the leading field of every request model | Puts every method into version skew simultaneously whenever ctx changes (§4.11) |
| The deadline as an absolute timestamp | Meaningful only if the two machines are time-synchronised. Fomoxa forbids wall-clock reads at every layer (B9) |
| A dedicated kind for an fRPC handshake | Duplicates an ordinary method invoked after READY, which already has schema, interceptors, deadlines and error codes |
| Prioritising ERROR/CANCEL within one call | Breaks causality with the data it terminates. Slow cancellation is addressed by discarding the terminated call's queue (§7.3) |
| One FIFO for the whole connection | A large stream blocks every other call (§7.1). The prohibition on reordering applies only within one call |
| CREDIT only adds; an allowance that is not established stays unlimited forever | The caller direction of client and bidi streams could never be limited, so a large upload has no back pressure (§5.6) |
| The caller direction starts at an allowance of zero and waits for CREDIT | A callee without flow control never sends CREDIT, so the call hangs until its deadline. Compatibility is lost (§5.6) |
| `initial_credit = 0` as an allowance of zero | Every caller that uses the ctx only for a deadline or a trace, and every sender of an older `FRpcCtx`, encodes 0; their streams would stall from the start. Contradicts the "unset" convention of §3.4 |
| A mandatory total limit for streams | Healthy watch streams are cut periodically. The reason for a mandatory limit is that CANCEL can be lost, which happens only on packet transports, where streams are undefined (§5.4, §5.7) |
| A random idempotency key per call | Loses the property that every SDK derives the same key. An operation id in the request model keeps determinism (§20.4) |

---

## 18. Open decisions

Points where the design chooses a direction without sufficient evidence:

1. 32-bit call ids (§4.7). Sufficient for the scenarios examined; the wrap-around rule must be implemented correctly on both sides.
2. 65,536 pending calls per direction (§4.8). Requires data from real load.
3. The `FRpcCtx` field set (§3.4). The current seven fields are an estimate. Appending is a valid upgrade, removal is not, so under-declaring is safer than over-declaring. `idempotency_key` and `initial_credit` were appended after the first revision, by exactly that route.
4. Parameterless methods require their own empty models (§3.3). Phase 4 addresses this through codegen. If a material cost remains afterwards, the place to change is codegen, not the wire format.
5. Unary over packet transports (§5.7). The design accepts possible duplication at the callee and leaves deduplication to the application. If every application has to implement it, it belongs in fRPC.
6. Compression fixed by configuration (§11). Workable when both sides are operated by the same party; not workable with a third party.

---

## 19. Two driving modes

§8 requires handlers not to block the tick loop. That constraint is only exposed in the API when the tick loop *is* the API.

Net already separates its pure state machine (which takes a frame and `now` and returns intent) from the glue that touches the transport. fRPC applies the same separation one layer up:

```
   PURE fRPC CORE          takes events + now, returns frames and call results
                           no threads, no clock, no I/O
        │
   ─────┼──────────────────────────────────────────────
        │
   SHELL ├─ app-driven   the application calls tick(now) in its own loop
         └─ self-driven  fRPC owns a thread and runs the loop itself
```

| | App-driven | Self-driven |
|---|---|---|
| Who calls `tick(now)` | The application | fRPC's thread |
| Handlers | Run sequentially on the driving thread | May block, on a worker |
| Caller API | Result available on a later tick | Blocks the calling thread until completion |
| Suited to | Deterministic tests, C and Rust without a runtime, applications with their own loop | Ordinary services |

The prohibition on spawning background threads in `01-overview` §12 appears in the table for transports and does not apply above the core. The applicable constraint is the sentence accompanying it: one session is driven by one thread. The self-driven mode satisfies that constraint while it observes five rules:

```
1. Exactly one thread touches the session. Everything entering is a
   message on a BOUNDED queue (§4.8), not a direct call.

2. The pure core MUST remain callable with an injected `now`. The
   self-driven shell uses a real clock; abandoning B8 removes the
   ability to run a full deadline cycle in tests.

3. Decode the body on the driving thread and hand only owned values
   across the queue. Passing a borrowed byte slice to another thread
   reads memory that is being overwritten (§2 C7).

4. Parallel handler execution MUST be declared per method, never by
   default. In sequential mode a handler needs no lock when touching
   shared state; enabling parallelism by default introduces races
   into code not written for them.

5. Each session carries a generation number. A handler result returning
   after the session ended MUST be discarded, not sent (§5.8).
```

### The wait step between two ticks

The tick loop fixes the order of work inside one tick. It does not fix the interval between two ticks. The shell chooses that interval, and the choice determines two quantities at once: the latency of a call and the CPU cost of an idle connection.

A fixed cadence degrades both together: a short cadence spends CPU while idle, a long one adds half a period to every hop of a call chain. A shell SHOULD NOT use a fixed cadence when the transport can signal readiness.

Instead the shell blocks until one of these events: data arrives, the transport becomes writable again, the connection closes, or the nearest instant the core is waiting on has come. Step 5 of the tick drains the send queues until they are empty or net reports congestion (§6), so after a tick any frame left in a queue always means net is congested. The wait conditions derive from the core:

```
   frames waiting to send (net congested) → wait for readable OR writable
   a handler not yet finished             → wait at most a short interval, to poll it again §8
   a deadline or idle limit running       → wait at most until the nearest one §5.3, §5.4
   nothing outstanding                    → wait at most until the idle ceiling
```

The rows combine: the wait ends at the earliest instant among the rows that apply. Waiting for readable is always present, including while waiting for writable; a side that only waits to write without reading can end up waiting on the other side's buffer while the other side waits on its own. A transport that cannot report writability replaces that condition with a short fixed interval. No row leads to a loop that does not wait: such a loop occupies a whole CPU core for as long as net stays congested.

The idle ceiling MUST be shorter than the net heartbeat interval, because probes are sent in step 3 of the net tick (`01-overview` §6).

The prohibition on blocking in `01-overview` §12 stands unchanged: a transport's four functions (send, receive, soft close, hard close) MUST NOT block. The wait step is a fifth function, called by the shell between two ticks, never by net. A transport that adds it still satisfies §12.

Thread count is shell policy, not protocol. One thread per connection allows blocking directly on that connection's socket and needs no multiplexing mechanism; its practical ceiling is in the low thousands of connections per process. Beyond that ceiling a shell uses a multiplexing reactor. Both produce identical bytes and leave the core unchanged.

Absolute wire order (`01-overview` B7) applies per connection: each connection MUST have exactly one writing thread. Parallelism lies between connections, not inside one.

Both modes produce identical bytes on the wire: no new kind, no new bit, no handshake change. They therefore belong to each language's API binding document rather than to the fRPC specification, following the division stated in `01-overview`: those documents describe how to call, not how the protocol behaves.

The app-driven mode MUST be fully supported. Requiring a runtime in order to run fRPC limits the number of languages that can reconstruct it, which conflicts with RFC-0001 §6.5.

One limit to record: the tick loop signals cancellation; it does not stop a running handler. The callee's processing limit (§5.3) informs a handler that it has expired; it does not stop a loop running on a worker. A handler that does not read the cancellation signal runs until it finishes on its own.

---

## 20. Retry

### 20.1 The core does not retry

`01-overview` §7 states that Fomoxa does not retransmit. fRPC keeps that rule at the RPC layer:

```
   client ── Purchase() ──> backend
                             transaction succeeded
                             response sent
                        X    connection ended
   client receives UNAVAILABLE
        → cannot determine whether the backend executed
```

The core has no information with which to decide whether a retry is safe, so it does not decide. When the connection ends, every pending call completes with `UNAVAILABLE` (§5.8).

### 20.2 The client stub provides retry

A long-lived connection (§1.2) still ends on a deploy or a network fault, and each time every pending call completes with `UNAVAILABLE`. Retry belongs to the client stub, above the core:

```
   application ── supplies connections, decides on reconnecting
      │
   RpcClient   ── RetryPolicy: attempts · backoff · retryable codes · budget
      │
   fRPC core   ── implements no retry
```

The stub retries on a connection the application supplies. When the old connection has ended, the stub asks the application for one, and the application decides whether to open a new connection or return an error. The stub does not reconnect by itself, per §13.1.

### 20.3 Four rules

A retry MUST draw from the deadline budget. Retrying with a fresh deadline makes the propagated `deadline_ms` of §5.3 meaningless at every later hop. The budget belongs to the whole retry sequence, not to each attempt.

Each method declares its retry safety in the descriptor (§12):

| Level | Meaning | Example |
|---|---|---|
| `unsafe` | The default. Running twice gives a different result from running once | `Purchase` with no operation id |
| `idempotent` | Running twice leaves the same state as running once, by the nature of the operation | Reads; `SetStatus(user, status)` |
| `keyed` | The operation has side effects, and the callee deduplicates by `idempotency_key` (§20.4) | `Purchase(order_id, item)` |

The default is `unsafe` because a wrong declaration in that direction only loses a retry, while a wrong declaration in the other direction runs a transaction twice.

The error code decides whether a retry is worth attempting:

| Code | Retryable | Reason |
|---|---|---|
| `UNAVAILABLE` | Only for `idempotent` or `keyed` methods | The connection ended, and whether the callee executed cannot be known §20.1 |
| `RESOURCE_EXHAUSTED` | Yes, with backoff | Temporary overload |
| `DEADLINE_EXCEEDED` | No | The budget is spent |
| `INVALID_ARGUMENT`, `PERMISSION_DENIED`, `UNAUTHENTICATED`, `UNIMPLEMENTED`, `FAILED_PRECONDITION` | No | Resending the same data yields the same result |
| `INTERNAL` | Policy dependent | Insufficient information to conclude |
| `CANCELLED` | No | The caller cancelled |

Streams MUST NOT be replayed automatically. A stream that breaks after the caller consumed N items would repeat those N items if replayed from the start. Resuming from the break requires a continuation cursor, which is an application-level concept.

### 20.4 Idempotency

`UNAVAILABLE` does not determine whether a transaction executed. The mechanism that resolves this is an idempotency key:

```
   key = hash of the encoded body
```

Because the bytes are deterministic (RFC-0001 §6.1), every SDK in every language derives the same key with no additional agreement. An RPC system built on a non-deterministic format does not have this property.

A key hashed from the body is correct only when the same body means the same operation. Two deliberate purchases of the same item have identical bodies, and the second would be filtered as a retry. The request model of a `keyed` method therefore MUST contain an operation id chosen by the caller, for example `order_id`. Retries of one operation carry the same id and so the same key; two different operations carry two ids and so two keys. The distinction lives in declared data, not in a hidden value outside the schema (§3.4).

An `idempotent` method needs no key. The stub sets `idempotency_key` only for `keyed` methods.

`idempotency_key` is the sixth field of `FRpcCtx` (§3.4), appended after the first revision together with `RetryPolicy`. A receiver that does not know the field stops at field five and treats the remainder as valid trailing bytes under RFC-0002 §9.1; the call still runs, it only loses deduplication.

---

## 21. Reflection

Reflection lets an outside tool, such as a command-line client, learn a server's methods and models without access to that service's source.

The handshake cannot provide this. A client never sees the server's schema (`02-flows` §3.1), and even if it did, the greeting carries only a message id, a field count and one 8-byte fingerprint per message. A fingerprint is a one-way hash; field names and types cannot be recovered from it. Encoding a request and decoding a response takes the model definitions themselves.

### 21.1 Method and models

```
   FRpc.Reflect      unary      FRpcReflectRequest → FRpcReflectResponse

   FRpcReflectRequest
     (no fields)

   FRpcReflectResponse
     methods          Array<FRpcMethodInfo>
     schema_json      Bytes

   FRpcMethodInfo
     name             String     "Player.Get"
     shape            UInt32     0 unary · 1 server stream · 2 client stream · 3 bidi
     request_id       UInt32
     caller_item_id   UInt32
     reply_id         UInt32     the response, or the reply-direction item
     retry_safety     UInt32     0 unsafe · 1 idempotent · 2 keyed (§20.3)
```

These three are system models: declared with the codec `rpc` (§4.5), and their model names, field names, types and order MUST be exactly as above. The fingerprint hashes field names, so a single differing name leaves a tool unable to compute the right fingerprint.

### 21.2 Rules

- Reflection is optional. A server that enables it MUST serve exactly the layout of §21.1. A server without it answers `UNIMPLEMENTED`, as it does for any REQUEST it does not serve (§4.10, §5.2), whether or not its schema declares the reflection models. The frame never reaches the application layer.
- `methods` lists every method the callee serves, `FRpc.Reflect` included. A method declared only for calling out, and not served, is not listed.
- `shape` and `retry_safety` are declared `UInt32`, not Enum, for the same reason as error codes (§2 C13). A tool that meets an unknown value displays the raw number.
- `caller_item_id` equals `request_id` when the method declares no separate caller-direction item type (§5.5).
- `schema_json` is the content of the schema file the code generator wrote for the very binary that is running (for `fomoxac`, `.fomoxa/schema.json`), UTF-8, verbatim. fRPC does not interpret it. The file SHOULD be embedded in the binary at build time so it always matches the running code.
- The response does not change for the life of the process. An implementation SHOULD encode it once and return the same bytes each time.
- `FRpc.Reflect` passes through the interceptor chain like any method, and MUST NOT be on the authentication exemption list by default (§10.1). The response exposes the name and type of every field; placing it outside authentication is the operator's decision.
- `schema_json` is indented, at about 860 bytes per model, so a large schema exceeds the 64 KiB recommendation of §7.2. That is acceptable for a call made rarely. An implementation SHOULD enable compression for this response (§11).

### 21.3 A tool's flow

```
   connection 1   greeting: FRpcVoid, FRpcError, FRpcReflectRequest, FRpcReflectResponse
                  the tool computes their ids and fingerprints from §21.1
                  → calls FRpc.Reflect → receives methods + schema_json

   connection 2   greeting: the whole schema just learned
                  → the handshake verifies every shared message
                  → calls the real method
```

Connection 1 is accepted even though its greeting is a small part of the server's schema: gate ③ only considers messages both sides have (`02-flows` §3.3). Differing schema fingerprints are not a reason to reject; the schema fingerprint is only a shortcut for when both sides match entirely.

A tool MAY store the learned schema and reuse it on later runs, skipping connection 1. If the server changes its schema afterwards, the handshake of the next connection detects it: a change that is still compatible is accepted under RFC-0002 §9.1, and a real conflict is rejected with reason 2. The tool then has to load the schema again.

Reflection adds no kind, bit or header field. It is an ordinary fRPC method, so R1 and R2 hold unchanged.
