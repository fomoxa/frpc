# fRPC: the RPC layer for Fomoxa

English · [Tiếng Việt](vi/README.md)

fRPC is an RPC layer placed on top of Fomoxa. This repository describes it at the conceptual level and at the byte level, without an SDK, an API or source code.

Status: design document. It is neither a specification nor an implementation guide yet. Section 18 lists the decisions still open, and §15.1 lists what must become normative before a second implementation exists.

## Document map

| Document | Content | Length |
|---|---|---|
| [en/01-design.md](en/01-design.md) | Position in the architecture, inherited constraints, wire format, flows, tick cycle, queues, authentication, invariants, roadmap, test checklist | ~1,100 lines |

The `vi/` directory holds the Vietnamese version. Other translations, if any, live in their own language-code directory.

Prerequisites, both in separate repositories:

| Repository | What fRPC uses from it |
|---|---|
| `specification` (RFC-0001, RFC-0002, RFC-0003) | Model encoding, version skew §9.1, determinism, the Enum rule |
| `implementation-guide` (`01-overview.md`, `02-flows.md`) | The three layers, the DATA frame, the fingerprint handshake, the tick cycle, the single pending frame slot, event data lifetime |

## Boundary with fomoxa-net

fRPC is a separate repository. It is outside fomoxa-net, is not an extension branch of it and does not appear on its roadmap. It sits on top of the specification and consumes a net implementation the same way an application does.

The boundary has one test:

> If fRPC requires a change inside fomoxa-net, the logic is being placed in the wrong layer.

Everything fRPC needs lives in the payload of the DATA frame, which the implementation guide defines as opaque bytes to the frame layer. Net adds no frame type, header byte or handshake field on fRPC's behalf.

## Read by task

| What you want | Sections to read |
|---|---|
| Understand where fRPC sits and what it inherits | §0–§2 |
| Only need the bytes on the wire | §4 |
| Implement dispatch and the unary path | §3.3, §4.4, §4.10, §5.1, §5.2 |
| Implement deadlines, cancellation and propagation | §5.3, §6 step 3 |
| Implement streaming | §5.4, §7 |
| Understand queueing, congestion and scheduling | §7 |
| Write handlers | §8, §19 |
| Implement authentication and authorization | §9, §10 |
| Implement compression | §11, §4.11 |
| Verify an implementation | §16, then §14 |
| Understand why a common RPC design choice was not used | §17 |

## One-page summary

Net answers which message type a sequence of bytes belongs to. fRPC adds two questions:

```
   net    → which message type do these bytes belong to
   fRPC   → which call do they belong to, and which component handles them
```

Everything travels inside the DATA payload, behind a fixed 5-byte header:

```
   ┌──────┬──────────────┐
   │ kind │ call id      │      bit 7  body compressed
   │ 1B   │ 4B u32 LE    │      bit 6  ctx block present
   └──────┴──────────────┘      bit 2-0 kind (0..6)
```

There are seven kinds: REQUEST, RESPONSE, ERROR, ITEM, END, CANCEL, CREDIT. Fixed overhead per message is 16 bytes, 11 from net plus 5 from fRPC.

The wire carries no method identifier. A method is identified by the message id of its request model, so the whole design depends on one rule: two methods must not share a request type.

There is no key/value metadata map. Cross-cutting information travels in `FRpcCtx`, a declared, optional, length-prefixed model that carries the remaining deadline, trace context, token and tenant.

Sitting on Fomoxa gives fRPC the following properties, and fRPC must preserve them:

- Net's handshake rejects schema disagreement at connect time, with a reason code.
- Version skew (RFC-0002 §9.1) applies per request type, so appending a field is a valid change per method.
- Bytes are deterministic. Payload signing, content-derived idempotency keys and replay therefore work without additional agreement between implementations.

A server can serve an optional reflection method, `FRpc.Reflect` (§21), which returns its method list and the schema file embedded at build time. An outside tool uses it to learn the models it needs, because the handshake never shows a client the server's schema.

The invariants every implementation must hold are in §14. Two of them are the most frequently broken: an fRPC-layer error terminates one call and never the session, and no check on a call's processing path may leave the process.

## Out of scope

- Message encoding and fingerprint computation belong to the `specification` repository.
- Framing, handshake, heartbeat and the transport contract belong to `implementation-guide` and to net.
- Per-language API bindings, including the choice between the app-driven and self-driven modes of §19. Each implementation picks its own names and call shapes. Both modes produce identical bytes, so the choice is outside the protocol.
- Retry policy, backoff and connection re-establishment belong to the client stub above the core (§20).

## License

CC BY 4.0, see [LICENSE](LICENSE). Software implementations are independent projects and pick whatever license their authors choose.
