# l2cap-nv

The Logical Link Control and Adaptation Protocol (L2CAP) is the Bluetooth
layer that turns a connection's stream of packets into channels. It is
specified in the [Bluetooth Core Specification](https://www.bluetooth.com/specifications/specs/core-specification/),
Volume 3, Part A. This package brings the Bluetooth Low Energy (LE) part
of it to novo-lang, with no radio and no controller underneath. Two other
packages on the registry run on channels this one defines:
[att-nv](https://novo-lang.org/packages/att-nv), the Attribute Protocol,
and [smp-nv](https://novo-lang.org/packages/smp-nv), the Security Manager.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What L2CAP is

A Bluetooth connection carries one undifferentiated stream of packets.
L2CAP puts a four-byte header in front of each payload, and that header
says which conversation the payload belongs to. The unit it defines is
the **basic-mode frame**, or B-frame: two bytes of payload length, two
bytes of **channel identifier** (CID), then the payload. Core Vol 3
Part A section 3.1 defines it. Every multi-byte field is little-endian,
which is the Bluetooth convention throughout.

On LE the specification fixes three CIDs, listed in section 2.1,
Table 2.1. Each names one protocol.

| CID | Channel | What runs on it |
| --- | --- | --- |
| 0x0004 | Attribute | The Attribute Protocol, which reads and writes a peer's attributes |
| 0x0005 | LE signalling | L2CAP's own commands, such as a request for a different connection interval |
| 0x0006 | Security Manager | Pairing and key distribution |

A frame does not always fit in one packet. The controller carries L2CAP
frames inside **ACL data packets**, and an ACL packet has its own size
limit, reported by the controller's `LE_Read_Buffer_Size` command. A
frame larger than that limit is sent as several ACL packets: the first
carries a start boundary, the rest carry a continuation boundary.
Section 7.2 defines the rule. Splitting one frame into ACL packets is
**fragmentation**; putting the pieces back together at the far end is
**reassembly**.

An ACL packet may split a frame at any byte, including between the two
bytes of the length field. A reassembler therefore cannot assume it has
a whole header to read when it is handed a packet.

The **maximum transmission unit** (MTU) is the largest payload the two
peers have agreed to exchange on a channel. Section 5.1 sets the LE
minimum at 23 bytes. The MTU is not carried on the wire with each frame.
It is settled once, by an Attribute Protocol exchange or by a signalling
negotiation, and both sides remember it.

This package performs no input or output. It holds no buffer of its own,
writes to no address and consults no clock. Every function is arithmetic
over bytes the caller already holds, so the same code runs in a
controller's receive path, in a host stack on a laptop, and in a capture
tool reading a file.

## Install

```
novo pkg add l2cap-nv
```

## Example

```novo
use l2cap

fn main() [io]
    // One basic-mode frame as it arrives: three payload bytes on the
    // attribute channel, whose channel identifier is 0x0004.
    let buf: [u8] = [0x03 as u8, 0x00 as u8, 0x04 as u8, 0x00 as u8,
                     0x0A as u8, 0x01 as u8, 0x00 as u8]

    // Ask how much of the buffer is one whole frame, without copying it.
    match l2cap.scan(buf)
        L2capPartial  => println("the frame has not all arrived")
        L2capWhole(n) => println("${n} bytes are one frame")

    // Read the frame and dispatch on the channel its identifier names.
    match l2cap.decode_frame(buf)
        Ok(frame) =>
            match l2cap.channel_of(frame.cid)
                AttributeChannel       => println("for the attribute protocol")
                SecurityManagerChannel => println("for pairing")
                SignallingChannel      => println("for signalling")
                DynamicChannel(cid)    => println("credit-based channel ${cid}")
                ForeignChannel(cid)    => println("not an LE channel: ${cid}")
        Err(e) => println("the frame is wrong: ${e.message()}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented: l2cap.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `l2cap` | The whole surface. The fixed channel identifiers, the basic-mode frame as a type, encoding and decoding, fragmentation into ACL packets, a reassembler that puts them back together, and the LE signalling commands. |

## How to choose an entry point

There are three ways to read an incoming frame, and they differ in how
much of the arithmetic you do yourself.

**`decode_frame` reads one whole frame from one buffer.** Use it when
something upstream has already given you exactly one frame's bytes. It
refuses a buffer that carries a tail beyond the frame.

**`scan` reads only the length field.** It answers how many bytes at the
front of the buffer are one frame, and it copies nothing. Use it when
you hold a receive buffer and slice frames out of it yourself.

**`L2capReassembler` handles the case where the controller split the
frame.** Feed it each ACL packet with `feed_acl`, then call `take` until
it stops producing frames. Use it in any stack that talks to a real
controller, because a controller splits frames whenever one exceeds its
ACL packet size. Keep one reassembler per connection.

There are also two ways to feed the reassembler. `feed_acl` takes an
`AclData` value, which is what a host stack's HCI codec hands up.
`feed_payload` takes the boundary and the payload as two separate
arguments, which is the shape a controller has: the header bits are in
registers and the payload is in a direct memory access buffer.

## The rules a user needs

1. **A frame that has not all arrived is not an error.** `scan` answers
   `L2capPartial` for a prefix, and the caller keeps the bytes.
   `L2capError` means something arrived complete and was wrong, and the
   caller drops the bytes and logs a fault. The two answers are opposite
   instructions, so they are separate types.
2. **One ACL packet can produce a frame and an error at once.**
   `feed_acl` returns an `L2capStep` carrying an optional frame and an
   optional error. A start packet arriving while the previous frame is
   still owed drops the half-frame with `L2capInterruptedFrame` and
   begins a new one. Check both fields.
3. **`feed_acl` drains one frame per call.** A controller with a large
   ACL buffer can complete two frames in one packet. Call `take` in a
   loop until it returns no frame.
4. **The MTU is yours to set.** This package neither performs the
   Attribute Protocol exchange nor the signalling negotiation that
   settles it. `reassembler()` starts at the LE minimum of 23 bytes
   (section 5.1). Call `set_mtu` the moment a negotiation settles, and
   a frame whose length field exceeds the MTU is refused with
   `L2capOverMtu` at the first packet rather than buffered to the end.
5. **`fragment` takes the controller's ACL packet size, not the L2CAP
   MTU.** These are different numbers. The ACL packet size is what
   `LE_Read_Buffer_Size` returns (section 7.2). A frame that fits in one
   packet comes back as one packet.
6. **Call `reset` on a disconnection.** A half-frame belongs to the
   connection it arrived on. Carrying it into the next connection would
   deliver one peer's bytes to another.
7. **Signalling intervals are in the specification's own units.** A
   connection interval is counted in 1.25 ms steps and a supervision
   timeout in 10 ms steps (section 4.20). This package does not convert
   them, because the controller wants the specification's number.
8. **A signalling response is paired with its request by the identifier
   byte.** Every `L2capSignal` carries one. This package has no timer
   and no table of outstanding requests, so the pairing is the caller's
   to keep (section 4).
9. **`encode_signal` produces a payload, not a frame.** Wrap it with
   `signal_frame` to get something `encode_frame` accepts.

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers the whole package. Every function is
integer arithmetic over bytes the caller supplies, and the reassembly
state is a value the caller owns rather than a buffer this package
hides.

```bash
novo build --target=nrf52-qemu src/main.nv
```

`tests/embedded_probe.nv` is that claim as a program, and it does not
build today. The probe names `FirstFlushable`, a variant of
hci-codec-nv's `Boundary`, and a variant constructor from a dependency
is not in scope without a `use` of that package. A dependency's type
names do cross without one, which is why `AclData` resolves in the same
file. Nothing yet checks the device build automatically, so the claim
rests on the code.

On a device the payload arrives in a direct memory access buffer and the
ACL header bits arrive in registers. `feed_payload` is the entry point
for that path, because building an `AclData` value to pass them would
allocate on the receive path.

## What is not included

- **LE credit-based flow control**, the signalling commands 0x14 to 0x18
  and the dynamic channels from 0x0040 up. A consumer that meets one
  gets a diagnosis rather than silence: `channel_of` names a dynamic CID
  and `decode_signal` reports the code it refused. What it does not get
  is a connection-oriented channel. This is the first thing version
  0.1.0 should add for a program that needs LE Audio or the Object
  Transfer Service.
- **Enhanced retransmission mode and streaming mode.** Both belong to
  Bluetooth Basic Rate and Enhanced Data Rate. This package is LE.
- **Timers and a table of outstanding requests.** The package has no
  clock, so nothing here times out a signalling request or resends one.
  A caller that sends a request keeps its identifier and matches the
  response itself.
- **A radio, a controller and a transport.** Frames arrive as bytes and
  leave as bytes. Getting them to and from the controller is
  [hci-codec-nv](https://novo-lang.org/packages/hci-codec-nv)'s work.

## Related packages

- [hci-codec-nv](https://novo-lang.org/packages/hci-codec-nv) is the
  layer below. It turns the controller's byte stream into commands,
  events and ACL data packets. This package depends on it, so that
  `fragment` and `feed_acl` speak its `AclData` and `Boundary` types
  rather than a second spelling of the same header bits.
- [att-nv](https://novo-lang.org/packages/att-nv) is the Attribute
  Protocol, which runs on CID 0x0004. It takes the payload of a frame
  this package decoded.
- [smp-nv](https://novo-lang.org/packages/smp-nv) is the Security
  Manager, which runs on CID 0x0006, in the same relation.
- [ble-link-codec-nv](https://novo-lang.org/packages/ble-link-codec-nv)
  is the link layer below HCI, for a program that speaks to a radio
  directly rather than to a controller.

## Tests

```bash
novo test tests/l2cap_tests.nv       # 23 tests against the signatures
```

The byte strings the suite asserts against are the Bluetooth Core
Specification's own, Volume 3, Part A: the section 3.1 basic-mode
header, the section 4.20 Connection Parameter Update Request and the
section 4.21 response. The suite checks that a CID and its channel round
trip, that a frame survives encoding and decoding, that a frame split
across ACL packets is reassembled byte for byte, and that each
`L2capError` is raised by the input that should raise it.

The tests compile today and fail at run, each on the `not implemented:
l2cap.<fn>` panic that is its body. That is the expected state of an
interface release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| The `CID_*`, `SIG_*` and `CONN_PARAM_*` constants | yes (they are constants) |
| `l2cap.channel_of`, `.cid_of` | no |
| `l2cap.encode_frame`, `.decode_frame`, `.scan` | no |
| `l2cap.fragment` | no |
| `l2cap.reassembler`, `.reassembler_with_mtu`, `.mtu`, `.set_mtu`, `.reset` | no |
| `l2cap.feed_acl`, `.feed_payload`, `.take`, `.pending_len` | no |
| `l2cap.encode_signal`, `.decode_signal`, `.signal_code`, `.signal_identifier`, `.signal_frame` | no |
| `l2cap.L2capError.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
