# Scope

This file defines the Scope of the Sendspin Working Group for the purposes of the [Community Specification License 1.0](LICENSE.md). The Scope bounds each contributor's and licensee's patent commitment. Changes to Scope are not retroactive.

## In scope

The Sendspin protocol specification: a protocol for orchestrating the devices that make up a multi-room music listening experience over a local IP network, as published in this repository. The Scope covers everything an implementer needs to build a conforming Sendspin client or server, including:

- discovery and connection establishment (mDNS service types, WebSocket transport, server- and client-initiated connections, multi-server arbitration);
- the encryption layer (Noise handshake pattern, cipher suites, identities, pre-shared keys, sentinel fallback, re-handshake) and the pairing methods with their message flows, code derivation, pairing tokens, and PAKE instantiation;
- message framing, binary message identifiers, fragmentation, and the core, management, and pairing messages;
- clock synchronization between clients and servers, including the time-filter method the specification requires, and the timestamp semantics used to schedule audio output, images, colours, visualization data, and metadata;
- the role definitions (player, source, controller, metadata, artwork, visualizer, color) with their messages, state objects, binary payloads, and behavioural requirements, including playback synchronization, correction, group volume, and external-source handling;
- any future roles, role versions, or messages adopted into this repository.

## Out of scope

- The audio codecs and image formats the specification references (Opus, FLAC, PCM, JPEG, PNG). The specification names them; it does not define them.
- Server-side content sourcing, playback queues, music analysis (beat, loudness, spectrum extraction), colour extraction from artwork, and lighting effect generation. The specification defines how their results are carried, not how they are produced.
- User interfaces of clients, servers, and controllers.
- Software that implements the specification. Reference implementations and SDKs carry their own licenses in their own repositories.
