# Progress Timeline

A milestone-by-milestone account of what's actually been built and verified, in roughly the order it happened. Internal implementation detail, exact dates, and infrastructure specifics are intentionally omitted — this is a record of *what works*, not *how*.

## Foundation: the network transport layer

Before anything game-specific could happen, the underlying network transport had to be reimplemented from scratch: the session handshake, the encryption scheme, integrity checking, compression, and the reliable-delivery mechanics that let large messages survive being split across multiple network packets and reassembled correctly. This layer has no game logic in it at all — it's pure "how do bytes get from the server to us intact." Verified against a live server from the start, not built against a mock and hoped-into-place afterward.

## Milestone: full login-to-zone-in sequence

The complete real sequence a client goes through — authenticating, retrieving the character list, selecting a galaxy and character, and entering the world — implemented and working end-to-end against a real, live Core3 server. This became the standing verification harness for everything built afterward: every subsequent piece of work was exercised against this same real connection, not synthetic test data alone.

## Milestone: world-state decoding

SWG synchronizes the entire visible world to the client via two kinds of messages: full object snapshots and incremental updates. A general-purpose, declarative decoding system was built for this — new object types get *described*, not hand-parsed, and an incorrectly-described field fails to build rather than silently misreading data at runtime. This system now covers the large majority of SWG's real object types.

A companion piece decodes the constant stream of real-time object-controller events (movement, animations, combat feedback, emotes, commands, and dozens of other sub-types) that ride alongside the baseline/delta traffic.

## Milestone: a persistent world model

Decoded network traffic started feeding a real, in-memory representation of the world — tracked objects, their positions, their state — rather than just being printed to a console and discarded. This became the foundation everything visual is built on top of.

## Milestone: first rendering — proving the pipeline

A deliberately minimal first rendering pass proved the whole chain end-to-end: real network traffic → decoded world state → something visible on screen. Wireframe boxes standing in for real objects, a basic camera, and real-time updates as the world state changed. Explicitly a proof-of-pipeline, not a production renderer — but the first time this project produced any visual output at all.

## Milestone: real game assets

A pipeline was built to read the actual game client's own data files and extract real 3D models and geometry from them — not guessed or reverse-engineered from screenshots, but parsed directly from the same archive format the original game client itself reads.

## Milestone: choosing and building the production renderer

With real rendering experience in hand from the earlier proof-of-pipeline work, a deliberate decision was made on the production rendering technology, and the renderer was rebuilt on top of it — the same visual capabilities as before, now on a foundation meant to last. Verified live, piece by piece, against a real server connection throughout.

## Milestone: real objects, rendered as real objects

The final piece connecting the asset pipeline to live gameplay: resolving a real object seen on the network into the correct real 3D model and rendering it as that model, not a placeholder shape. This had been an open problem for a while — the underlying identification mechanism the game uses to connect a live object to its model turned out to have a subtle, non-obvious wrinkle that took real investigative work to track down. Once resolved, real in-game content — structures, everyday objects — started appearing as their actual models for the first time.

## Ongoing: code health

As the codebase has grown, deliberate time has been spent keeping it navigable — splitting overgrown files along real seams, not just adding more to the pile. Unglamorous but treated as a real priority, not an afterthought.

## Milestone: solving a long-standing protocol mystery

SWG's right-click interaction menu — the mechanism behind almost every "use," "examine," or context-specific action in the game — had a real, unresolved wire-format ambiguity blocking its decoding. This was tracked down and fully resolved through careful, methodical source analysis, not guesswork, and cross-checked against multiple independent pieces of evidence before being trusted. This one matters beyond itself: it's understood to be a likely prerequisite for meaningfully decoding both the crafting and combat systems, neither of which has been touched yet.

## Milestone: the interaction-menu protocol, live-confirmed

The right-click interaction-menu mystery from the previous milestone was solved on paper first, then confirmed against a real, deliberately-captured live interaction — not just trusted because the reasoning looked sound. Closing this out properly (not just "probably right") mattered because it unblocks meaningful work on both crafting and combat.

## Milestone: real procedural terrain — research, parsing, and generation, live-verified twice over

