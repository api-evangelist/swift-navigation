---
name: consume-skylark-corrections
description: >-
  Select a Skylark NTRIP caster, endpoint and mountpoint for a given region, receiver and reference
  frame, then open an authenticated correction stream and keep it alive.
api: Skylark Precise Positioning Service
generated: '2026-08-29'
method: generated
grounded_in:
  - examples/swift-navigation-skylark-ntrip-sourcetable-*.txt (probed, HTTP 200, gnss/sourcetable)
  - examples/swift-navigation-skylark-country-availability.json (probed, HTTP 200)
  - examples/swift-navigation-skylark-receiver-catalog.json (probed, HTTP 200, 134 receivers)
  - https://support.swiftnav.com/support/solutions/articles/44002519397-ntrip-client-configuration-for-skylark-corrections
operations:
  - 'GET http://<region>.<freq>.skylark.swiftnav.com:2101/            # sourcetable, no auth'
  - 'GET https://www.swiftnav.com/wp-json/gy9i81x4mh/data             # country availability, no auth'
  - 'GET https://www.swiftnav.com/wp-json/e4lm6SFEc9/data/            # receiver catalog, no auth'
  - 'GET http://<region>.<freq>.skylark.swiftnav.com:2101/<MOUNTPOINT> # stream, HTTP Basic'
---

# Consume Skylark corrections over NTRIP

Swift Navigation has no REST API for this. The whole flow is four HTTP requests, three of which need
no credentials.

## 1. Confirm the country is covered

```
GET https://www.swiftnav.com/wp-json/gy9i81x4mh/data
```

Returns a flat array of `{name, nx, cx, dx}`. If the country's flag for your intended variant is
`false`, stop here — no mountpoint will help.

## 2. Confirm the receiver is supported

```
GET https://www.swiftnav.com/wp-json/e4lm6SFEc9/data/
```

Returns `{"version":"v1","count":134,"items":[...]}`. Match on `brand_name` and `receiver_name`, then
read `compatibility_statement` for which Skylark variants apply and `configuration` for any
receiver-specific setup. The receiver must support NTRIP and RTCM 3.x.

## 3. Read the sourcetable to pick a mountpoint

Endpoint host is built, not looked up: `<region>.<freq>.skylark.swiftnav.com`, where region is
`NA`, `EU` or `AP` (case-insensitive) and freq is `L1L2`, `L1L5` or `all-freq`. Port 2101 is plain,
2102 is TLS, 2103 is mTLS and requires NTRIP v2.

```
GET http://na.all-freq.skylark.swiftnav.com:2101/
Ntrip-Version: Ntrip/2.0
```

`200`, `content-type: gnss/sourcetable`. Parse the `STR;` records — field 1 is the mountpoint name,
field 3 is the format, field 4 is the RTCM message list, field 6 is the constellation set.

**Choose the mountpoint by reference frame, not by convenience.** The datum is encoded in the name and
is not negotiable at runtime:

| Mountpoint | Frame | Where |
|---|---|---|
| `NXRTK-MSM5` | ITRF2020 current epoch | all regions |
| `NXRTK-NAD83-MSM5` | NAD83(2011) epoch 2010.0 | contiguous US only |
| `NXRTK-DREF91-MSM5` | ETRS89 DREF91(2016) epoch 1989.0 | Europe only |
| `NXRTK-ETRF2000-MSM5` | ETRS89 ETRF2000 epoch 2010.0 | Europe only |
| `NXRTK-WGS84-G1674-MSM5` | WGS84(G1674) epoch 2005.0 | all regions |
| `MSM4` / `MSM5` | ITRF2020 (Skylark Cx, wide-lane RTK) | all regions |
| `OSR` / `SSR` / `SSR-integrity` | ITRF2020, Swift proprietary encoding | all regions |
| `DX-MSM1` | ITRF2020 (Skylark Dx, code corrections) | all regions |

Picking a mountpoint in the wrong datum yields a position that is precise and wrong by up to a metre,
with no error signalled anywhere. This is the single most expensive mistake available on this API.

## 4. Open the stream

```
GET http://na.all-freq.skylark.swiftnav.com:2101/NXRTK-MSM5
Authorization: Basic <base64(ntrip_username:ntrip_password)>
Ntrip-Version: Ntrip/2.0
```

Credentials are per DEVICE, issued in the Skylark User Portal when the device is created. They are
not account credentials, and the password is displayed exactly once.

The connection stays open. The caster writes RTCM 3.x frames; you push NMEA GGA sentences containing
your approximate position back up the same connection. GGA is **required** for the `NXRTK-*`
mountpoints — they are virtual-reference-station streams and cannot generate corrections without
knowing roughly where you are.

## Failure handling

Swift Navigation documents no error reference for the caster, so you must infer:

- A sourcetable body returned where you expected a binary stream means the mountpoint name was
  wrong — mountpoint names are **case sensitive**.
- `401` means the credential is wrong, expired, or its subscription lapsed. These are
  indistinguishable from the response; check the portal.
- There is no status page. A stream that stops delivering has no public place to check, and no
  rate-limit or Retry-After header is ever returned.

## Before you act

- Nothing in this skill mutates anything. All four calls are reads.
- Do **not** rotate an NTRIP password to "fix" a connection problem. Rotation is irreversible and
  breaks every receiver already configured with the old credential.
