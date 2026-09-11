# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

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