The game's real terrain isn't a stored heightmap — it's *generated* at runtime from a fairly intricate rule system (think: layered regions, boundaries, filters, and effects, each narrowing or blending into the next) plus a real coherent-noise function, all driven by data extracted from the game's own files. Getting this right meant, in order: a dedicated research pass into the real format (much of it recovered from historical source history, not just a current snapshot); a from-scratch parser for the on-disk file structure; an independent, from-scratch port of the actual noise/generation math; and finally the full rule-system walk that turns "a point on the map" into "a real height."

Every stage was checked against real data before moving to the next — including a deliberate step of scanning every real map file this project has access to and tabulating which format variations actually appear in practice, so effort went toward what's real rather than every theoretical possibility the format historically supported.

The final piece — does the generated height actually match the real game world? — needed something no amount of source-reading could provide: real, live ground-truth. Two independent real servers were used for this specifically so a match on one couldn't be dismissed as a fluke of that particular server's configuration. Both matched the computed height closely (well under a meter of difference on open ground), and a deliberately-included mismatched case (a position known to be inside a building rather than standing on terrain) confirmed the test itself was meaningful rather than trivially passing. See the Technical Notes for what this cross-check also turned up.

Visual rendering of this terrain (turning generated height/color data into an actual 3D surface on screen) is the next concrete step — the generation logic itself is done and verified; hooking it into the rendering pipeline isn't yet.

## Milestone: terrain rendering — a real 3D surface, live-verified

The generation logic from the previous milestone is now actually hooked into the rendering pipeline: real, generated terrain renders as an actual 3D surface in the live view, streamed in and out around the viewer as they move rather than loaded all at once. A character now visibly stands on real ground instead of a flat placeholder plane.

Getting there surfaced a wider version of the axis-mislabeling issue described in the Technical Notes: the original fix covered one message type, but the same mistaken assumption turned out to have been silently copied into several sibling message types that carry position data — every one of them traced down and corrected the same way, not just the one that happened to get caught first. Worth calling out because of *how* it was caught again: not from re-reading source more carefully, but because the newly-rendered real terrain gave, for the first time, an independent way to visually notice that a tracked position didn't match the ground underneath it. A live-only class of bug, invisible from source alone.

## Milestone: player-driven movement

A human can now actually walk the character through the live world, rather than only spectating decoded world state. Keyboard input drives real movement, height-clamped to the newly-rendered real terrain, and — for the first time — this project sends something back to the server: a real outbound network message reporting the character's own movement, built the same careful, verify-against-the-real-protocol way as every decoder before it. Live-verified end to end against a real server connection.

Deliberately scoped to open-world walking for this first pass; movement inside building interiors, running, and other locomotion states are follow-up work, not yet built.

## Milestone: a real multi-threaded foundation, not just a working prototype

Before adding more visible features, a deliberate pause to get something less glamorous right: how the pieces actually run. Networking, background loading (real object models and terrain streaming in as the world unfolds), and the visible frame being drawn each moment now genuinely run independently of each other, rather than all sharing one sequential loop. Loading real content used to occasionally cause a visible stutter — a burst of nearby terrain generating all at once, for instance — because that work briefly held up the frame being drawn. That's fixed: loading now happens in the background with a real budget on how much gets absorbed per frame, and a noticeable stutter at the start of a session measurably improved as a result.

The instinct going in was to keep building outward — more of the world, more to look at. The choice made instead: treat this as foundational work worth doing deliberately, not as a reaction to a crisis. A steady, unglamorous stretch like this is exactly the kind of work that's easy to skip and expensive to skip *twice*.

## What's next

- **World objects.** Most real in-game objects already render correctly (see "real objects, rendered as real objects" above), but player-placed structures specifically — houses and similar buildings — were found not to, during this pass's live terrain testing. The cause has been precisely identified; the fix hasn't been written yet.
- **Crafting.** A closer look at what crafting actually requires under the hood turned up a genuine surprise: an earlier assumption about what was blocking it turned out to be wrong, and the real path there is shorter than expected. Still real work ahead — a dedicated interaction system, and the ability to carry and hand over items — but not the large undertaking it was thought to be.
- **Combat protocol decode.** Still a large, mostly-unexplored surface, intentionally deferred until the above is in place.
