# deno_webrtc

WebRTC for Deno, as globals (W3C WebRTC 1.0): `RTCPeerConnection`,
`RTCDataChannel`, `RTCRtpTransceiver` / `RTCRtpSender` / `RTCRtpReceiver`,
`MediaStream`, `MediaStreamTrack`, `RTCCertificate`, `RTCDtlsTransport`,
`RTCIceTransport`, `RTCSessionDescription`, `RTCIceCandidate`, their events and
`RTCStatsReport`; and `Deno.webrtc`, the server-side extension (sending encoded
media and RTP, reading received RTP, keeping a DTLS certificate). There is no
capture device or codec: a local track is a `Deno.webrtc.EncodedMediaTrack` the
program feeds.

The protocol stack (ICE, DTLS, SCTP, the data channel protocol, SRTP, RTP / RTCP
and its interceptors) is the sans-I/O `rtci` crates from crates.io
([Brainwires/rtci](https://github.com/Brainwires/rtci), a fork of
[webrtc-rs/rtc](https://github.com/webrtc-rs/rtc), synced with upstream);
`peer.rs` owns the UDP sockets, the ICE-TCP connections (RFC 6544, on a
`Deno.webrtc.createNetwork({ tcpPort })` network), the mDNS socket that resolves
(and, with `mdns: true`, publishes) `.local` host candidates, the timers and the
STUN/TURN clients around it, and gates every datagram through the net
permission; `net.rs` holds the networks' shared sockets and TCP listeners;
`media.rs` maps transceivers, senders and receivers onto the engine's.
`Deno.webrtc.setLogLevel()` shows the engine's own log on stderr (off by
default).

Status: this crate is the native half only. The JavaScript surface (the globals
above and `Deno.webrtc`) and its registration in the runtime come in follow-up
changes; until then the extension is not part of any worker, and
`cargo test -p deno_webrtc` exercises the engine glue directly (two peers on
loopback, without V8).

Spec: https://www.w3.org/TR/webrtc/
