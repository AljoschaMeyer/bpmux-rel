# bpmux/rel

Specification for providing the [bpmux abstractions](https://github.com/AljoschaMeyer/bpmux) over an ordered, reliable, bidirectional communication channel, such as tcp or unix domain sockets.

## General

Bpmux/rel asigns 64 bit identifiers to all entities, so that packets can be "routed" to the correct entities. It also keeps track of whether an entity was created by the endpoint itself (*out-entities*) or by the peer (*in-entities*), to prevent trouble when both parties create entities with the same identifier concurrently.

Data gets transmitted in chunks. Each chunk begins with a 1-byte *tag*, followed by a [VarU64](https://github.com/AljoschaMeyer/varu64-rs) encoding the identifier of an identity, followed by additional bytes whose interpretation depends on the tag. The tag consists of three bits for the *type*, followed by five bits of *flags*.

The type indicates what kind of entity the identifier following the tag refers to:

| bits | type        |
|------|-------------|
| 000  | Out-Request |
| 001  | In-Request  |
| 010  | Out-Sink    |
| 011  | In-Sink     |
| 100  | Out-Stream  |
| 101  | In-Stream   |
| 110  | Duplex      |
| 111  | Partial     |

The flags are (from fourth-most-significant to least significant bit):

- `payload`
- `credit`
- `ping`
- `end`
- `ack-end`

Unless noted otherwise:

- identifiers are always unsigned 64 bit integers, and when transmitted, they are encoded as [VarU64](https://github.com/AljoschaMeyer/varu64-rs)s
- payloads are byte strings of length up to 2^64 - 1, and are encoded as a VarU64 followed by that many bytes
- credits are always unsigned 64 bit integers, and when transmitted, they are encoded as VarU64s
- end-payloads work just like regular payloads, except they are indicated by the `end` flag
- if multiple flags among `payload`, `credit` and `end` are set, the chunk contains the corresponding data in that order
- protocol violations should be handled by immediately terminating the connection

## State

The state of an endpoint consists of a `State` as defined in the following code block:

```
type Entity = {
  credit_out: U64,
  credit_in: U64,
  end_out: bool,
  end_in: bool,
  partial: U64,
}

type State = {
  req_out: Map<U64, Entity>,
  req_in: Map<U64, Entity>,
  sink_out: Map<U64, Entity>,
  sink_in: Map<U64, Entity>,
  stream_out: Map<U64, Entity>,
  stream_in: Map<U64, Entity>,
}
```

The initial state consists of empty maps, except for sink_out and stream_out, both containing an entity with id 0, representing sink and stream halves of the top-level context. Initially, `credit_out`, `credit_in` and `partial` are set to zero, and their end flags are set to false.

## Receiving Chunks

This section describes how to update the state upon sending chunks with a type other than `Partial`. `Partial` chunks are described in a later section.

The `type` of the chunk indicates which part of the state to operate upon (note that this is mirrored from what you might expect, since the type was set from the peer's perspective):

| Chunk type  | Map of entities  |
|-------------|------------------|
| Out-Request | state.req_in     |
| In-Request  | state.req_out    |
| Out-Sink    | state.stream_in  |
| In-Sink     | state.stream_out |
| Out-Stream  | state.sink_in    |
| In-Stream   | state.sink_out   |
| Duplex      | special handling |

Let `ent` be the result of looking up the chunk's identifier from the appropriate map in the state.

If `ent` is not in the map, the identifier is zero, and the chunk type is one of `In-Sink`, `In-Stream` or `Duplex`: This is a protocol violation (you can't reopen the top-level context).

Otherwise, if `ent` is not in the map:

- if the chunk is of type `In-Request`, this is a protocol violation
- create a new entry in the appropriate map with the chunk's identifier, with credits and `partial` set to zero, and the end flags set to false. `ent` now refers to the newly created entity.
  - if the chunk is of type `Duplex`:
    - if either the `sink_in` or the `stream_in` maps contain the identifier, this is a protocol violation
    - else, create new identifies in both maps. `ent` now refers to the entity created in the `sink_in` map
- if the `payload` flag is not set, this is a protocol violation
  - else, subtract the payload length from `ent.credit_in`
    - if an overflow occurs, this is a protocol violation
    - if no overflow occurs, forward the payload to the application layer
- if the chunk has the `credit` flag set, add the credit to `ent.credit_out`
  - if an overflow occurs, this is a protocol violation
- if the chunk has the `ping` flag set, forward the heartbeat ping to the application layer
- if the chunk has the `end` flag set, subtract the end payload length from `ent.credit_in`
  - if an overflow occurs, this is a protocol violation
  - if no overflow occurs:
    - forward the end payload to the application layer
    - set `ent.end_in` to true
    - send back an acknowledgement chunk (see below)
- if the chunk has the `ack-end` flag set, this is a protocol violation

If `ent` has already been in the map:

- if the chunk has type `Duplex`, this is a protocol violation.
- if the chunk has the `payload` flag set:
  - if the chunk is not of type `In-Request`, `In-Sink` or `Out-Sink`, this is a protocol violation
  - subtract the payload length from `ent.credit_in`
    - if an overflow occurs, this is a protocol violation
    - if no overflow occurs, forward the payload to the application layer
  - if the chunk is of type `In-Request`, send back an acknowledgement chunk (see below)
- if the chunk has the `credit` flag set, add the credit to `ent.credit_out`
  - if an overflow occurs, this is a protocol violation
- if the chunk has the `ping` flag set, forward the heartbeat ping to the application layer
- if the chunk has the `end` flag set:
  - if the chunk is of type `In-Request` and also has the `payload` flag set: this is a protocol violation
  - if `ent.end_in` is already set to true, this is a protocol violation
  - subtract the end payload length from `ent.credit_in`
    - if an overflow occurs, this is a protocol violation
    - if no overflow occurs:
      - forward the end payload to the application layer
      - if `ent.end_out` is set to true, remove the entity from the map
      - else:
        - set `ent.end_in` to true
        - send back an acknowledgement chunk (see below)
- if the chunk has the `ack-end` flag set:
  - if any other flag is also set, this is a protocol violation
  - if `ent.end_out` is set to false, this is a protocol violation
  - remove the entity from the map
- if no flags were set, the chunk does nothing. It should still be reported for heartbeat purposes (effectively, this is how the protocol encodes heartbeat pongs)
- there is some special handling around receiving `In-Request` chunks:

To send an acknowledgment chunk, send a chunk with the same identifier, no flags but the `ack-end` flag, and with the `type` derived by inverting "In" and "Out" as well as "Sink" and "Stream" (so that the peer matches it to the correct entity).

## Sending Chunks

To send a chunk, do the inverse of receiving a chunk. Send the data that makes the peer do what you want, according to the procedure of receiving chunks. In particular, don't do anything that results in protocol violations.

When sending a chunk, update entities as follows:

- if the `payload` flag is set, subtract the size of the payload from `ent.credit_out`
  - if there is an overflow, you violated the protocol, don't do this
- if the `credit` flag is set, add the amount of credit to `ent.credit_in`
  - if there is an overflow, you violated the protocol, don't do this
- if the `end` flag is set, subtract the size of the end-payload from `ent.credit_out`
  - if there is an overflow, you violated the protocol, don't do this
  - set `ent.end_out` to true

## Partial Mode

Chunks of type `Partial` work completely different then regular chunks. They are not followed by a VarU64 identifier, and the five remaining bits are not treated as flags. Instead, the two middle bits are ignored, and the three least significant bits are treated as an unsigned integer `length` between zero and seven. The tag is followed by `length + 1` many bytes encoding an big-endian unsigned integer, the `total_size`. If the `total_size` is zero, this is a protocol violation. If the the `total_size` is encoded in more bytes then needed (e.g. the value 42 encoded in two bytes rather than one byte), this is a protocol violation.

The entity addressed by the chunk following the `Partial` chunk is put into *partial mode*, by setting the (possibly newly created) entity's `partial` value to the `total_size`. When receiving payloads for that entity, the payload is not directly handed to the application layer. Instead, the payload gets buffered and its size is subtracted from the `partial` value. If an overflow occurs, this is a protocol violation. Only when the `partial` value reaches exactly zero is the concatenation of all buffered payloads for that entity passed to the application layer. Interleaving regular payload and end payload on the same entity in partial mode is a protocol violation.
