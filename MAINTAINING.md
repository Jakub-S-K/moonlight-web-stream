# Fork maintenance (moonlight-web-stream + microphone)

This fork adds **browser microphone support** and a few playback fixes to
[moonlight-web-stream](https://github.com/mrcreativ3001/moonlight-web-stream).
The microphone is forwarded to a Wolf host using the Foundation Sunshine mic
protocol (`MoonlightStream::send_microphone_opus_data`).

## Branch layout
- `feat/microphone` – our commits on top of upstream `master` (`ed70db14`).
- `upstream` – points at `https://github.com/mrcreativ3001/moonlight-web-stream.git`.

Our commits (oldest first):
1. `feat(mic): send browser microphone to the host` – WebSocket + WebRTC paths,
   WebCodecs Opus encoding, `window.moonlightMic` helper.
2. `build: add multi-stage Dockerfile for source builds` – `Dockerfile.build`.
3. `fix(web): audio/video playback robustness` – AudioContext resume, audio
   sync pipeline, WebCodecs decoder recovery.
4. `fix(deps): patch Foundation microphone depayloader panic` – vendored
   `moonlight-common` with the depayloader `expect()` removed + `[patch]`.

## One-time setup (your GitHub fork)
```bash
git remote add upstream https://github.com/mrcreativ3001/moonlight-web-stream.git
git remote add origin   git@github.com:<YOUR_USER>/moonlight-web-stream.git
git push -u origin feat/microphone
```
Create the empty fork on GitHub first (without a README).

## Updating from upstream
```bash
git fetch upstream
git checkout feat/microphone
git rebase upstream/master
git push --force-with-lease origin feat/microphone
```

## The vendored crate (`vendor/moonlight-common-rust`)
Upstream pins `moonlight-common` to a git revision. The depayloader panic
(`failed to get frame after having successfully produced it`) kills the stream,
so this fork vendors the crate with the panic removed and wires it through:

```toml
[patch."https://github.com/MrCreativ3001/moonlight-common-rust.git"]
moonlight-common = { path = "vendor/moonlight-common-rust" }
```

When upstream bumps the `moonlight-common` revision, **re-vendor** it: clone the
new revision into `vendor/moonlight-common-rust`, re-apply the one-line fix in
`src/stream/proto/video/depayloader.rs` and commit. Alternatively fork
`moonlight-common-rust` and point the dependency at your fork instead of using
`vendor/`.

Note: the WASM frontend build (uniffi/ubrn) still fetches the upstream crate
directly; the panic fix only matters on the server side.

## Building
Build on an x86_64 machine with Docker (the phone-class ARM boxes are too slow):
```bash
docker build -f Dockerfile.build -t moonlight-web:mic .
```
The image contains `web-server` + the static frontend and runs as a non-root
user. The runtime base is `ubuntu:24.04` because the binary needs glibc 2.39.

## Running (reference setup)
```bash
docker run -d --name moonlight-web --network=host \
  -e WEBRTC_NAT_1TO1_HOST=<LAN_IP> \
  -e WEBRTC_PORT_RANGE=40000:40100 \
  -v moonlight-web-server:/moonlight-web/server \
  --restart unless-stopped moonlight-web:mic
```
- `--network=host` avoids WebRTC ICE/NAT problems; put an HTTPS reverse proxy in
  front (webcam/mic APIs require a secure context).
- The server binds `0.0.0.0:8080` by default; change
  `web_server.bind_address` in `server/config.json` if the port is taken.
- In the stream settings, **Data Transport = WebSocket** is the most reliable
  path for the microphone (WebRTC depends on ICE/mDNS).
- Chrome/Edge are required for microphone over WebSocket
  (`MediaStreamTrackProcessor` + `AudioEncoder`); Safari lacks them.
