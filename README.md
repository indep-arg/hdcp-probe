# hdcp-probe

Check whether HDCP actually works between your GPU and your display, straight from the browser. No install, no drivers, nothing changed on your system.

**Run it:** https://indep-arg.github.io/hdcp-probe/

## Why

Netflix, Max and others drop you to 720p or worse when HDCP isn't engaged, and they won't tell you why. The usual checks don't help much either:

- The NVIDIA Control Panel "HDCP status" page only tells you that the GPU and the display *claim* HDCP support (it reads the EDID). It doesn't tell you whether the handshake actually works.
- Browser checks based on `getStatusForPolicy()` ask a hypothetical question, and some CDMs answer it badly.

hdcp-probe runs both kinds of checks and, more importantly, cross-checks them against a **real PlayReady license** that requires HDCP. That's the same thing a streaming service does, so if this fails, they fail too.

## What it does

### 1. Policy query

For each DRM system the browser exposes (Widevine L3/L1, PlayReady SL2000/SL3000) it asks, via [EME `getStatusForPolicy()`](https://wicg.github.io/hdcp-detection/), whether HDCP 1.0 through 2.3 could be enforced right now.

Each profile is first queried with an empty policy (the **control**). If even that comes back restricted, the profile can't measure anything and is marked *unreliable* instead of reporting "no HDCP".

It also checks HEVC support for the hardware DRM path and whether 1080p H.264 / 2160p HEVC decode in hardware.

### 2. Real license test (PlayReady)

Plays a protected clip from [Microsoft's public PlayReady test server](https://learn.microsoft.com/playready/advanced/testservers/query-string-syntax) three times, each with a different license:

| case | license policy | meaning |
|---|---|---|
| control | none | proves DRM, network and test server work |
| requires HDCP (any) | `ucvopl:300` | digital output only allowed with HDCP engaged |
| requires HDCP 2.2 | `ucvopl:300` + HDCP Type 1 `dvop` | needs HDCP 2.2 or newer |

A case **passes** if the clip plays for 2 seconds with all keys `usable`. It **fails** if the CDM marks the keys `output-restricted`, or if hardware PlayReady freezes the video while the control case played fine on the same server (hardware PlayReady often just freezes instead of reporting anything).

## Requirements

- Windows 10/11
- **Microsoft Edge in a normal window.** InPrivate disables hardware PlayReady. Chrome and Firefox only have software Widevine on Windows, so the policy query works there but the license test doesn't.
- Hardware acceleration enabled in Edge (`edge://settings/system`)

## Reading the results

| what you see | what it means |
|---|---|
| License test: control passes, HDCP 2.2 passes | Your chain does HDCP 2.2. 4K streaming should work. If it doesn't, look at the service, your plan or HEVC. |
| Control passes, HDCP (any) passes, 2.2 fails | HDCP 1.x only. You'll get 1080p, not 4K. Common with DisplayPort 1.2, older HDMI ports, cheap cables or "HDMI 1.4 mode" in the display's OSD. |
| Control passes, HDCP (any) fails | No HDCP at all on this link. Streaming services will cap you at 540p/720p. |
| Control fails | The test couldn't run (network, CORS, test server down). Says nothing about your display. Check the error detail. |
| Policy query: everything `output-restricted`, but control `usable` | The CDM says no HDCP. Confirm with the license test before trusting it. |
| Policy query: no SL3000 in Edge | You're in InPrivate, hardware acceleration is off, or the driver blocks hardware DRM. |

### Narrowing down a failure

Run the tests after each change:

1. Unplug every other display, capture card, KVM or adapter.
2. Close anything that hooks the display output: screen recorders, OBS, Parsec, Sunshine, AnyDesk, virtual display drivers.
3. Try another cable, then another port (HDMI vs DisplayPort).
4. Check the display's OSD for an HDMI mode setting (HDMI 1.4 / 2.0, "Deep Color", "UHD color").
5. Plug the PC into a TV. **If it passes there, your display is the problem.** If it still fails, it's the PC side (driver, software).

## Running it locally

It's one static HTML file. Any local server works:

```sh
python -m http.server 8000
# open http://localhost:8000 in Edge
```

Opening the file directly (`file://`) may not count as a secure context, and EME needs one.

## Sharing results

Use **Copy report** and open an issue with the [result report template](../../issues/new?template=report.yml). Results from different GPUs, displays and cables help figure out where the policy query is wrong and which setups break HDCP.

## How it works (for the curious)

- Policy query: `navigator.requestMediaKeySystemAccess()` per key system and robustness level, then `MediaKeys.getStatusForPolicy({ minHdcpVersion })` for each version.
- License test: [shaka-player](https://github.com/shaka-project/shaka-player) loaded from jsDelivr, playing Microsoft's *Tears of Steel* test stream. The test manifest embeds an old license URL that no longer resolves, so a request filter forces every license request to the test server URL with the chosen policy.
- Several test server endpoints are tried in order. The first one where the control case plays is used for the HDCP cases.

## Limitations

- The license test only covers PlayReady, so it's Edge on Windows only for now.
- It depends on Microsoft's public test server being up.
- It tells you whether HDCP works on the path the browser uses. It doesn't read the HDCP version negotiated on the cable directly.

## Not a bypass

This tool only *diagnoses*. It doesn't remove, spoof or work around HDCP or any DRM, and it won't. If your setup fails, the fix is hardware or configuration: cable, port, display mode, or a display that actually supports HDCP.

## Roadmap

- Native companion (Rust) that reads the negotiated HDCP state through Windows OPM / NVAPI, without the browser.
- Widevine license test for Chrome/Linux.
- Detect common HDCP breakers (virtual displays, capture apps) automatically.

## License

[MIT](LICENSE)
