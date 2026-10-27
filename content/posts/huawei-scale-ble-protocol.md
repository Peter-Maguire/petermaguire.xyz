---
title: "Reverse Engineering Huawei's Smart Scale Protocol"
date: 2026-10-27T12:00:00+00:00
draft: false
toc: false
images:
tags:
  - reverse-engineering
  - bluetooth
---

**Note: Source code for a Home Assistant integration built from this research is available [here](https://github.com/Peter-Maguire/hacs-huawei-scale)**

# Introduction

Huawei sells a range of Bluetooth body composition scales: the Scale 3, Scale 3 Pro, Scale 2 Pro, a few Honor-branded ones and some cheaper overseas models.
They all work with the Huawei Health app, which pairs with the scale, sends it your age, height and sex, and pulls back your weight, body fat percentage and a pile of other numbers.

I didn't want to install yet another app and get locked into an ecosystem just to use a set of scales.
What I actually wanted was for the scales to keep a history by themselves, without me having to pull my phone out every time I stepped on them, and the obvious place for that is [Home Assistant](https://www.home-assistant.io/).
So I needed to find out how the Huawei Health app talks to them.

# The App

Huawei Health has been pulled from the Play Store, and these days the only way to get it is to download an APK directly from Huawei's website.
That's not a great experience for anyone who just bought a scale, and it's a big part of why I wanted to avoid the app in the first place.
It did mean I had an APK in my hands though, and unlike the [last time I tried this](/posts/zhiyun-weebil-s-ble-protocol/) it decompiled just fine.

I ran Huawei Health (`com.huawei.health`) through [jadx](https://github.com/skylot/jadx).
It's a big app, around 50 dex files, and most of the class names are obfuscated into things like `dqb` and `dpy`.
The names I use below are ones I gave them after working out what they do.

I started from the list of supported devices in `SupportDevices.xml`, which maps the BLE advertising name of a scale to a product:

| BLE name contains | Device |
|---|---|
| `Dobby`, `HUAWEI Scale 3 BLE-` | Huawei Scale 3 |
| `Hagrid-B29`, `HUAWEI Scale 3 Pro-` | Huawei Scale 3 Pro |
| `HAG`, `HUAWEI Scale 2 Pro-` | Huawei Scale 2 Pro |
| `CH100` / `CH100S`, `CH18`, `AH100` | Huawei / Honor body-fat scales |
| `LUP-`, `AH111`, `HONOR Scale 2-` | Honor Scale 2 |

Following those names through the code, it turned out there are two completely different protocols hiding behind the one app.

# Path 1: The Chipsea Scales

The cheaper scales aren't really Huawei's at all.
They use the Chipsea "OKOK" chipset, and the app bundles Chipsea's entire third party SDK (`com.careforeyou.library`) to talk to them.
Huawei's own glue code is a small class that I'd guess is called `ChipSeaMeasureController`.

The most interesting thing about these scales is that you don't even need to connect to them.
The scan doesn't filter on any service UUIDs, it just matches on MAC address or name afterwards, and a separate class parses weight, impedance and a "lock" flag (meaning the reading has settled) directly out of the manufacturer data in the advertisement.

If you do connect, the GATT layout is:

| Service | Characteristic | Purpose |
|---|---|---|
| `fff0` | `fff1` | Notify: live weight |
| `fff0` | `fff2` | Write without response: commands to the scale |
| `181b` | `2a9c` | Standard body composition measurement |
| `181b` | `fa9c` | Custom: history |
| `1805` | `2a08` | Date/time, used to sync the clock |

The SDK supports several protocol variants, picked by a factory based on the scale:

| Variant | Frame header | Notes |
|---|---|---|
| OKOK | `0xCA` | Command `0x10` is a live frame with fat, water, muscle, BMR, visceral fat and bone as 16-bit values. `0x11` is weight, history or user match. |
| OKOKCloud | `0x02 0x00`, `0x06 0x03`, `0x06 0x23` | Impedance at bytes 9-10 |
| OKOKCloudV3/V4 | `0xC0`, `0xC4`, `0xC5`, `0xC8` | Impedance at bytes 8-9 (x0.1), and optional 5 segment impedance in `0xC8` |

On the Cloud variants the scale only sends weight and impedance, and the body fat figures are calculated somewhere else.
I didn't trace whether that is the app or Huawei's servers.

Weight is decoded in one place, and a units byte decides whether you're looking at kg, jin (multiply by 0.5), pounds (multiply by 0.4535924) or stones and pounds.

Going the other way, the app sends the scale a 20 byte frame which sets the clock and the user profile in one go:

```
CA 10 0E 01 <yy mm dd hh mm ss> <userId:4> <sex|age> <age> <height> <xor checksum>
```

On the OKOK variant the scale uses the sex, age and height to do its own body composition maths.
There are also frames for sending the user list and for setting up a user with their last weight (`A0 <id:4> <sex|age> <height> <weight*10>`).
Clock sync can be done either with a 7 byte date (year as a big-endian short, then month, day, hour, minute, second) or a plain 4 byte unix timestamp.

All of this is delightfully plain. No encryption, no authentication, just frames with an XOR checksum.

# Path 2: Huawei's Own "Hagrid" Scales

The Scale 2 Pro and Scale 3 Pro are completely different and propriatary.
The package is named `hagrid` (matching the Scale 3 Pro advertising as `Hagrid-B29`), it's built on a generic Huawei device framework with commands, acknowledgements and an "encrypt" flag that defaults to true.
The Home Assistant integration I built from this targets the Scale 3; the Pro models use a closely related protocol, but I haven't tried them.

## GATT

The scale exposes the usual standard services: Weight Scale (`181d`), Body Composition (`181b`), User Data (`181c`) and Current Time (`1805`).
The app uses the standard characteristics for setting the age, gender, height and the time, but everything interesting lives in custom characteristics hung off those services.

On User Data (`181c`) are the ones used for pairing and managing users:

| Task | Characteristic UUID |
|---|---|
| REQUEST_AUTH | `02b2a08e-f8b0-4047-b1fd-f4e0efeee679` |
| AUTH_TOKEN | `32330a04-15d9-421a-91c5-2a2d5c7525c9` |
| SEND_WORK_KEY | `a3d330f8-b84f-4f48-a78c-f8d1e33b597a` |
| BIND_REQUEST | `42596cbe-d291-4da3-8ca6-d1ae5d1c9174` |
| SET_USER_INFO | `8cc61d7d-66c0-4802-89c3-38c5a163592e` |
| SEND_GET_USER_DATA | `78bdadc1-5cae-4ec4-8832-21a020b1080b` |
| SEND_DELETE_USER_DATA | `7749db19-9db5-4430-b6e2-5fcfb3f5b988` |

Under Current Time (`1805`) are the Wi-Fi provisioning and settings ones (these scales can upload to Huawei's cloud by themselves):

| Task | Characteristic UUID |
|---|---|
| SEND_SSID | `33ff0cad-834f-4e42-ab91-a0747856d574` |
| SEND_WIFI_PASSWORD | `b4e384f2-d880-44e7-8c7e-5249239633b4` |
| GET_DEVICE_SSID | `75143e79-f878-4a00-a628-edc40509de7e` |
| GET_SCALE_VERSION | `1f5d3d5c-496d-4290-af03-c7a8d5419741` |
| SET_WEIGHT_UNIT | `11872f15-a91d-49da-ac89-5107284f3425` |
| SEND_WAKE_UP | `e3c7a876-3307-414d-84f6-fbcd64710dc3` |

And on Body Composition (`181b`) are the two that actually matter if you just want your weight:

| Task | Characteristic UUID |
|---|---|
| GET_WEIGHT_REAL_TIME_DATA | `46797c17-d639-488d-9476-4789e8472878` |
| GET_WEIGHT_HISTORY_DATA | `0212f42a-5f19-4bc1-ba52-d7ec7ccb71a4` |

There's also a firmware update service on `fff0` (`fff1` to `fff4` for request, SHA check, data and result) and a file transfer one on `fe86`.
The full table has a few more entries, but those are the ones I'd start with.

The app always enables notifications on a characteristic, waits 100ms, and then writes to it.

## Framing

Payloads are cut into chunks of 222 bytes if encrypted or 238 if not, and each chunk gets a two byte prefix of `[totalChunks-1][chunkIndex]`.
Those chunks are then cut further into BLE sized frames:

- The first byte is `0xDC` (encrypted) or `0xDB` (plain) when sending, and `0xCD` (encrypted) or `0xBD` (plain) when receiving.
- Then a length byte (on receive, the data length is this minus 3).
- Then a counter packed into one byte, with the frame index in the low nibble and the frame total in the high nibble.
- Up to 15 bytes of data.
- A two byte CRC16-CCITT (polynomial `0x1021`) at the end.

Fifteen bytes per frame is tiny, but it presumably keeps everything within the default 23 byte BLE MTU.

## Encryption

Encrypted messages are AES-128-CTR with no padding.
Every message gets a fresh random 16 byte IV, and the message on the wire is `IV || ciphertext`.

The key for this is called the **work key**, and it's a very low-tech thing: 16 random characters from `[a-z0-9]`, generated by the app once per scale and kept in a static `HashMap` in memory.
It's never written to disk, so it's gone as soon as the app restarts and the next connection makes up a new one.
That's quite nice from a security point of view, but it means the scale has to be told what the key is every single time, which is where the rest of the machinery comes in.

## Authentication

Before the app will hand over the work key, there's a mutual challenge-response handshake using HMAC-SHA256:

1. The scale supplies `randA`, 16 random bytes, after the app writes to REQUEST_AUTH.
2. The app generates its own `randB`.
3. The app sends `randB || T("1123")` to AUTH_TOKEN.
4. The scale replies with a token, and the app only carries on if it matches `T("9856")`.

Where `T(X)` is, with `RR = randA || randB`:

```
inner = HMAC_SHA256(key = CAK || ASCII(X), msg = RR)
T(X)  = HMAC_SHA256(key = inner,           msg = RR)
```

`CAK` here is a 16 byte secret baked into the app, and the markers `"1123"` and `"9856"` make sure that a response can't just be replayed back at its sender.

The markers were an amusing thing to track down.
In the decompiled code they didn't appear as strings, they appeared as `TarConstants.LF_LINK`, `LF_SYMLINK` and so on, because jadx decided the byte constants `0x31 0x31 0x32 0x33` looked an awful lot like the Apache Commons Compress tar file type flags and helpfully inlined them.
Translating them back gave me the ASCII digits.

## Wrapping Key

Once the scale has authenticated, the app sends it the work key on SEND_WORK_KEY.
That message is itself encrypted with a second key, the **wrapping key**, derived like this (with `mac` being the scale's Bluetooth address without the colons):

```
c3   = ASCII(mac + "0000")                       (16 bytes)
x[i] = (A[i] << 4) ^ B[i]
h    = SHA256(x)[0:16]
r[i] = (h[i] >> 6) ^ c3[i]
wrapKey = SHA256(r)[0:16]
```

`A` and `B` are two more 16 byte secrets from the same place as the CAK.
I have to say this is a strange construction. The shift by 6 means only the top two bits of each byte of `h` make it into `r`, so the final key is really the MAC address with a small, fixed, device-independent mask XORed over it before one last hash.
I'd be surprised if that is what the author intended, but it's what the code does.

## Where the Secrets Come From

The three secrets (CAK, A and B) aren't in the Java code.
They're fetched through a JNI call into `libwhiteBoxJniLib-arm64-v8a.so`, which first checks the signing certificate of the calling APK against around 13 hardcoded hashes, and refuses to hand anything over if it doesn't match.

That check only guards the *door*, though.
Opening the library in Ghidra, the three values turn out to be plain 32 character hex strings sitting in the binary, which the Java side decodes into 16 raw bytes.
Which string is which is chosen by a switch statement for the CAK and a table of relative offsets for the other two, so I cross checked the results by evaluating the pointer arithmetic in Ghidra and by regex scanning the raw strings, and both agreed.

## Measurement Data

Once you've got past all of that, the data itself is blessedly simple.
A live or history record has:

| Bytes | Value |
|---|---|
| 0-1 | Weight, little-endian u16, divide by 100 for kg |
| 2-3 | Body fat, u16, divide by 10 for % |
| 4-11 | Timestamp |
| 24-25 | Heart rate, u16 |

There are segmental impedance values after that which I haven't decoded.
The app's data model has left/right hand and foot fields (`lfrf`, `lhrh`, `lhlf`, `lhrf`, `rhlf`, `rhrf`) as well as high-frequency variants, so the 8-electrode scales are sending raw segmental impedance and the app does the work of turning it into limb fat and muscle mass.

The "mini user" record that the scale stores for each person looks like this:

| Offset | Size | Field |
|---|---|---|
| 0 | 2 | Weight (/100) |
| 2 | 12 | Resistance |
| 14 | 32 | huid (Huawei user ID) |
| 46 | 1 | Sex |
| 47 | 1 | Age |
| 48 | 2 | Height |
| 50 | 1 | isOver |
| 51 | 1 | isMainId |
| 52 | 8 | Time |

## Firmware Updates

Firmware updates use a separate TLV channel (service ID 9) layered on top of the same encrypted framing, with a tag byte, a length byte and a value.
The app sends the file name, size and type, negotiates a packet size and interval with the scale, streams the data with a bitmap of received packets, and the scale reports a result at the end.
I haven't gone any deeper than the command list.

# Prior Art

I wasn't the first person to look at this.
[openScale](https://github.com/oliexdev/openScale) is an open source Android app for tracking weight and body measurements from Bluetooth scales without needing an account, and it's the reference for how a lot of scale protocols work.
Its Huawei driver is the basis for the Home Assistant integration, which is a port of it.
It was also a handy source of known good data, since the framing, CRCs, key derivation and decoding can all be checked against a real capture.

On the band side, [Honor Band 4 notes](https://gist.github.com/psolyca/68fdfee0960737c1234f8d7f7a6c143a) cover the same app, which uses a totally different protocol but shares the same `libwhiteBoxJniLib.so`.
It confirmed that the secrets are stored split across the binary, and also pointed at an easier route for bands: the app caches a per-device master key in `SharedPreferences`, in plain text before version 10.0.1.319.

I checked whether the scale code does the same, and it doesn't, so that shortcut wasn't available.
The only preferences files the weight code touches hold a user ID, some UI state and a per-MAC flag for the Chipsea scales, and the work key lives only in memory.
There's nothing to pull off a rooted phone, and the white box library really is the only way in.

# The Home Assistant Integration

With the protocol worked out, the end result is a [HACS integration](https://github.com/Peter-Maguire/hacs-huawei-scale) that does the job of the app.
The scale only advertises while someone is standing on it, so the integration waits for that, then connects, authenticates, sets the user profile, waits for the measurement, publishes it and disconnects.
It goes through Home Assistant's own Bluetooth stack, so a local adapter or an ESPHome Bluetooth proxy both work.

Each weigh-in turns into Home Assistant sensors (weight, BMI, body fat, body water, muscle, bone mass, visceral fat and so on), and because the scale has no idea who just stepped on it, you pick the person for the next weigh-in from a select entity.
Home Assistant keeps the history, so no phone is needed.

