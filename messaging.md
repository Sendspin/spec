## Communication

Once the WebSocket connection is established, Client and Server perform an initial handshake before exchanging application data:

1. Client → Server: [`client/init`](#client--server-clientinit) (cleartext)
2. Server → Client: [`server/init`](#server--client-serverinit) (cleartext)
3. Server → Client: [`noise/handshake`](#client--server-noisehandshake) - Noise message 1 (cleartext)
4. Client → Server: [`noise/handshake`](#client--server-noisehandshake) - Noise message 2 (cleartext)
5. Both sides switch to Noise transport mode. From this point, all Sendspin application data is sent as WebSocket binary messages whose payloads are Noise transport messages.
6. Server → Client: [`server/hello`](#server--client-serverhello) (encrypted)
7. Client → Server: [`client/hello`](#client--server-clienthello) (encrypted)
8. Server → Client: [`server/activate`](#server--client-serveractivate) (encrypted)

The server MUST NOT send other Sendspin messages until it sends the initial [`server/activate`](#server--client-serveractivate), except for [`server/error`](#server--client-servererror) sent in place of `server/init` on an init failure. The client MUST NOT send other Sendspin messages until it receives that activation, except that it MAY send an encrypted [`client/goodbye`](#client--server-clientgoodbye) once the initial Noise handshake has completed. Silent failures defined in [Failure Handling](connection.md#failure-handling) still close the connection without sending any further message. See [Encryption](connection.md#encryption) for cryptographic details.

Cleartext handshake messages (`client/init`, `server/init`, `noise/handshake`, `server/error`) are each sent as one complete WebSocket **text** message containing JSON. After the encrypted channel is established, all messages are sent as WebSocket **binary** messages carrying Noise transport messages.

WebSocket messages may span multiple RFC 6455 frames. Sendspin operates only on complete WebSocket messages. This WebSocket fragmentation is distinct from Sendspin [fragmentation](#fragmentation).

WebSocket control frames (Ping, Pong, Close; RFC 6455) are not Sendspin messages: they remain valid at any time, are not encrypted at the Noise layer, and Ping/Pong is the expected connection-liveness mechanism.

In field definitions, `?` indicates an optional field (e.g., `field?`: type means the field may be omitted).

All JSON messages have a `type` field identifying the message and a `payload` object containing message-specific data. The payload structure varies by message type and is detailed in each message section below.

**Message type prefixes.** The prefix before the `/` in a message `type` identifies a group of messages. `client/` and `server/` name the sender. `stream/` groups the messages that control a binary channel from the server to the client, and `client-stream/` those that control a binary channel from the client to the server; the two kinds of channel are independent and have separate lifetimes. `group/`, `pair/`, and `noise/` name a subject. Only `client/`, `server/`, and `client-stream/` imply a direction; for every other prefix each message's definition gives it.

**Forward compatibility.** Clients and servers MUST ignore unrecognized `payload` fields (keys not defined for the message) rather than treating them as an error. Clients and servers MUST NOT send fields the specification does not define for the message, other than the `_`-prefixed [application-specific role](README.md#application-specific-roles) objects a message explicitly permits, or support objects in `client/hello` for advertised application-specific role versions (e.g., `player@_experimental_support`).

Clients and servers MUST ignore JSON messages with an unrecognized `type`, provided the message is a valid JSON object with a `type` string and a `payload` object. They MUST also ignore binary messages whose ID they do not implement. These rules apply only when the [initial handshake](#communication) and [re-handshake](connection.md#re-handshake) rules allow application messages.

Before sending messages or fields for a feature added by a future revision of this specification, senders MUST confirm support through role activation or explicit capability negotiation. The feature's specified negotiation fields are exempt from this requirement. Not receiving an error does not prove that the receiver understood or acted on the message. Future revisions of this specification MAY define negotiation fields in existing messages that older receivers can ignore. See [Protocol evolution](README.md#protocol-evolution).

Noise authentication and the validation, direction, and sequencing rules for recognized messages still apply. An ID the receiver implements is still recognized when its role is inactive. An unrecognized value in a known field follows that field's rules, not the rule for ignoring unrecognized fields.

Message format example:

```json
{
  "type": "stream/start",
  "payload": {
    "server_transmitted": 1234567890,
    "player": {
      "codec": "opus",
      "sample_rate": 48000,
      "channels": 2,
      "bit_depth": 16
    },
    "artwork": {
      "channels": [
        {
          "source": "album",
          "format": "jpeg",
          "width": 800,
          "height": 800
        }
      ]
    }
  }
}
```

WebSocket binary messages are used to send JSON payloads, audio chunks, media art, and visualization data. Each complete binary message carries exactly one Noise transport message; after AEAD decryption, the first byte is a uint8 representing the message type. Throughout this specification, bit 0 refers to the least significant bit.

Receivers that discard a binary role payload MUST still process Noise messages, follow the fragmentation rules below, and perform the role's required message checks.

### Binary Message ID Structure

The first byte of every decrypted binary message is its message ID. IDs are assigned from the table below; each role's binary message definitions name the exact IDs it uses.

| IDs | Assignment |
|---|---|
| 0 | JSON message body (UTF-8) |
| 1 | [Fragmentation](#fragmentation) |
| 2-3 | Reserved for future use |
| 4-7 | Player role |
| 8-11 | Artwork role |
| 12-15 | Source role |
| 16-23 | Visualizer role |
| 24-191 | Reserved for future roles |
| 192-255 | Available for use by [application-specific roles](README.md#application-specific-roles) |

Future roles will be allocated aligned blocks of 4 or 8 IDs from the reserved 24-191 range.

**Note:** Role versions share the same binary message IDs (e.g., `player@v1` and `player@v2` both use IDs 4-7).

### Fragmentation

A single Noise transport message is limited to 65535 bytes by the Noise specification. Both defined cipher suites use a 16-byte AEAD authentication tag, and the message type byte occupies the first byte of the AEAD plaintext, so the application payload per Noise transport message is at most 65535 − 16 − 1 = 65518 bytes. Larger messages MUST be split across multiple WebSocket binary messages using the fragment message type.

**Wire format** (inside the AEAD-protected plaintext of each fragment message):

- First fragment: `[1][flags][orig_type][data]`
- Subsequent fragments: `[1][flags][data]`

`flags` is a uint8. Bit 1 is set on the first fragment of a message and bit 0 on the last. Bits 2-7 are reserved and MUST be zero.

The concatenated `data` from all fragments yields the original message's payload (the bytes that would have followed the message type byte in a non-fragmented message of type `orig_type`).

**Constraints:**

- Only one fragmented message may be in flight at a time per direction. A sender MUST finish a fragmented message with a last fragment before sending any other binary message in that direction, whether fragmented or not.
- Senders SHOULD NOT fragment messages that fit in a single Noise transport message.
- A sender MUST NOT use `1` as `orig_type`.

**Receiver behavior:** maintain a single reassembly buffer along with the in-flight `orig_type`. On a first fragment, read `orig_type` from byte 2 and start a new buffer with its `data`; on any other fragment, append its `data` to the buffer. When bit 0 is set, dispatch the buffer as a single message of type `orig_type` and clear it.

The [ignore rules](#communication) also apply to fragmented messages. If the receiver does not implement `orig_type`, it MAY discard each fragment's `data` instead of allocating a reassembly buffer. It MUST still authenticate every Noise transport message, track the fragment sequence, and enforce the malformed-sequence rules below. The last fragment clears the sequence state. The discarded message is not dispatched.

**Malformed sequences** are protocol errors; the receiver MUST close the connection. They are: a first fragment received while a fragmented message is in flight, a non-first fragment received with none in flight, a non-fragment binary message received while a fragmented message is in flight, a nonzero reserved flag bit, and an `orig_type` of `1`.

## Clock Synchronization

Clients send `client/time` messages to maintain an accurate mapping between their clock and the server's clock. Implementations MUST send these messages frequently enough to keep the filter convergent. The time-filter library's [Recommended Usage](https://github.com/Sendspin-Protocol/time-filter#recommended-usage) section describes a known-good burst-strategy baseline.

Binary audio messages contain timestamps in the server's time domain indicating when the audio should be played. Clients MUST use the [time-filter](https://github.com/Sendspin-Protocol/time-filter) algorithm to translate server timestamps to their local clock for synchronized playback. The time filter is a two-dimensional Kalman filter that tracks both clock offset and drift. See the [time-filter](https://github.com/Sendspin-Protocol/time-filter) repository for a C++ reference implementation and [aiosendspin](https://github.com/Sendspin-Protocol/aiosendspin/blob/main/aiosendspin/client/time_sync.py) for a Python implementation.

Each [`server/time`](#server--client-servertime) response provides the four timestamps needed by the filter: the client's transmitted timestamp, the server's received timestamp, the server's transmitted timestamp, and the client's receive time (captured locally when the response arrives). Clients feed these into the time filter via its `update` method and use its `compute_client_time` method to convert server timestamps to local clock values for playback scheduling.

A player MUST NOT report `available: true` until its time filter has converged enough to begin scheduling playback. A source MUST NOT report `available: true` until its time filter has converged enough to timestamp captured audio.

### Transmit timestamps

Two things report when the server transmitted a message: the `server_transmitted` field, and the `send_ahead` interval carried in binary audio chunks. The server takes this time as late as its implementation permits, after any application-level queueing or per-client scheduling, immediately before the message is encrypted for transmission. A server MUST NOT stamp a message at the time it is enqueued for later transmission.

Delay accruing after that point - transport send buffering, an earlier fragmented message still in flight, link contention - is not represented in the value and is observed by the client as network delay.

A client measuring transit takes its `arrival` time for the message once the message is available to the application: after AEAD decryption, and after reassembly for a fragmented message. Both ends of the measurement therefore sit at the application boundary.

## Core messages
This section describes the fundamental messages that establish communication between clients and the server. These messages handle initial handshakes, ongoing clock synchronization, stream lifecycle management, and role-based state updates and commands.

Every client and server MUST implement all messages in this section regardless of their specific roles. Role-specific object details are documented in their respective role sections and need to be implemented only if the client supports that role.

[Pairing](pairing.md#pairing) messages are REQUIRED for all servers; clients implement the subset matching their advertised pairing methods.

WebSocket preserves message order within each direction, but messages in opposite directions can cross. Clients MUST process `stream/start`, `stream/clear`, and `stream/end` in delivery order, even when discarding role data. Servers MUST likewise process `client-stream/start` and `client-stream/end` in delivery order.

### Client → Server: `client/init`

First message sent by the client after the WebSocket connection is established. Contains information necessary for conducting the Noise handshake.

- `client_id`: string - client's static public key (43-character base64url-encoded Curve25519, no padding). See [Identities](connection.md#identities). Persistent across reconnections so servers can associate clients with previous connections (e.g., remembering group membership, settings, playback queue)
- `version`: integer (MUST be `1`) - version of the core message format that the client implements (independent of role versions)
- `suite`: '25519_ChaChaPoly_SHA256' | '25519_AESGCM_SHA256' - Noise cipher suite the client picked for this connection. See [Cipher Suites](connection.md#cipher-suites)

`version` (here and in [`server/init`](#server--client-serverinit)) is an exact-match field naming the single core message format the sender speaks, not a minimum-supported version. Under this specification both sides send `1` and abort the handshake on any other value (see [Failure Handling](connection.md#failure-handling)). A future revision that requires a new core version under [Protocol evolution](README.md#protocol-evolution) will use a new value and define how it is negotiated.

### Server → Client: `server/init`

Response to the [`client/init`](#client--server-clientinit) message with corresponding information about the server.

The server sends `server/init` immediately followed by the first [`noise/handshake`](#client--server-noisehandshake) message (Noise message 1) without waiting for any client message in between.

- `server_id`: string - server's static public key (43-character base64url-encoded Curve25519, no padding). See [Identities](connection.md#identities)
- `version`: integer (MUST be `1`) - version of the core message format that the server implements (independent of role versions)

### Client ↔ Server: `noise/handshake`

Carries one Noise handshake message. Sent twice during the handshake: once by the server (Noise message 1, sent immediately after [`server/init`](#server--client-serverinit)), and once by the client in response (Noise message 2).

- `data`: string - base64url-encoded Noise handshake message bytes (no padding)

The encrypted payload carried inside each Noise handshake message is a UTF-8 JSON object:

- **Noise message 1 payload** (server → client): 
  - `psk_id`: string - 43-character base64url-encoded SHA-256 hash derived from the PSK. Used by the client to select the PSK before processing message 2; the message-1 payload is decryptable without the PSK (see [Pre-Shared Key](connection.md#pre-shared-key)).
  - `psk_category`: 'lt' | 'pr' | 'sn' - the category the server is using the referenced PSK as: long-term, pairing, or Sentinel. A `psk_id` the client holds only under a different category is a lookup miss (see [Pre-Shared Key](connection.md#pre-shared-key)). The codes share one length, so the encrypted payload's length is independent of the category.
- **Noise message 2 payload** (client → server): the empty object as the literal two bytes `{}` (not a zero-length Noise payload)

A malformed inner handshake payload (not valid UTF-8 JSON of the shape above, including a `psk_category` outside the three defined codes) is a [silent failure](connection.md#failure-handling) and closes the WebSocket.

After both handshake messages have been exchanged, both sides switch to Noise transport mode (all subsequent messages travel as the binary messages described above).

The same `noise/handshake` message is used for the in-band [re-handshake](connection.md#re-handshake): the two messages then travel as ordinary encrypted JSON messages (binary messages, message type `0`), not bare Noise bytes. Noise message 2 is still encrypted under the pre-re-handshake transport keys; the first binary message each side sends after the handshake completes uses the new keys.

### Server → Client: `server/error`

Sent by the server in place of [`server/init`](#server--client-serverinit) when it cannot accept the client's [`client/init`](#client--server-clientinit). The server closes the connection after sending. See [Failure Handling](connection.md#failure-handling).

- `reason`: string - one of:
  - `unsupported_version` - the client's `version` is not one the server implements
  - `unsupported_suite` - the client's `suite` is not one the server implements
  - `malformed` - `client/init` is not valid JSON of the defined shape

### Server → Client: `server/hello`

First message sent by the server after the initial Noise handshake completes. Sent once per connection as an encrypted message (binary message, message type `0`).

- `name`: string - friendly name of the server
- `languages?`: string[] - non-empty list of [BCP 47](https://www.rfc-editor.org/info/bcp47) language tags in descending operator preference (e.g. `["ca", "es", "en"]`) - a hint about the languages the operator understands, informing any operator-facing output
- `source@v1_support?`: object - required if the server supports the `source@v1` role, absent otherwise ([see server-side source@v1 support object details](roles/source/v1.md#server--client-serverhello-sourcev1-support-object))

### Client → Server: `client/hello`

Sent by the client once it has received [`server/hello`](#server--client-serverhello). Sent once per connection as an encrypted message (binary message, message type `0`). Contains information about the client's capabilities and roles.

Clients that can output audio SHOULD have the role `player`.

- `name`: string - friendly name of the client
- `device_info?`: object - optional information about the device
  - `product_name?`: string - device model/product name
  - `manufacturer?`: string - device manufacturer name
  - `software_version?`: string - software version of the client (not the Sendspin version)
  - `mac_address?`: string - MAC address of the network interface the connection is opened on, in lowercase colon-separated form (e.g., `aa:bb:cc:dd:ee:ff`)
- `supported_roles`: string[] - versioned roles supported by the client (e.g., `player@v1`, `controller@v1`). Defined versioned roles are:
  - `player@v1` - outputs audio
  - `source@v1` - captures audio from a local input and streams it to the server
  - `controller@v1` - controls the current group
  - `metadata@v1` - displays text metadata describing the currently playing audio
  - `artwork@v1` - displays artwork images
  - `visualizer@v1` - visualizes audio
  - `color@v1` - receives colors derived from the current audio
- `player@v1_support?`: object - required if `player@v1` is listed, absent otherwise ([see player@v1 support object details](roles/player/v1.md#client--server-clienthello-playerv1-support-object))
- `source@v1_support?`: object - required if `source@v1` is listed, absent otherwise ([see source@v1 support object details](roles/source/v1.md#client--server-clienthello-sourcev1-support-object))
- `visualizer@v1_support?`: object - required if `visualizer@v1` is listed, absent otherwise ([see visualizer@v1 support object details](roles/visualizer/v1.md#client--server-clienthello-visualizerv1-support-object))
- `supported_pair_methods`: object - pairing methods this client currently offers, keyed by method identifier, each value a [pair-method descriptor](pairing.md#client--server-clienthello-pair-method-descriptor). Every client offers at least the Pairing PSK method, and at most one pairing-code method may be listed (see [Pairing](pairing.md#pairing)).
- `unpaired_access`: object - whether this client currently admits [unpaired access](pairing.md#unpaired-access)
  - `enabled`: boolean

When a role version defines a support object, its key in `client/hello` or `server/hello` is the role-version identifier followed by `_support` (e.g., `player@v1_support`, `player@v2_support`). Application-specific roles or role versions follow the same pattern (e.g., `_myapp_display@v1_support`, `player@_experimental_support`).

If a role version requires a support object, the server MUST NOT activate that version when the object is missing from `client/hello`.

### Server → Client: `server/activate`

Declares the server's current purpose on this connection. Sent as an encrypted message (binary message, message type `0`). MAY be re-sent to change the activity set, active roles, or pairing parameters, subject to the activation and pairing rules.

- `activities`: ('playback' | 'pairing')[] - the set of currently-active purposes on this connection. MAY be empty. Members are unordered and unique.
- `active_roles?`: string[] - versioned roles that are active for this client (e.g., `player@v1`, `controller@v1`). Required on the first `server/activate`; persists across subsequent `server/activate` messages that omit it. MUST be empty on connections not capable of playback (see below). A client treats a first `server/activate` that omits it as carrying an empty `active_roles`.
- `pairing?`: object - parameters of the pairing attempt this activation admits. Required when `'pairing'` is in `activities`; absent otherwise. A client ignores this field when `activities` does not include `'pairing'`.
  - `method`: 'dynamic_pairing_code' | 'pairing_psk' | 'static_pairing_code' - pairing method the server picked, drawn from the client's `supported_pair_methods`.
  - `format?`: 'digits' | 'qr_code' - the dynamic [emission format](pairing.md#dynamic-pairing-code-flow), drawn from the client's `dynamic_pairing_code` descriptor. Required when `method` is `'dynamic_pairing_code'`; absent otherwise. The server selects `qr_code` only when its operator interface can scan a QR code.

The activity sets the server may legitimately declare are constrained by which PSK matched during the [Noise handshake](connection.md#encryption):

| PSK matched | Allowed activity sets |
|---|---|
| [long-term PSK](README.md#definitions) | `[]` or `['playback']` |
| [pairing PSK](README.md#definitions) | `[]`, `['pairing']`, `['playback']`¹, `['playback', 'pairing']`¹ |
| [Sentinel PSK](connection.md#pre-shared-key) | `[]`, `['pairing']`, `['playback']`¹, `['playback', 'pairing']`¹ |

¹ Only when the client has [unpaired access](pairing.md#unpaired-access) enabled.

When `'pairing'` is in `activities`, `pairing.method` MUST be `'pairing_psk'` if and only if the matched PSK is the [pairing PSK](README.md#definitions), and MUST be a method present in the client's [`supported_pair_methods`](#client--server-clienthello).

**Playback-capable connections.** A connection is *playback-capable* when its `activities` extended with `'playback'` are an allowed set for the matched PSK; a connection already declaring `'playback'` is therefore playback-capable exactly when its `activities` are an allowed set. Only a playback-capable connection MAY carry a non-empty `active_roles`, and it MAY do so even when `'playback'` is not currently in `activities`. The client re-evaluates this constraint on every `server/activate` against the persisted `active_roles`: if a later activation changes `activities` so the connection is no longer playback-capable without explicitly sending `active_roles`, the persisted roles are treated as empty rather than the message rejected.

`server/activate` is *admissible* when it satisfies the constraints above. When one is not admissible, the client rejects it, selecting the response by the first rule that applies:

- If the session is [unpaired](README.md#definitions), the client does not have [unpaired access](pairing.md#unpaired-access) enabled, and enabling unpaired access would make the activation admissible - close the connection with [`client/goodbye`](#client--server-clientgoodbye) reason `'pairing_required'`.
- If `activities` is not an allowed set for the matched PSK, or `active_roles` is non-empty on a connection that is not playback-capable - close the connection with [`client/goodbye`](#client--server-clientgoodbye) reason `'unauthorized'`.
- If `'pairing'` is in `activities` with a `pairing.method` the matched PSK disallows or the client does not currently offer, or a `pairing.format` the client does not currently offer - reply with [`pair/abort`](pairing.md#client--server-pairabort) reason `method_not_supported`, leaving the connection open.

**Worked example (`pairing_required`).** A Sentinel-keyed connection to a client with unpaired access disabled receives `activities: ['playback']` and `active_roles: ['player@v1']`. Under a hypothetical `unpaired_access: enabled`, `['playback']` would be an allowed set for the Sentinel PSK and the connection would be playback-capable, so the activation would be admissible: the client closes with `'pairing_required'`.

Servers SHOULD declare the minimal set of activities that reflects the connection's current purpose, and drop an activity as soon as that purpose ends. Admission between competing connections is decided by the highest-ranked declared activity (see [Multiple servers](connection.md#multiple-servers-server-initiated)), so keeping an unused activity declared would degrade multi-server cooperation.

Servers normally activate the client's [preferred](README.md#priority-and-activation) version of each role, but MAY omit a role at their discretion (e.g., based on whether the session is paired, deployment context, or operator policy). Checking `active_roles` is therefore required to determine what the client may actually use on this session.

Before a `server/activate` removes a server-to-client stream role (`player`, `artwork`, `visualizer`, or an application-specific role with such a stream), the server MUST send [`stream/end`](#server--client-streamend) for that role if its stream is active. If the first activation after a [re-handshake](connection.md#re-handshake) will remove such a role, the server MUST send any required `stream/end` before starting the re-handshake.

When applying that activation, the client MUST stop the removed role's remaining output, clear its buffers, and release temporary output effects applied by that role, such as ducking. This applies even if an earlier `stream/end` allowed buffered data to finish playing.

Both requirements apply to explicit removals, implicit removals when the connection is no longer playback-capable, and replacement of an active role version.

When applying a `server/activate`, the client MUST immediately discard the current state and any pending scheduled update for every removed role that defines a [`server/state`](#server--client-serverstate) object (`metadata`, `color`, `controller`, or an application-specific role). This applies to explicit removals, implicit removals when the connection is no longer playback-capable, and replacement of an active role version. No preceding `server/state` is required. State for roles that remain active at the same version is unchanged.

Servers MUST ignore inactive-role objects in `client/state` and `client/command` without closing solely for their presence, since the client may not yet have received the role removal. Client-level fields and objects for active roles are processed normally.

### Client → Server: `client/time`

Sends current internal clock timestamp (in microseconds) to the server.
Once received, the server responds with a [`server/time`](#server--client-servertime) message containing timing information for the [time filter](#clock-synchronization).

- `client_transmitted`: integer - client's internal clock timestamp in microseconds

### Server → Client: `server/time`

Response to the [`client/time`](#client--server-clienttime) message with timestamps for the [time filter](#clock-synchronization).

For synchronization, all timing is relative to the server's monotonic clock. These timestamps have microsecond precision and are not necessarily based on epoch time.

- `client_transmitted`: integer - client's internal clock timestamp received in the `client/time` message
- `server_received`: integer - timestamp that the server received the `client/time` message in microseconds
- `server_transmitted`: integer - timestamp that the server transmitted this message in microseconds

### Client → Server: `client/state`

Client sends state updates to the server. Contains client-level state and role-specific state objects.

Sent once the client is ready to report its operational status (`available`), and whenever any state changes thereafter. A player or source reports `available: true` only after it has established [clock synchronization](#clock-synchronization). When a role that defines a state object becomes active in `active_roles`, the client MUST send an update that includes that role's object. The server MUST NOT send that role's binary data until it has received that object. For a role that defines no state object, the client's initial `client/state` opens its binary data instead. On reactivation, such roles use the latest reported `available` without requiring a new `client/state`.

A client whose `active_roles` are non-empty sends the initial `client/state` even when none of its roles defines a state object.

Every message MUST carry `available` and the full state of each role object it includes. Omitting a role object leaves that role's state unchanged.

- `available`: boolean - whether the client is available to participate in Sendspin playback
  - `true` - client is operational and ready to participate in playback; for a player or source this means its clock is synchronized with the server.
  - `false` - the client is in use by an external system and will not yield to Sendspin on request. See [External Source Handling](#external-source-handling)
- `player?`: object - only if the `player` role is active ([see player state object details](roles/player/v1.md#client--server-clientstate-player-object))
- `source?`: object - only if the `source` role is active ([see source state object details](roles/source/v1.md#client--server-clientstate-source-object))
- `artwork?`: object - only if the `artwork` role is active ([see artwork state object details](roles/artwork/v1.md#client--server-clientstate-artwork-object))
- `visualizer?`: object - only if the `visualizer` role is active ([see visualizer state object details](roles/visualizer/v1.md#client--server-clientstate-visualizer-object))

[Application-specific roles](README.md#application-specific-roles) MAY also include objects in this message (keys starting with `_`).

### External Source Handling

A client can be taken over by a non-Sendspin activity (playing other media, another protocol, an HDMI input, and so on). How it reports this depends on whether it will still yield to Sendspin on request.

Changing local availability does not itself end a server-to-client stream. Clients MUST continue processing `stream/start`, `stream/clear`, and `stream/end` while unavailable, since those messages may have been sent before the server received the availability update.

#### Interruptible activity (client stays available)

If the external activity can be interrupted by Sendspin at any time, the client SHOULD remain `available: true` so the server can take it over.

To stop taking part in its group's playback while performing non-Sendspin activity, a client MAY leave its current group with [`client/leave`](#client--server-clientleave). This is only needed while the group's `playback_state` is `'playing'`: a client in a stopped group keeps its grouping by staying, and later playback may still take it over.

#### Non-interruptible activity (client becomes unavailable)

When a client reports `available: false`, it indicates the client is in use by an external system (e.g., a different audio source, HDMI input, or local media playback) and will not participate in Sendspin playback with this server until it returns to `available: true`.

A client SHOULD report `available: false` only while it will not yield to Sendspin, and SHOULD return to `available: true` as soon as it is again willing to be taken over.

#### Server behavior when a client becomes unavailable (`available: false`):

If the client is in a multi-client group:
1. Remember the client's current group as its "previous group" (see [switch command cycle](roles/controller/v1.md#switch-command-cycle))
2. Move the client to a new solo group (stopped)
   - Send [`group/update`](#server--client-groupupdate) with the new group information
   - Send [`stream/end`](#server--client-streamend) for all active streams, if any

If the client is already in a solo group:
- Stop playback and send [`stream/end`](#server--client-streamend) for all active streams, if any
- If `playback_state` was not already `'stopped'`, send [`group/update`](#server--client-groupupdate) reporting `playback_state: 'stopped'`

When a client returns to `available: true`, the server MUST NOT auto-rejoin it to its previous group or restart playback; the client remains in the solo group and rejoins only via an explicit [`switch`](roles/controller/v1.md#switch-command-cycle).

### Client → Server: `client/command`

Client sends commands to the server. Contains command objects based on the client's active roles.

- `controller?`: object - only if the `controller` role is active ([see controller command object details](roles/controller/v1.md#client--server-clientcommand-controller-object))

[Application-specific roles](README.md#application-specific-roles) MAY also include objects in this message (keys starting with `_`).

### Client → Server: `client/leave`

Leaves the client's current group. No payload fields.

A client sends this when it no longer wants to take part in its group's playback, for example while performing an [interruptible non-Sendspin activity](#interruptible-activity-client-stays-available).

The server handles it as [when the client becomes unavailable](#server-behavior-when-a-client-becomes-unavailable-available-false): the client ends up in a solo group with playback stopped, and rejoins only via an explicit [`switch`](roles/controller/v1.md#switch-command-cycle).

### Server → Client: `server/state`

Server sends state updates to the client. Contains role-specific state objects.

Every message MUST carry the full state of each role object it includes. Omitting a role object leaves that role's state unchanged and any pending scheduled update in place. For the `metadata` and `color` objects, a future `timestamp` defers when the state takes effect (see scheduled updates for [`metadata`](roles/metadata/v1.md#scheduled-metadata-updates) and [`color`](roles/color/v1.md#scheduled-color-updates)).

After a `server/activate` adds or re-adds a role that defines a `server/state` object, the server MUST promptly send a `server/state` containing that role's current state. The server MUST NOT activate such a role until it can provide a complete state object as defined by that role, and MUST deactivate it if it can no longer do so.

The server MUST promptly report changes to active roles' `server/state` objects. Scheduled updates taking effect and playback progress advancing as reported require no new message.

The first `server/state` sent for a role on a connection, and the first after that role is re-added to `active_roles`, MUST carry a past or present `timestamp` if the role object has one, so the client is brought up to date before any scheduled update follows.

- `metadata?`: object - only if the `metadata` role is active ([see metadata state object details](roles/metadata/v1.md#server--client-serverstate-metadata-object))
- `controller?`: object - only if the `controller` role is active ([see controller state object details](roles/controller/v1.md#server--client-serverstate-controller-object))
- `color?`: object - only if the `color` role is active ([see color state object details](roles/color/v1.md#server--client-serverstate-color-object))

[Application-specific roles](README.md#application-specific-roles) MAY also include objects in this message (keys starting with `_`).

### Server → Client: `server/command`

Server sends commands to the client. Contains role-specific command objects.

- `player?`: object - only if the `player` role is active ([see player command object details](roles/player/v1.md#server--client-servercommand-player-object))
- `source?`: object - only if the `source` role is active ([see source command object details](roles/source/v1.md#server--client-servercommand-source-object))

[Application-specific roles](README.md#application-specific-roles) MAY also include objects in this message (keys starting with `_`).

### Server → Client: `stream/start`

Starts a stream for one or more roles. If sent for a role that already has an active stream, updates the stream configuration without ending the stream. Each role defines how data received under the previous configuration is handled.

- `server_transmitted`: integer - timestamp that the server transmitted this message in microseconds
- `player?`: object - only if the `player` role is active ([see player object details](roles/player/v1.md#server--client-streamstart-player-object))
- `artwork?`: object - only if the `artwork` role is active ([see artwork object details](roles/artwork/v1.md#server--client-streamstart-artwork-object))
- `visualizer?`: object - only if the `visualizer` role is active ([see visualizer object details](roles/visualizer/v1.md#server--client-streamstart-visualizer-object))

[Application-specific roles](README.md#application-specific-roles) MAY also include objects in this message (keys starting with `_`).

The server MUST NOT send `stream/start` unless the latest [`client/state`](#client--server-clientstate) it has received reports `available: true`.

Each role's stream configuration is derived from what the client reports about itself: the role's support object in [`client/hello`](#client--server-clienthello) for constant capabilities, and the role's [`client/state`](#client--server-clientstate) object for the stream-configuration fields the client may change during the connection (each role may define both). After a role that defines a `client/state` object is added or re-added to `active_roles`, the server MUST wait for the [`client/state`](#client--server-clientstate) update the activation requires before starting that role's stream, so it does not start from stale state. When a `client/state` changes a role's stream-configuration fields while a stream is active for that role, the server re-derives the stream configuration and, if it changed, sends a `stream/start` with the new configuration. When no stream is active for the role, the server MUST NOT start one in response; the updated state applies to the next stream it starts for that role.

Clients may change their stream-configuration fields to adapt to changing network conditions, CPU constraints, or display requirements. The server maintains separate encoding for each client, allowing heterogeneous device capabilities within the same group.

### Server → Client: `stream/clear`

Clients MUST clear buffers for the specified roles without ending their streams. Used for seek operations and track jumps (switching to a different track without stopping the stream).

The server MUST NOT send this message when no targeted streams are active.

- `server_transmitted`: integer - timestamp that the server transmitted this message in microseconds
- `roles?`: non-empty string[] - roles to clear: '[player](roles/player/v1.md#server--client-streamclear-player)', '[visualizer](roles/visualizer/v1.md#server--client-streamclear-visualizer)', or both. Every listed role MUST have an active stream. If omitted, clears all active player and visualizer streams

[Application-specific roles](README.md#application-specific-roles) MAY also be included in this array (names starting with `_`).

### Server → Client: `stream/end`

Ends the stream for one or more roles. Each side MUST treat the targeted streams as inactive once it sends or receives this message, even if buffered output continues.

For each specified role, clients MUST stop output and clear its buffers unless that role explicitly defines different completion behavior. In that case, clients MUST follow the role's rules, such as finishing playback of buffered data.

For roles following the media queue, this message is expected to be sent when playback is over and the queue is empty. Specifically:

- **Track transitions** (a track ends and the next begins naturally): stream commands SHOULD NOT be sent, except `stream/start` to update the existing stream configuration. The stream continues uninterrupted to support gapless playback and server-inserted crossfade.
- **Seeks** (jumping to a position within the current track): send `stream/clear` instead.
- **Track jumps** (skipping to a different track): treat identically to a seek, sending `stream/clear` instead of `stream/end`. Conceptually, the entire queue is a single continuous stream.

Servers MUST NOT send `stream/end` in these cases because it signals actual playback termination, causing clients to stop output entirely rather than continue playing.

The server MUST NOT send this message when no server-to-client streams are active.

- `roles?`: non-empty string[] - roles to end streams for ('player', 'artwork', 'visualizer'). Every listed role MUST have an active stream. If omitted, ends all active streams

[Application-specific roles](README.md#application-specific-roles) MAY also be included in this array (names starting with `_`).

### Server → Client: `group/update`

State update of the group this client is part of.

The server MUST promptly send this message after the first `server/activate` on a connection and whenever any field listed below changes.

Every message MUST carry all fields listed below.

- `playback_state`: 'playing' | 'stopped' - playback state of the group
- `group_id`: string - group identifier
- `group_name`: string - friendly name of the group

### Server → Client: `server/unpair`

Sent by a paired server to drop its own pairing record from the client. Valid regardless of the current `activities`, subject to the [initial message sequence](#communication) and [re-handshake restrictions](connection.md#re-handshake). No payload fields.

The server also removes its corresponding pairing record.

Client behavior:

- Remove the matched pairing record, send [`client/goodbye`](#client--server-clientgoodbye) reason `'unpaired'`, and close the connection.
- If the session is [unpaired](README.md#definitions), ignore the message and continue unchanged.

### Client → Server: `client/goodbye`

Sent by the client before gracefully closing the connection. This allows the client to inform the server why it is disconnecting.

Upon receiving this message, the server SHOULD initiate the disconnect.

- `reason`: 'another_server' | 'shutdown' | 'restart' | 'user_request' | 'unauthorized' | 'pairing_required' | 'concurrent_attempt' | 'unpaired'
  - `another_server` - client is switching to a different server. A client that leaves one server for another MUST send this reason to the server it is leaving. Server SHOULD NOT auto-reconnect but SHOULD show the client as available for future playback
  - `shutdown` - client is shutting down. When the device is powering off or otherwise not coming back and no more specific reason applies, clients SHOULD send this reason. Server SHOULD NOT auto-reconnect
  - `restart` - client is restarting and will reconnect. Server SHOULD auto-reconnect
  - `user_request` - user explicitly requested to disconnect from this server. Server SHOULD NOT auto-reconnect
  - `unauthorized` - the server declared an activity set or `active_roles` the client is not authorized for (see [`server/activate`](#server--client-serveractivate)). Server SHOULD NOT auto-reconnect with the same activity set
  - `pairing_required` - the client refused, or no longer admits, an [unpaired access](pairing.md#unpaired-access) connection because it does not have unpaired access enabled. Server SHOULD NOT auto-reconnect without pairing first
  - `concurrent_attempt` - the client refused the connection under the [multiple-server admission rules](connection.md#multiple-servers-server-initiated). Server MAY retry later
  - `unpaired` - the client has processed [`server/unpair`](#server--client-serverunpair) from this server. Server SHOULD NOT auto-reconnect

On a client-initiated connection the server cannot reconnect; the reconnect guidance then applies to the client re-establishing the connection.

Connections may be lost without this message (e.g., crash, network loss). Clients MAY deliberately close without sending `client/goodbye` unless this specification requires it. After sending `client/goodbye`, clients MAY close immediately without waiting for the server to disconnect. When a client disconnects without sending `client/goodbye`:

- On a connection whose `activities` are empty, or include `'playback'`, servers SHOULD assume the disconnect reason is `restart` and attempt to auto-reconnect.
- Otherwise, servers SHOULD treat the drop as a session termination and not auto-reconnect; resumption, if desired, is operator-driven.
- Servers SHOULD also apply backoff on repeated Noise-handshake failures to avoid tight reconnect loops.
