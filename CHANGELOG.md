# Changelog

## 0.1.0 (2026-10-20)

First public version.

- Policy query with `getStatusForPolicy()` across Widevine L3/L1 and PlayReady SL2000/SL3000, with a control query per profile so broken CDMs show up as "unreliable" instead of "no HDCP".
- Real license test against Microsoft's PlayReady test server: control, HDCP required, HDCP 2.2 required. Hardware and software PlayReady.
- Detects the hardware PlayReady "freeze instead of error" behavior and counts it as a fail when the control case played.
- Unwraps shaka-player errors down to the real HTTP/network error.
- Plain-text report and JSON export for sharing results.
