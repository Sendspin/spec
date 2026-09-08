# Design provenance and prior art

This file records when the mechanisms of the Sendspin protocol were first published in this repository, and the public prior art they build on. It exists so that anyone can establish, from the git history, what was public and when - for example to contest a later patent application that claims one of these mechanisms - and so that implementers can see that the protocol deliberately assembles long-published techniques.

Everything in this repository has been public on GitHub since the first commit. Git commit dates are authoritative; the table below is a convenience index. Commits are in the `Sendspin/spec` repository at <https://github.com/Sendspin/spec>.

## Publication timeline

First appearance of each mechanism in the public repository (`git log --reverse -S<term>`; earlier commits may describe the same mechanism under a different name - the day-one specification already carried the three-timestamp time exchange and timestamped audio chunks).

| Mechanism | First commit | Date |
|---|---|---|
| Repository created; timestamped audio chunks; client/server time exchange with `server_received` / `server_transmitted`; per-client `buffer_capacity`; artwork role | `df40233`, `5d7252f`, `2bbe31a` | 2025-06-05 |
| Visualizer role; `stream/request-format` per-client format negotiation | `9e42404` | 2025-09-15 |
| `switch` command cycling a client through groups | `cac5e56` | 2025-09-17 |
| Group volume model (delta, clamp, redistribute) | `4f2a0d6`, `5f98440` | 2025-11-17 / 2025-11-20 |
| Kalman-filter clock offset and drift tracking named as the required method | `b26d14c` | 2025-11-17 |
| `stream/clear` (seek and track-jump without ending the stream) | `cfef5c4` | 2025-12-01 |
| mDNS service types `_sendspin._tcp` / `_sendspin-server._tcp`; server- and client-initiated connections | `e1a3a88` | 2025-12-04 |
| Color role (palette derived from artwork, scheduled by timestamp) | `a8f8f67`, `fad6e9c` | 2026-03-04 / 2026-04-15 |
| Time-filter library referenced as the normative synchronization method | `0efbddb` | 2026-04-10 |
| Noise `KKpsk2` encryption, pre-shared keys, Sentinel PSK, CPace pairing with commit-and-reveal (`commit_B`) | `131dc9b` | 2026-05-06 |
| Spectrum configuration for the visualizer | `10cdfc6` | 2026-05-21 |
| `required_lead_time_ms` | `133383e` | 2026-06-01 |
| Unpaired access with operator approval | `d968604` | 2026-06-17 |
| Source role (line-in captured, timestamped in the server clock domain); perceived-loudness volume curve; sample deletion/insertion correction strategy and sync accuracy requirements | `0f5a9b3`, `6453922`, `de382cd` | 2026-06-30 |
| External-source handling (`available: false` moves a client to a solo group) | `e7cf66a` | 2026-07-07 |
| Pairing token (base32, `SP:` prefix, QR) | `d9154c4` | 2026-07-29 |
| Dynamic and static pairing code methods with pairing window and failure counter | `9a526a4` | 2026-08-18 |
| `output_delay_ms` compensation | `485617f` | 2026-08-20 |
| `client-stream/start` for source streams | `092ffb1` | 2026-08-24 |
| `send_ahead` arrival-delay measurement | `bda23f3` | 2026-08-28 |

## Prior art the design builds on

Sendspin does not claim to have invented synchronized multi-device playback, clock synchronization, timestamp-scheduled output, or code-based pairing. It combines techniques that were published long before this repository existed. Non-exhaustive, with approximate first-publication dates:

**Timestamped playback and clock synchronization**

- Network Time Protocol: the four-timestamp offset/delay exchange used by `client/time` and `server/time` is the NTP client-server exchange (RFC 1305, 1992; RFC 5905, 2010).
- Kalman filtering of clock offset and skew: standard in the clock-synchronization literature since the 1990s and in PTP servo implementations.
- Sonos, "System and method for synchronizing operations among a plurality of independently clocked digital data processing devices", US provisional filed July 2003, published 2005: a distribution device sends time-stamped tasks and members compute a clock differential to execute them locally; late-joining members receive future-stamped data; sample insertion/deletion corrects drift. Expired 2024.
- Implicit Networks, "Method and system for synchronization of content rendering", filed December 2001: master/slave rendering time versus device time. Expired 2022.
- Apple AirPlay / RAOP (2004): RTP audio with NTP-style timing to AirPort Express receivers.
- Slim Devices SlimServer / Squeezebox synchronized playback (2005 onward), later Logitech Media Server and Squeezelite.
- PulseAudio RTP sender/receiver modules (mid-2000s).
- Snapcast (2015): open-source server that sends timestamped, per-client encoded audio (PCM, FLAC, Opus, Vorbis) to clients that translate server time to local time and correct drift by sample insertion/removal.
- Roc Toolkit (2017): open-source real-time audio transport with clock recovery and latency tuning.
- Per-device output-delay ("lip-sync") adjustment: standard on AV receivers and media players since the 2000s (VLC audio desynchronization control, Kodi audio offset, AV-receiver lip-sync menus).

**Groups and control**

- Multi-zone grouping, group volume, and joining a playing zone from a device's own controls: shipped in Sonos (2005-2014), Squeezebox, and other systems; controller-less join from a device button is described in Sonos literature from 2014.

**Timestamped light and visual control**

- DMX512 (1986), MIDI Show Control (1991), Art-Net (1998), SMPTE timecode: scheduling lighting changes to a shared time base.
- Philips Ambilight (2004) and Philips amBX (2006): lighting derived from and synchronized to media content; Philips Hue Entertainment streaming API (2018).
- Music visualizers driven by loudness, spectrum, and beat features (Winamp/MilkDrop and predecessors, late 1990s).
- Extracting dominant and accent colours from artwork for UI theming (Android Palette API, 2014; Material You, 2021).

**Pairing and transport security**

- Noise Protocol Framework (Trevor Perrin, 2016 onward), including PSK modifiers and the `KK` pattern.
- CPace (CFRG, draft-haase-cpace 2019/2020, draft-irtf-cfrg-cpace) with mutual confirmation.
- Short authentication strings with commit-and-reveal binding to a key exchange: ZRTP (2006; RFC 6189, 2011). Numeric comparison pairing: Bluetooth Secure Simple Pairing (Core 2.1, 2007).
- Setup codes and QR codes for device onboarding with a PAKE: Apple HomeKit (2014), Matter (Project CHIP 2019, Matter 1.0 2022).
- mDNS and DNS-SD (RFC 6762 / 6763, 2013); WebSocket (RFC 6455, 2011).

**Codecs and formats**

- Opus (RFC 6716, 2012), FLAC (2001), PCM, JPEG (1992), PNG (1996), base32/base64 (RFC 4648).

## How to use this file

If you encounter a patent application that appears to claim a mechanism first published here, cite the commit hash and date from the table above together with the repository URL; the full history is available with `git log --follow` on the relevant source file. Additions to this file are welcome, in particular earlier public prior art for any mechanism listed.
