# satlib — Satellite Tracking Library Protocol

**Version:** 1  
**Default port:** 4534  
**Status:** Draft

---

## 1. Overview

**satlib** is a lightweight, read-only protocol for exposing real-time satellite tracking data over a local network. A compliant server serves a snapshot of the currently tracked satellite — position, Doppler-corrected frequencies, CTCSS tones, and pass timing — as a JSON object in response to any HTTP GET request.

The protocol is intentionally minimal: one endpoint, one method, one response format. Any HTTP client can consume it without a custom parser.

---

## 2. Relationship to rigctld and rotctld

satlib belongs to the same toolchain family as Hamlib's [rigctld](https://hamlib.sourceforge.net/) (rig frequency control, port 4532) and [rotctld](https://hamlib.sourceforge.net/) (rotator control, port 4533):

| Port | Daemon  | Function                       |
|------|---------|--------------------------------|
| 4532 | rigctld | Rig/frequency control (Hamlib) |
| 4533 | rotctld | Rotator control (Hamlib)       |
| 4534 | satlib  | Satellite tracking data (read) |

**Key difference from rigctld/rotctld:** Those daemons use a bare TCP text protocol with stateful sessions and bidirectional command/response pairs. satlib uses **HTTP/1.1 over TCP** — same wire transport, but with HTTP framing on top. This makes satlib universally accessible to any HTTP client (curl, browser, fetch, Python requests) without implementing a custom line protocol.

satlib is **read-only**: it exposes data produced by the satellite tracker. Frequency commands and rotator commands remain the responsibility of rigctld and rotctld respectively.

---

## 3. Transport

- **Protocol:** TCP
- **Default port:** 4534 (configurable)
- **Connections:** short-lived; server closes after each response (`Connection: close`)
- **Binding:** `0.0.0.0` (all interfaces) — accessible to any device on the local network

---

## 4. HTTP Layer

satlib implements a minimal subset of HTTP/1.1. Clients send a standard GET request; the server responds with 200 OK and a JSON body regardless of the request path or headers.

### Request

```
GET / HTTP/1.1\r\n
Host: <server-ip>:4534\r\n
\r\n
```

Any path is accepted. The server only reads the first request line and ignores headers.

