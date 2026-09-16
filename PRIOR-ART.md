# Design provenance and prior art

This file records when the mechanisms of the Sendspin protocol were first published in this repository, and the public prior art they build on. It exists so that anyone can establish, from the git history, what was public and when - for example to contest a later patent application that claims one of these mechanisms - and so that implementers can see that the protocol deliberately assembles long-published techniques.

Everything in this repository has been public on GitHub since the first commit. Commits are in the `Sendspin/spec` repository at <https://github.com/Sendspin/spec>. The project was developed under the working title Resonate until `e1a3a88` (2025-12-04); earlier commits, service types, and links use that name, and the former `Resonate-Protocol` GitHub paths redirect to `Sendspin`. Git commit dates are set by the committer and are not proof on their own. GitHub's server-side timestamps corroborate them: each pull request cited below records when it was opened and merged, and the opening date is often the earlier public disclosure (#69, for example, was opened on 2026-02-26 and merged on 2026-06-01). Hashes are abbreviated; resolve them to full SHAs or GitHub commit URLs when citing. Independent snapshots of the repository, such as a Software Heritage archive, strengthen the record further where they exist. The table below is a convenience index.

## Publication timeline

First appearance of each mechanism in the public repository (`git log --reverse -S<term>`). Where a mechanism was later renamed, the row cites the commit that introduced it under its original name and notes the rename; the day-one specification already carried the NTP-style four-timestamp time exchange (three timestamps on the wire, the fourth taken on receipt) and timestamped audio chunks.

