# bpmux/rel

Specification for providing the [bpmux abstractions](https://github.com/AljoschaMeyer/bpmux) over an ordered, reliable, bidirectional communication channel, such as tcp or unix domain sockets.

bpmux/rel assigns 32 bit ids to requests, sinks, streams and duplexes, so that payloads can be "routed" to the correct "destination". A single endpoint has full control over the ids it assigns, it does not need to take into account any ids that have been assigned by the other endpoint. The only requirements are:

- An endpoint may not send a request with an id which has been used on a previous request from this endpoint that has been neither responded to, nor cancelled, nor closed.
- An endpoint may not create a sink with an id of any other currently open sink/stream/duplex that has been opened by this endpoint.
- The same holds for streams and duplexes.

bpmux/rel transmits data in chunks. Each chunk is preceded by a tag that indicates how the following bytes are to be interpreted, and how many of the following bytes to interpret. The first four bits of the tag indicate its *type* (all types are listed in the following section). The remaining four bits are split up into the named arguments in the parentheses after the name of the type. Most of the time, these arguments do not store data, but only indicate how many of the following bytes contain the actual data.

There are various ways of sending invalid chunks, including but not limited to: Disrespecting the credit limit, responding to nonexistent requests, sending messages down nonexistent sinks, acknowledging nonexistent cancellations/closings, etc. A conforming implementation can not produce invalid chunks accidentally. Whenever an endpoint receives an invalid chunk, it should abort the connection (without signaling any errors or closing and cancelling the top-level).

Conceptually, each chunk maps to a concept from the bpmux specification. There are chunks to open sinks/streams/duplexes, there are chunks to send messages/requests/responses, there are chunks for giving credit and sending/answering heartbeat signals, and there are chunks to cancel and close things. To avoid id ambiguity, there are specialized chunks for dealing with sinks/streams created by this endpoint and for dealing with sinks/streams created by the peer. And finally, there is the need to acknowledge cancellations and closings to avoid race conditions.

To partition data into arbitrarily small batches, there is a mechanism to put any payload-carrying chunk into *partial* mode. When entering partial mode, the total size of the payload is specified. All further matching chunks then contribute to the payload until it has been fully sent, at which point partial mode is automatically disabled.

## Chunk Types

### `Sink(id: uint2_t, len: uint2_t)`, type nibble: 10 (`0b1010`)

Open up a new sink.

If `id != 3`, the `2 ^ id` bytes following the tag are an integer that serves as the id of the sink.

The `2 ^ len` bytes following the sink id bytes are an unsigned integer indicating the length of the payload.

After the `len` bytes, place that many bytes of payload. This consumes as much top-level credit.

If `sink == 3`, the chunk acknowledges a cancellation on a stream opened by the peer. The `2 ^ len` bytes following the tag specify the id of the stream on which to acknowledge the cancellation. `len` may not be 3.

### `Stream(id: uint2_t, len: uint2_t)`, type nibble: 11 (`0b1011`)

Open up a new stream.

If `id != 3`, the `2 ^ id` bytes following the tag are an integer that serves as the id of the stream.

The `2 ^ len` bytes following the stream id bytes are an unsigned integer indicating the length of the payload.

After the `len` bytes, place that many bytes of payload. This consumes as much top-level credit.

If `stream == 3`, the chunk acknowledges the closing of a sink opened by the peer. The `2 ^ len` bytes following the tag specify the id of the sink on which to acknowledge the closing. `len` may not be 3.

### `Duplex(id: uint2_t, len: unint2_t)`, type nibble: 12 (`0b1100`)

Open up a new duplex.

If `id != 3`, the `2 ^ id` bytes following the tag are an integer that serves as the id of the duplex.

The `2 ^ len` bytes following the duplex id bytes are an unsigned integer indicating the length of the payload.

After the `len` bytes, place that many bytes of payload. This consumes as much top-level credit.

If `id == 3`, the chunk is a heartbeat ping for a request. The `len ^ 2` bytes following the tag specify the id of the request for which to send the ping. `len` may not be 3.

### `Msg(sink: uint2_t, len: uint2_t)`, type nibble: 6 (`0b0110`)

Send a message down a sink that was opened by this endpoint (or the top-level).

If `sink == 3`, send the message to the top-level. Else, the `2 ^ sink` bytes following the tag are an integer specifying which sink to send the message to.

The `2 ^ len` bytes following the sink id bytes (which may be zero bytes) are an unsigned integer indicating the length of the payload.

After the `len` bytes, place that many bytes of payload. This consumes as much credit for the sink.

### `Msg:Peer(sink: uint2_t, len: uint2_t)`, type nibble: 7 (`0b0111`)

Send a message down a sink that was opened by the peer.

If `sink != 3`, this uses the same encoding as the regular `Msg` chunk.

If `id == 3`, the chunk is a heartbeat pong for a request. The `len ^ 2` bytes following the tag specify the id of the request for which to send the pong. `len` may not be 3.

### `Req(id: uint2_t, len: uint2_t)`, type nibble: 8 (`0b1000`)

Send a request to the peer.

If `id != 3`, the `2 ^ id` bytes following the tag are an integer that serves as the id of the request.

The `2 ^ len` bytes following the request id bytes are an unsigned integer indicating the length of the payload.

After the `len` bytes, place that many bytes of payload. This consumes as much top-level credit.

### `Res(req: uint2_t, len: uint2_t)`, type nibble: 9 (`0b1001`)

Send a response to a request.

If `id != 3`, the `2 ^ req` bytes following the tag are an integer specifying the id of the request you are responding to.

The `2 ^ len` bytes following the request id bytes are an unsigned integer indicating the length of the payload.

After the `len` bytes, place that many bytes of payload. This consumes as much top-level credit.

### `Credit(stream: uint2_t, amount: uint2_t)`, type nibble: 14 (`0b1110`)

Give a certain amount of credit to a stream that was opened by this endpoint (or the top-level).

If `stream == 3`, give the credit to the top-level. Else, the `2 ^ stream` bytes following the tag are an integer specifying which stream to give the credit.

The `2 ^ amount` bytes following the stream id bytes (which may be zero bytes) are an unsigned integer indicating how much credit to give.

### `Credit:Peer(stream: uint2_t, amount: uint2_t)`, type nibble: 15 (`0b1111`)

Give a certain amount of credit to a stream that was opened by the peer.

If `stream != 3`, this uses the same encoding as the regular `Credit` chunk.

If `stream == 3`, the chunk activates partial mode for the next chunk following it. The `amount ^ 2` bytes following the tag specify the *total length* of the following payload.

If the next chunk is a `Msg` or `Msg:Peer` chunk, the sink on which the message is sent is put into partial mode. All message payloads on that sink are considered part of a single message's payload, until the *total length* has been reached. At that point, the sink switches back to normal operation.

If the next chunk is any other payload-carrying chunk, the entity specified by the combination of chunk type and id is put into partial mode, and all further chunks of matching type and id contribute to the payload, until the *total length* has been reached.

Following the partial mode activation with a non-payload-carrying chunk is invalid. Writing partial payloads that exceed the *total length* is invalid.

### `Cancel:Req(req: uint2_t, len: uint2_t)`, type nibble: 0 (`0b0000`)

Cancel a request that originated from this endpoint. Note that the endpoint issuing this chunk can not consider the request cancelled (for resource clean-up and id allocation) until it has received a confirmation from the peer. Otherwise, there would be race conditions with responses sent before the cancellation arrived. This note applies to all `Cancel` chunks.

If `req != 3`, the `2 ^ req` bytes following the tag are an integer specifying the id of the request you are cancelling.

The `2 ^ len` bytes following the request id bytes are an unsigned integer indicating the length of the payload.

After the `len` bytes, place that many bytes of payload. This consumes as much top-level credit.

If `req == 3`, the chunk acknowledges the cancellation of a response previously awaited by the peer. The `2 ^ len` bytes following the tag specify the id of the request on which to acknowledge the cancellation. `len` may not be 3.

### `Close:Res(req: uint2_t, len: uint2_t)`, type nibble: 1 (`0b0001`)

Close the response to a request that originated from the peer. This signifies that no response will be sent to the request. Note that the endpoint issuing this chunk can not consider the response closed (for resource clean-up and id allocation) until it has received a confirmation from the peer. Otherwise, there would be race conditions with cancellations sent before the closing arrived. This note applies to all `Close` chunks.

If `req != 3`, the `2 ^ req` bytes following the tag are an integer specifying the id of the request whose response you are closing.

The `2 ^ len` bytes following the request id bytes are an unsigned integer indicating the length of the payload.

After the `len` bytes, place that many bytes of payload. This consumes as much top-level credit.

If `req == 3`, the chunk acknowledges the closing of a request sent by this endpoint. The `2 ^ len` bytes following the tag specify the id of the request on which to acknowledge the closing. `len` may not be 3.

### `Cancel:Stream(stream: uint2_t, len: uint2_t)`, type nibble: 2 (`0b0010`)

Cancel a stream that was opened by this endpoint (or the top-level).

If `stream == 3`, this cancels the top-level. Else, the `2 ^ stream` bytes following the tag specify the id of the stream to cancel.

The `2 ^ len` bytes following the stream id bytes are an unsigned integer indicating the length of the payload.

After the `len` bytes, place that many bytes of payload. This consumes as much top-level credit.

### `Cancel:Stream:Peer(stream: uint2_t, len: uint2_t)`, type nibble: 3 (`0b0011`)

Cancel a stream that was opened by the peer.

If `stream != 3`, this uses the same encoding as the regular `Cancel:Stream` chunk.

If `stream == 3`, the chunk acknowledges the closing of a sink opened by this endpoint. The `2 ^ len` bytes following the tag specify the id of the sink on which to acknowledge the closing. `len` may not be 3. Note that it is neither necessary nor possible to acknowledge the closing of the top-level.

### `Close:Sink(sink: uint2_t, len: uint2_t)`, type nibble: 4 (`0b0100`)

Close a sink that was opened by this endpoint (or the top-level).

If `sink == 3`, this cancels the top-level. Else, the `2 ^ sink` bytes following the tag specify the id of the stream to cancel.

The `2 ^ len` bytes following the sink id bytes are an unsigned integer indicating the length of the payload.

After the `len` bytes, place that many bytes of payload. This consumes as much top-level credit.

### `Close:Sink:Peer(sink: uint2_t, len: uint2_t)`, type nibble: 5 (`0b0101`)

Close a sink that was opened by the peer.

If `sink != 3`, this uses the same encoding as the regular `Close:Sink` chunk.

If `sink == 3`, the chunk acknowledges a cancellation on a stream opened by this endpoint. The `2 ^ len` bytes following the tag specify the id of the stream on which to acknowledge the cancellation. `len` may not be 3. Note that it is neither necessary nor possible to acknowledge cancellation of the top-level.

### `Hearbeat(pong: bool, peer: bool, stream: uint2_t)`, type nibble: 13 (`0b1101`)

Handles heartbeats on streams. If `pong` is `0`, this is a ping, else a pong. If `peer` is `0`, `stream` refers to a stream opened by this endpoint, else `stream` refers to a stream opened by the peer.

If `sink == 3`, this refers to the top-level. Else, the `2 ^ sink` bytes following the tag specify the id of the stream to which the hearbeat applies.

If `peer` is `1`, `sink` may not be `3`.