### Response headers

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: <bytes>
Access-Control-Allow-Origin: *
Connection: close
```

`Access-Control-Allow-Origin: *` is always present to allow consumption from browser-based tools without CORS configuration.

---

## 5. Response Body

The response body is a UTF-8 encoded JSON object. Field order is fixed as shown below. All numeric values use their natural JSON representations (no string wrapping). Nullable fields are `null` when no value is available.

### Example — active pass, FM satellite

```json
{
  "version": 1,
  "satName": "ISS (ZARYA)",
  "catNum": 25544,
  "azimuthDeg": 247.35,
  "elevationDeg": 12.84,
  "altitudeKm": 421.10,
  "distanceKm": 1893.44,
  "subSatLatDeg": 48.7612,
  "subSatLonDeg": -3.4201,
  "aboveHorizon": true,
  "txFrequencyHz": 145827340,
  "rxFrequencyHz": 145826100,
  "ctcssTxToneHz": 67.0,
  "ctcsRxToneHz": null,
  "mode": "FM",
  "aosTime": 1752012345000,
  "losTime": 1752012945000,
  "timestamp": 1752012567000
}
```

### Example — idle (no satellite being tracked)

```json
{
  "version": 1,
  "satName": "",
  "catNum": 0,
  "azimuthDeg": 0.0,
  "elevationDeg": 0.0,
  "altitudeKm": 0.0,
  "distanceKm": 0.0,
  "subSatLatDeg": 0.0,
  "subSatLonDeg": 0.0,
  "aboveHorizon": false,
  "txFrequencyHz": null,
  "rxFrequencyHz": null,
  "ctcssTxToneHz": null,
  "ctcsRxToneHz": null,
  "mode": null,
  "aosTime": 0,
  "losTime": 0,
  "timestamp": 0
}
```

Clients MUST detect the idle state by checking `satName == ""` **or** `timestamp == 0`. All numeric fields carry zero-value defaults when idle and MUST NOT be treated as valid tracking data.

---

## 6. Field Reference

| Field | Type | Unit | Nullable | Description |
|---|---|---|---|---|
| `version` | integer | — | no | Protocol version. Currently `1`. Incremented on breaking schema changes. |
| `satName` | string | — | no | Satellite name. Empty string when idle. |
| `catNum` | integer | — | no | NORAD catalog number. `0` when idle. |
| `azimuthDeg` | float | degrees | no | Azimuth of satellite from observer. `0°` = North, `90°` = East. Range: `[0, 360)`. |
| `elevationDeg` | float | degrees | no | Elevation above horizon. Negative values indicate satellite is below horizon. |
| `altitudeKm` | float | km | no | Satellite altitude above Earth's surface. |
| `distanceKm` | float | km | no | Slant range from observer to satellite. |
| `subSatLatDeg` | float | degrees | no | Sub-satellite point latitude. Range: `[-90, 90]`. |
| `subSatLonDeg` | float | degrees | no | Sub-satellite point longitude. Range: `(-180, 180]`. |
| `aboveHorizon` | boolean | — | no | `true` when `elevationDeg > 0`. |
| `txFrequencyHz` | integer | Hz | yes | Doppler-corrected uplink (TX) frequency. `null` if no transponder selected. |
| `rxFrequencyHz` | integer | Hz | yes | Doppler-corrected downlink (RX) center frequency. `null` if no transponder selected. |
| `ctcssTxToneHz` | float | Hz | yes | CTCSS tone the ground station transmits on uplink. `null` if none set. Standard CTCSS values (e.g. `67.0`, `88.5`). |
| `ctcsRxToneHz` | float | Hz | yes | CTCSS tone expected on the satellite downlink. `null` if none or unknown. |
| `mode` | string | — | yes | Emission mode of the selected downlink (e.g. `"FM"`, `"USB"`, `"CW"`). `null` if no transponder selected. |
| `aosTime` | integer | ms (Unix) | no | Acquisition of Signal time for current pass. `0` when idle. |
| `losTime` | integer | ms (Unix) | no | Loss of Signal time for current pass. `0` when idle. |
| `timestamp` | integer | ms (Unix) | no | Server time when this snapshot was computed. `0` when idle. |

### Frequency notes

- `txFrequencyHz` and `rxFrequencyHz` include Doppler correction applied at the moment of the snapshot. They change every update tick as the satellite moves.
- For simplex FM satellites, `txFrequencyHz == rxFrequencyHz`.
- For linear (SSB/CW) transponders, `rxFrequencyHz` is the center of the downlink passband, and the TX/RX offset follows the transponder inversion if applicable.
- Frequencies are always positive integers in Hz (e.g. `145825000` = 145.825 MHz).

---

## 7. Update Rate

The server updates its internal snapshot once per second as the satellite tracker computes a new position fix. Clients polling faster than 1 Hz will receive repeated identical snapshots. A polling interval of **1000 ms** matches the native update rate; **500 ms** is acceptable for responsive UIs.

---

## 8. Idle State Detection

A client MUST consider the server idle when either of the following is true:

- `satName` is an empty string (`""`)
- `timestamp` equals `0`

When idle, all numeric fields hold zero-value defaults and MUST be ignored. Clients should display a "no active pass" indicator rather than treating zero-valued fields as real data.

---

## 9. Versioning

The `version` field carries an integer that increments on every breaking schema change (field removed, field type changed, field semantics changed). Adding new optional fields is non-breaking and does not increment `version`.

Clients that do not recognise the `version` value SHOULD warn the user and MAY refuse to operate.

Current version: **1**

---

## 10. Error Handling

satlib does not return HTTP error codes. If the server is running, it always responds `200 OK`. If the server is not reachable (process not running, port not open, network unreachable), the TCP connection will be refused — clients should handle this as a connection error and retry with backoff.

---

## 11. Client Quick-Start

### curl
```sh
curl http://192.168.1.10:4534/
```

### Python
```python
import urllib.request, json
data = json.loads(urllib.request.urlopen("http://192.168.1.10:4534/").read())
print(data["satName"], data["elevationDeg"])
```

### JavaScript (browser / Node)
```js
const data = await fetch("http://192.168.1.10:4534/").then(r => r.json());
console.log(data.satName, data.elevationDeg);
```

---

## 12. Reference Implementation

The reference server implementation is **Look4Sat** (Android, this fork):

- Class: `Satlib` (`core/data/framework/Satlib.kt`)
- Interface: `ISatlib` (`core/domain/repository/ISatlib.kt`)
- Data model: `SatApiData` (`core/domain/model/SatApiData.kt`)

The server starts automatically with the application and runs on port 4534. Data is pushed from the radar screen's 1-second tracking loop while a satellite pass is active.