| Mechanism | First commit | Date |
|---|---|---|
| Repository created; timestamped audio chunks; client/server time exchange with `server_received` / `server_transmitted`; per-client `buffer_capacity`; artwork role | `df40233`, `5d7252f`, `2bbe31a` | 2025-06-05 |
| Visualizer role; `stream/request-format` per-client format negotiation; group volume field; mDNS discovery with the `_resonate._tcp` and (from `bdc59c5`, 2025-09-22) `_resonate-server._tcp` service types, renamed `_sendspin._tcp` / `_sendspin-server._tcp` in `e1a3a88` (2025-12-04) | `9e42404` | 2025-09-15 |
| `group/switch` command moving a client to another group; moved to the controller role's `switch` command in `e8babf3` (2025-10-09, merged in #26) | `cac5e56` | 2025-09-17 |
| Kalman-filter clock offset and drift tracking recommended, with the time-filter library as reference implementation | `b26d14c` | 2025-11-17 |
| `switch` command cycle through groups | `879d8b2` | 2025-11-17 |
| Group volume model (delta, clamp, redistribute) | `4f2a0d6`, `5f98440` | 2025-11-17 / 2025-11-20 |
| `stream/flush` (seek and track-jump without ending the stream), renamed `stream/clear` in `cfef5c4` (2025-12-01) | `04bfc8c` | 2025-11-26 |
| Perceived-loudness volume scale | `01a9e5b` | 2025-12-01 |
| External-source handling (`state: 'external_source'` moves a client to a solo group; previous-group priority in the switch cycle), expressed as `available: false` since `e7cf66a` (2026-07-07) | `5cccacf` | 2025-12-12 |
| `static_delay_ms` output-delay compensation, renamed `output_delay_ms` in `485617f` (2026-08-20) | `f686efa` | 2026-01-30 |
| Color role (palette derived from artwork, scheduled by timestamp) | `a8f8f67`, `fad6e9c` | 2026-03-04 / 2026-04-15 |
| Time-filter algorithm made the required synchronization method | `0efbddb` | 2026-04-10 |
| Noise `KKpsk2` encryption, pre-shared keys, Sentinel PSK, CPace pairing with commit-and-reveal (`commit_B`), dynamic and static PIN methods with pairing window and failure counter (renamed pairing code in `9a526a4`, 2026-08-18), QR-code pairing | `131dc9b` | 2026-05-06 |
| Unpaired playback on the Sentinel PSK (unpaired access; adjusted in `d968604`, operator approval flow in `e8f7a9e`, 2026-08-25) | `1c3bcec` | 2026-05-19 |
| `visualizer@v1` role as specified today, with spectrum configuration | `10cdfc6` | 2026-05-21 |
| `required_lead_time_ms` (pull request #69, opened 2026-02-26) | `133383e` | 2026-06-01 |
| Source role (line-in captured, timestamped in the server clock domain) with `client_stream/start`, renamed `client-stream/start` in `092ffb1` (2026-08-24); volume-to-amplitude curve; sample deletion/insertion correction strategy and sync accuracy requirements | `0f5a9b3`, `6453922`, `de382cd` | 2026-06-30 |
| Pairing token format (base32, `SP:` prefix) | `d9154c4` | 2026-07-29 |
| `send_ahead` arrival-delay measurement | `bda23f3` | 2026-08-28 |

## Prior art the design builds on

Sendspin does not claim to have invented synchronized multi-device playback, clock synchronization, timestamp-scheduled output, or code-based pairing. It combines techniques published in open standards and open-source software long before this repository existed. The list is non-exhaustive and deliberately limited to open standards and open-source projects, with approximate first-publication dates:

**Timestamped playback and clock synchronization**

- Network Time Protocol: the four-timestamp offset/delay exchange used by `client/time` and `server/time` is the NTP client-server exchange (RFC 1305, 1992; RFC 5905, 2010).
- Kalman filtering of clock offset and skew: standard in the clock-synchronization literature since the 1990s and in PTP (IEEE 1588, 2002) servo implementations.
- PulseAudio RTP sender/receiver modules (mid-2000s).
- Snapcast (2015): open-source server that sends timestamped, per-client encoded audio (PCM, FLAC, Opus, Vorbis) to clients that translate server time to local time and correct drift by sample insertion/removal.
- Roc Toolkit (2017): open-source real-time audio transport with clock recovery and latency tuning.
- Per-device output-delay ("lip-sync") adjustment: VLC's audio desynchronization control and Kodi's audio offset (2000s).

**Groups and control**

- Grouping clients for synchronized playback with per-group control: PulseAudio `module-combine-sink` (late 2000s); Snapcast groups (2017).

**Timestamped light and visual control**

- DMX512 (1986), MIDI Show Control (1991), Art-Net (1998), SMPTE timecode: scheduling lighting changes to a shared time base.
- Ambient lighting derived from and synchronized to screen content: boblight (2010), Hyperion (2013).
- Music visualizers driven by loudness, spectrum, and beat features: XMMS visualization plugins (late 1990s), projectM (2003 onward).
- Extracting dominant and accent colours from images for theming: Color Thief (2011), Vibrant.js (2015).

**Pairing and transport security**

- Noise Protocol Framework (Trevor Perrin, 2016 onward), including PSK modifiers and the `KK` pattern.
- CPace (CFRG, draft-haase-cpace 2019/2020, draft-irtf-cfrg-cpace) with mutual confirmation.
- Short authentication strings with commit-and-reveal binding to a key exchange: ZRTP (2006; RFC 6189, 2011). Numeric comparison pairing: Bluetooth Secure Simple Pairing (Core 2.1, 2007).
- Setup codes and QR codes for device onboarding with a PAKE: Matter (Project CHIP, 2019; Matter 1.0, 2022).
- mDNS and DNS-SD (RFC 6762 / 6763, 2013); WebSocket (RFC 6455, 2011).

**Codecs and formats**

- Opus (RFC 6716, 2012), FLAC (2001), PCM, JPEG (1992), PNG (1996), base32/base64 (RFC 4648).

## How to use this file

If you encounter a patent application that appears to claim a mechanism first published here, cite the commit hash and date from the table above together with the repository URL; the full history is available with `git log --follow` on the relevant source file. Additions to this file are welcome, in particular earlier open standards or open-source prior art for any mechanism listed.
