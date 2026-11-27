# Port notes: fork train denext31 (for PR 2)

Branch `feat/webrtc-sync-d31` brings the Rust (native crate) half of the
fork's `ext/webrtc` changes between Brainwires/deno `7f994b6ace` (PR 1's
source) and `train/denext31` `81c56d13c5` (DENEXT.md items 560-570). The
JavaScript half belongs to PR 2 (the W3C surface) and is listed here.

Easiest path for PR 2: take `01_webrtc.js` and `05_transfer.js` from the
train (`81c56d13c5` or later), not from `7f994b6ace`. The Rust crate on this
branch already has the interface the train's JavaScript expects, and the
older JavaScript does not match it (see "Interface changes").

## Interface changes the JavaScript must follow

| Rust change (this branch) | What the JavaScript does (train) | Item |
| --- | --- | --- |
| `op_webrtc_receiver_muted(rid, key, generation)` takes a third `u32` | `rearmUnmute()` moves `receiver[_state].unmuteGeneration` on and passes it; the older two-argument call no longer matches the op | 561 |
| `trackunmute` events carry `generation` | `dispatchEngineEvent` ignores a `trackunmute` whose `generation` is not the receiver's current one; `restoreRemoteTracks` and `syncTransceivers` (received again after a pause) re-arm | 561 |
| `dcerror` events carry `errorDetail`, `message`, `sctpCauseCode`, `label`, `id` (camelCase) | `dataChannelError(event, channel)` builds the `RTCError` from them; `05_transfer.js` forwards them to a transferred channel and builds the same error there | 569 |
| `set_header_extensions_to_negotiate` no longer turns a stopped mandatory extension back on; an engine refusal is `InvalidModificationError` | `RTCRtpTransceiver.setHeaderExtensionsToNegotiate()` throws `InvalidModificationError` when stopping `urn:ietf:params:rtp-hdrext:ssrc-audio-level` on audio (RFC 8834 section 5.2.2), before the op | 570 |

## JavaScript-only changes, by fork item

- 561 (`9ab8958636`): the unmute generation above (`setMuted` /
  `rearmUnmute`, `RTCRtpReceiver` state `unmuteGeneration: 0`).
- 567 (`9aa4b7c404`): RFC 6904 encrypted header extensions. `extmapsOf`
  reads `a=extmap:<id> urn:ietf:params:rtp-hdrext:encrypt <uri>` as `<uri>`
  with `encrypted: true` (`ENCRYPT_URI`); `headerExtensionsNegotiated` takes
  the clear form first; `requireStableExtensionIds` treats a URI in its
  clear and encrypted forms as two mappings. Unit tests:
  `webrtcRtpAndKeyFrameRequests` uses video, `webrtcEcnMarksVideoEct1` takes
  `a=rtcp-fb:* ack ccfb` (`tests/unit/webrtc_test.ts`).
- 568 (`63955346c1`): `getStats()` drops the engine's own `media-source`
  objects (the one made from the track, which `mediaSourceId` names, stays);
  `requireRtcpMux` looks for the `a=rtcp-mux` line itself, not
  `a=rtcp-mux-only`. Unit tests: `tests/unit/webrtc_datachannel_test.ts`,
  `webrtc_hdrext_test.ts`, `webrtc_media_test.ts`, `webrtc_test.ts`.
- 569 part 1 (`f54ee334e2`): `setLocalDescription({type: "rollback"})`
  outside `have-local-offer` and `setRemoteDescription({type: "rollback"})`
  outside `have-remote-offer` throw `InvalidStateError` (RFC 9429 section
  4.1.10.2).
- 570 (`3b23b33a28`): a data channel's stats object is matched to the
  engine's by SCTP stream id (`dataChannelIdentifier`) and keeps the id
  `RTCDataChannel_<handle>` (rtci 0.2 ids are opaque); the
  `AUDIO_LEVEL_EXTENSION` refusal above. Unit test:
  `tests/unit/webrtc_hdrext_test.ts` (`webrtcHeaderExtensionsNegotiated`
  stops the RFC 6465 level and checks that stopping RFC 6464's throws).
- 569 part 3 (`184ba43678`): `send()` of a Blob, and of any data queued
  behind one (`state.queue` not empty), calls `checkBufferSpace()`: an `OperationError` when `bufferedAmount` plus what
  is queued plus the new data would pass the transport's `[[MaxBufferSize]]`
  (the larger of the peer's max-message-size and 16 MiB,
  `SEND_BUFFER_BYTES`). `getStats()` fills a remote candidate's empty
  `usernameFragment` from the remote description's ufrag.
- 569 (`92e3184409`): `announcedSdp()` counts kept and hidden candidates; when
  every candidate is hidden, `withoutDefaultDestination()` puts the m= port
  back to 9, `c=IN IP4 0.0.0.0` and `a=rtcp:9 IN IP4 0.0.0.0` (a rejected or
  bundle-only section keeps port 0), so a description read right after
  `setLocalDescription()` is the one `createAnswer()` returned.
- 569 part 4 (`79ffd6074e`): `dataChannelError()` and the `05_transfer.js`
  error forwarding above.

## Outside `ext/webrtc` (other PRs)

- WPT expectations (PR 5): the train changed
  `tests/wpt/runner/expectations/webrtc.json` (`0aba41b93c`, `f54ee334e2`,
  `184ba43678`) and `webrtc-stats.json` (`184ba43678`); regenerate against
  Dodo's WPT revision rather than copying.
- Items 555 and 564 (worker, `MessagePort` and `node:worker_threads`
  messages each their own task) are not WebRTC code; 555 is PR W, and 564
  (`ext/web/13_message_port.js`, `ext/node/polyfills/worker_threads.ts`)
  belongs with it. `transfer-datachannel.html` passes only with both.
