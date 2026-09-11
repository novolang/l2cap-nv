# l2cap-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The Bluetooth L2CAP layer with no radio and no controller under it
(Core Vol 3 Part A): basic-mode frames to and from bytes, the three LE
fixed channels as a type rather than as three magic numbers, the
reassembly of a frame that a controller split across several ACL
packets, the fragmentation that splits one, and the signalling commands
an LE peripheral actually meets.

It sits between hci-codec-nv and att-nv / smp-nv, and it speaks their
vocabulary rather than a second spelling of it: a frame arrives inside
hci-codec-nv's `AclData` and leaves the same way, and `CID_SMP` is the
`CID` smp-nv runs on.  A host stack drives it over a UART; a controller
drives it over an in-RAM ring; a capture tool drives it over a file.

## Adding it, and checking it

```bash
novo pkg add l2cap-nv         # into your novo.toml
novo pkg build                # type- and effect-check the package
novo test tests/l2cap_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion below the first constants fails with `not implemented:
l2cap.<fn>`.  They turn green one at a time as bodies land.

## The one example that will work

```novo
use l2cap

// The controller handed up one ACL packet.  Feed it, take what came
// out, and keep taking — one packet can finish two frames.
fn on_acl(r: L2capReassembler, acl: AclData) -> L2capReassembler
    var cur = r
    var step = l2cap.feed_acl(cur, acl)
    var go = true
    while go
        cur = step.reassembler
        match step.error
            None    => go = true
            Some(e) => on_link_fault(e)
        match step.frame
            None    => go = false
            Some(f) =>
                dispatch(f)
                step = l2cap.take(cur)
    cur

fn dispatch(f: L2capFrame)
    match l2cap.channel_of(f.cid)
        AttributeChannel       => on_att(f.payload)
        SecurityManagerChannel => on_smp(f.payload)
        SignallingChannel      => on_signal(f.payload)
        DynamicChannel(cid)    => on_unhandled(cid)
        ForeignChannel(cid)    => on_unhandled(cid)
```

## The layer, and why

`core`.  Everything here is arithmetic over bytes the caller already
holds.  The reassembly state is `L2capReassembler`, a value the caller
owns one of per connection — this package writes to no address, holds
no buffer and consults no clock, so the same code compiles for a
Cortex-M and for a laptop.

`tests/embedded_probe.nv` is that claim in a form that either builds or
does not, and **it builds**: `novo build --target=nrf52-qemu` produces
a Cortex-M4 ELF from the probe, this package and hci-codec-nv.

The shard audit's `core-embedded` row **cannot run that build today**,
and the reason is filed rather than designed around: the row assembles
its scratch package with an empty `[dependencies]`, so a probe whose
`use` lines reach a dependency — this one's reach hci-codec-nv — never
resolves.  The row is expected to fail on this package until that is
fixed, and the hand-linked build above is what stands in for it.

## The load-bearing interface

Two decisions, and both of them are about what a caller is told when
something is not a frame yet.

```novo
pub enum L2capScan
    L2capPartial
    L2capWhole(length: Int)

pub struct L2capStep
    reassembler: L2capReassembler
    frame: ?L2capFrame
    error: ?L2capError
```

**Waiting is not an error, and being wrong is not waiting.**  The
reference stack answers both questions with the same integer: a
`b_frame_length` of `-1` means "too short to read a header", and
`b_frame_complete` returning false means either "more is coming" or
"the length field lies", with no way to tell them apart.  On a link
those are opposite instructions — one says keep the bytes, the other
says drop them and log a fault — and `L2capScan` against `L2capError`
is that difference made a type.

`L2capStep` carries **both** an optional frame and an optional error
because feeding one ACL packet can do both at once: a start packet that
arrives while the previous frame is still owed drops the half-frame
(`L2capInterruptedFrame`) and begins a new one, and a step that could
only report one of those would lose whichever the caller cared about.

The third decision is the MTU:

```novo
pub fn reassembler_with_mtu(mtu: Int) -> L2capReassembler
pub fn set_mtu(r: L2capReassembler, mtu: Int) -> L2capReassembler
```

The MTU is the **caller's**, set from whatever an ATT exchange or a
signalling negotiation settled, and a frame whose length field goes past
it is refused at the first packet rather than buffered to the end.  A
runaway length field should raise its alarm at byte four, not at
kilobyte sixty-five, and a package with no MTU of its own has no way to
do that.

## Taking the boundary separately

`feed_acl` takes an `AclData`; `feed_payload` takes the boundary and the
payload as two arguments.  They are both here because the two callers
have the twelve header bits in different places: a host stack has an
`AclData` because that is what its HCI codec handed up, and a controller
has the bits in registers and the payload in a DMA buffer, where
building an `AclData` to pass them would be an allocation on the receive
path.

## What is not here, and what a consumer should expect

**LE credit-based flow control** — the signalling commands 0x14 to 0x18
and the dynamic channels 0x0040 and up.  `channel_of` names a dynamic
CID and `decode_signal` names the code it refused, so a consumer that
meets one gets a diagnosis rather than silence; what it does not get is
a connection-oriented channel.  The reference stack allocates none, and
a procedure written from the specification with no implementation to
measure against would be the one part of this package nobody had run.
It is the first thing a 0.1.0 should add for a consumer who needs LE
Audio or the Object Transfer Service.

**Enhanced retransmission and streaming modes.**  BR/EDR's, and this is
an LE package.

## The reference implementation

`orbit/ble/src/host/l2cap.nv` — 218 lines, six functions, and the header
comment that says what a split has to add: *"Only the single-PDU happy
path is implemented: a frame that spans more than one PDU is dropped
rather than reassembled."*  Its callers are
`host/conn_event_dispatch.nv` and `host/smp_loop.nv`, both of which call
`build_b_frame` directly and neither of which can send a payload longer
than one connection event will carry.

Four things change in the port.  `b_frame_length`'s and `b_frame_cid`'s
`-1` become `L2capScan` and `L2capError`.  `classify_cid`'s `1`/`2`/`3`
/`0` return becomes `L2capChannel`, so a CID cannot be compared against
an ATT opcode by accident.  Reassembly and fragmentation are new — the
reference has neither, and they are the reason an L2CAP layer exists at
all.  And the signalling channel is new: the reference recognises
CID 0x0005 in `classify_cid` and has nothing behind it, so a peripheral
built on it cannot ask its central for a longer connection interval.

The Bluetooth Core Specification Vol 3 Part A is the source of the test
vectors.

## Status

| item | implemented |
| --- | --- |
| the `CID_*`, `SIG_*` and `CONN_PARAM_*` constants | yes — they are constants |
| `l2cap.channel_of`, `.cid_of` | no |
| `l2cap.encode_frame`, `.decode_frame`, `.scan` | no |
| `l2cap.fragment` | no |
| `l2cap.reassembler`, `.reassembler_with_mtu`, `.mtu`, `.set_mtu`, `.reset` | no |
| `l2cap.feed_acl`, `.feed_payload`, `.take`, `.pending_len` | no |
| `l2cap.encode_signal`, `.decode_signal`, `.signal_code`, `.signal_identifier`, `.signal_frame` | no |
| `l2cap.L2capError.message` | no |
