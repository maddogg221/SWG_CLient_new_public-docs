# SWG Client New — Public Progress Docs

A modern, from-scratch client for [SWGEmu](https://www.swgemu.com/) / [Core3](https://github.com/swgemu/Core3) — reimplementing Star Wars Galaxies' network protocol and, as of this update, a real rendering pipeline, from first principles, verified continuously against a live server.

**This is a public status/progress mirror, not the source code.** The private repository containing the actual implementation is not public. This repo exists so people following the project can see real, verifiable progress — milestones and a few illustrative snippets — without exposing the full implementation.

## Project Goal

Star Wars Galaxies' original client is closed-source and effectively unmaintainable for new development. Core3, the open-source SWGEmu server emulator, is GPL-licensed — its source is an invaluable protocol *reference*, but not something this project copies from. **SWG Client New** is an independent, clean-room client implementation: every wire format, encryption scheme, and message layout was derived by reading Core3's source for the *facts* of the protocol, then implemented fresh in original code — the same legitimate interoperability approach any protocol-compatible client is built on.

The long-term goal is a complete, modern client capable of real gameplay: movement, combat, crafting, and everything else. Progress so far has been deliberately sequenced — protocol correctness first, then real rendering, now moving toward the interaction systems needed for actual gameplay.

## Current Status

**Working today:**
- A complete, from-scratch reimplementation of SWG's transport-layer protocol (session handshake, encryption, integrity checking, compression, and reliable packet delivery) — verified continuously against a live server, not a mock.
- The full login → character-select → zone-in sequence against a real Core3 server.
- A schema-driven decoder covering the majority of SWG's object types and their real-time incremental update traffic.
- Real-time movement and position tracking for every nearby object.
- A real rendering pipeline (**Vulkan**), decided on and built after direct hands-on rendering experience with an earlier prototype backend. Live-verified: a real 3D view showing tracked objects, a ground reference grid, a minimap, and floating name labels.
- **Real game asset loading**, resolving actual objects from the live server into their real 3D models pulled directly from the game's own client data files — not placeholder shapes. This closed a long-standing internal mystery in how the game identifies which model belongs to which object; solving it made real content start rendering for the first time.
- The game's right-click radial-menu interaction protocol, solved and then live-confirmed against real captured traffic — a prerequisite for meaningfully decoding crafting and combat traffic, both of which are next.
- **Real procedural terrain**, generated and rendered — the game's actual terrain isn't a stored heightmap but a rule-driven generation system plus a real coherent-noise function; this project's own from-scratch implementation of it is complete, cross-checked against real ground height on two independent live servers (matching to well under a meter), and now actually rendered as a real 3D surface in the live view rather than a flat placeholder.
- **Player-driven movement** — a human can now walk the character through the live world with real keyboard input, height-clamped to the real rendered terrain, with real movement reported back to the server over the network. Open-world walking only so far.
- A large and growing automated test suite, a substantial fraction of it built on real byte fixtures captured from a live server, not synthetic data alone.

**In progress / deliberately deferred:**
- Player-placed structures (houses and similar buildings) don't yet render as their real models — found during this pass's live terrain testing, cause precisely identified, fix not yet written.
- Movement is open-world walking only — no interiors, running, or other locomotion states yet.
- Crafting and combat protocol decode — both are real, substantial undertakings, largely undecoded so far, gated behind the terrain and radial-menu work mentioned above.

## Architecture, at a glance

Deliberately described here at a high level only:

- A transport layer handling the game's real network protocol independently of any game-specific logic.
- An application protocol layer that decodes login, character, zone, and real-time object-state traffic through a declarative, compile-time-checked schema system — not a hand-rolled parser per message type.
- An in-memory world-state model that real decoded traffic feeds into, and that the rendering layer reads from.
- A real-asset pipeline that resolves the game's own archive files into renderable geometry.
- A from-scratch procedural terrain generation layer, independent of the rendering pipeline and directly verifiable on its own (see Progress).
- A Vulkan-based rendering layer built on top of all of the above.
- A set of test/verification tools used to exercise every piece of this against a real server.

## Design Principles

- **Clean-room, not copy-paste.** Core3's source is read for protocol *facts* and reimplemented independently; its (GPL) code is never copied into this project.
- **The wire is the source of truth — not the source code.** Core3's source shows what *can* exist on the wire; only real, captured traffic confirms what a live server actually sends.
- **Detect and stop, don't guess.** Unknown or ambiguous wire data produces a clear, logged failure — never a silent misdecode.
- **Verify against real traffic, always.** No decoder or rendering feature ships without being exercised against a genuine live server.

## A note on transparency

This project maintains an unusually thorough internal paper trail — every protocol fact, every bug found and fixed, every design decision, all logged as it happens. That level of detail isn't reproduced here, since it would double as an implementation guide. What you'll find in this repo instead is an honest, accurate account of *what's been achieved and when* — no exaggeration, no vaporware. If something is described here as working and live-verified, it is.

## Documentation

| File | What it is |
|---|---|
| [`PROGRESS.md`](PROGRESS.md) | A milestone-by-milestone timeline of what's been built, in the order it happened. |
| [`TECHNICAL_NOTES.md`](TECHNICAL_NOTES.md) | A handful of illustrative technical notes and small code snippets — enough to show the work is real, not enough to reconstruct the implementation. |

## License

Not yet finalized. All code in the private implementation is original, independently written from protocol facts (not copied from Core3 or any other GPL-licensed source).
