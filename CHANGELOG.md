# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it. Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`. Adding this
package works and calling it panics.

- `L2capFrame` and `L2capChannel` — a basic-mode frame and the three LE
  fixed channels as types rather than as four header bytes and three
  magic numbers.
- `L2capScan` for "how much of this buffer is a frame" and `L2capError`
  for "this frame arrived whole and is wrong". Two types on purpose: on
  a link those carry opposite instructions, and the reference stack
  answers both with `-1`.
- `L2capReassembler`, `feed_acl`, `feed_payload`, `take` and
  `pending_len` — the state between an ACL packet and a frame, as a
  value the caller owns one of per connection. An ACL packet may split a
  frame anywhere, including between the two bytes of the length field,
  and that is the case the shape exists for.
- `fragment` — the other direction, over hci-codec-nv's `AclData` and
  `Boundary` rather than a second spelling of them.
- The MTU as a number the caller sets, so a runaway length field is
  refused at the first packet instead of buffered to the end.
- `L2capSignal` — Command Reject, the Connection Parameter Update
  request and response, and the disconnection pair. LE credit-based
  flow control is deliberately absent; the README says why.

### Design notes

Moved here from the README, which now states only what a user needs.

- The device claim is checked by hand rather than by the audit's
  `core-embedded` row. That row assembles its scratch package with an
  empty `[dependencies]`, so a probe whose `use` lines reach a
  dependency — this package's reach hci-codec-nv — never resolves. The
  row is expected to fail on this package until that is fixed, and
  `novo build --target=nrf52-qemu` over the probe stands in for it.
- `L2capScan` and `L2capError` are separate types because the reference
  stack answers both questions with `-1`: a `b_frame_length` of `-1`
  means "too short to read a header", and `b_frame_complete` returning
  false means either "more is coming" or "the length field lies".
- `L2capStep` is a struct rather than a tuple because its two halves are
  used at different places: the caller reassigns the reassembler and
  dispatches on the frame.
- The port from `orbit/ble/src/host/l2cap.nv` changes four things.
  `b_frame_length` and `b_frame_cid` lose their `-1` returns.
  `classify_cid`'s integer return becomes `L2capChannel`. Reassembly and
  fragmentation are new; the reference has neither. The signalling
  channel is new; the reference recognises CID 0x0005 and has nothing
  behind it.
