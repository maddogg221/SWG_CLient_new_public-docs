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

A recent, concrete example: the visualizer's live debug tooling — dozens of individually-toggleable hypothesis switches, an on-screen diagnostic capture pipeline, and a set of automated A/B test harnesses, all built up over a long animation-debugging effort — had grown to dominate the file it lived in, to the point that the file itself was flagged as a real sustainability risk rather than something to keep adding to. It was split into focused, single-purpose modules along the seams that debugging effort had already revealed, with zero change in behavior — verified with a full rebuild, the existing automated test suite, and a live run against a real server before and after. The split also surfaced a real, previously-unnoticed bug: one of the newer automated test harnesses had its own exit condition backwards, meaning ordinary interactive use (without that harness enabled) would have exited immediately — invisible until the refactor forced a fresh look at exactly that code path, and only caught because "did it actually still run" was checked directly rather than assumed from a successful compile.

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

## Milestone: real character models, in a static pose

The next real gap this project had never touched: characters and creatures. Every player and creature had rendered as a placeholder shape until now — this closed that. Real 3D models are now resolved and rendered for both the player's own character and nearby creatures, parsed directly from the game's own character-model archive files (a distinct, previously-unexplored format from the general object-model pipeline above — skeleton, skinned mesh, and multi-part appearance data, each researched and parsed from scratch). Deliberately scoped to a static pose for this pass — real animation playback is separate, future work.

Getting from "parses correctly against a handful of hand-picked real files" to "actually renders live against a real server" surfaced three genuine bugs along the way, each found and fixed against real data rather than guessed.

## Milestone: real building rendering, exterior and interior alike

Player-placed structures — houses, guild halls, and similar buildings — now render as their real, full geometry, not a placeholder shape. This turned out to need a format the project had never encountered before, entirely separate from the one ordinary objects and characters use: a real building is described as a set of interior rooms, each with its own real 3D model, connected to each other and to the outside by doorways. Untangling that from the game's own archive files — and confirming, against real captured data rather than assumption, which piece represents a building's outer shell versus its inside — was this milestone's real work.

Verified live against an actual player-owned house: a genuine three-story structure with a working elevator, all eighteen real rooms (exterior included) resolving and rendering correctly.

## Milestone: real skeletal animation

Characters had rendered in a single static pose since the milestone above — this closed that gap. Real keyframe animation now plays back for both the player's own character and nearby creatures, driven by the game's own real animation-state and clip data, resolved fresh for this project rather than assumed from documentation.

Getting from "the pipeline runs without crashing" to "the result actually looks like a person" surfaced a genuinely stubborn defect: a mesh-tearing artifact at the wrists and fingers that survived many separate fix attempts across a long stretch of this project's own history, each one validated as thoroughly as the last one's failure suggested was needed, each one still wrong once actually tested live. The eventual real cause, and the lesson learned isolating it, is written up in the technical notes. A related, smaller defect (fingers curling the wrong direction) was found and fixed immediately afterward using the same discipline.

Live-verified against a real server connection: a full character body — torso, arms, legs, hands, and fingers — now renders as a single connected, correctly-proportioned, animated shape.

## Milestone: reading the platform's own original production code directly

Genuine leaked source material for this platform's actual client — not editor tooling around it, but the literal compiled logic the original game uses to compute a character's pose — became available, and was used to check several standing assumptions against ground truth rather than continued inference.

The formula this project already uses to combine a bone's rest-orientation corrections with its animated rotation turned out to be character-for-character identical to the real one — a strong, independent confirmation of math that had already survived a lot of scrutiny, though on its own it didn't explain the resting-pose difference, since that mismatch was already known to live elsewhere. The correction values themselves were confirmed as genuine, deliberate rig data from the original 3D animation tool this platform's artists used, closing off any lingering doubt about whether they might be a decode artifact safe to discard. A branching content-selection structure decoded earlier turned out to have one more real layer than previously modeled — what had looked like a simple mood lookup is actually a set of many small ambient variations (idly shifting weight, checking a device, and similar) — a richer piece of the original design, once it wasn't necessary to explain the specific defect being chased. A suspected secondary motion-blending layer was checked directly and ruled out: it only activates for a specific, deliberate gesture, correctly staying dormant otherwise.

One genuinely new, precisely measured fact came out of this pass: pulling every single recorded keyframe across the character's real idle motion and checking the total range of motion for the affected limb found it tops out at a few tens of degrees at most — nowhere near the roughly quarter-turn a shoulder would need to travel from this project's own computed rest configuration to a natural at-the-sides stance. Every mechanism checked this pass is confirmed either correct or not the cause, leaving a real, precisely-quantified fact with no identified explanation yet.

## Milestone: exhausting the skeleton file format — and a real decode bug found and fixed along the way

Fresh, deliberate side/back/front comparison screenshots against the original game confirmed something worth stating plainly: yes, the original's "resting" stance is itself a real animation loop with its own ambient gestures — but regardless of which gesture is playing, the arm and leg placement is consistently different from this project's own, ruling out "wrong moment to compare" as an explanation.

Went back to the raw skeleton file format and, this time, read the platform's own original file-loading code directly rather than continuing from a partially reverse-engineered understanding. The result was exhaustive rather than incremental: every single piece of data that file format can contain is already read and used by this project — with the one exception being a value the platform's own original code discards immediately after reading it, matching this project's own choice not to use it either. There is nothing left unread in this specific file format.

Reading that same original code raised an adjacent question worth chasing: what does a bone that a given clip doesn't explicitly animate actually do? The answer turned up a real, previously-wrong assumption in this project — such a bone isn't simply "unchanged" (the assumption this project had quietly made); the format stores an actual, fixed, deliberately-authored resting value for it specifically, in a part of the file this project had never read at all. Implemented, decoded, and wired that piece of data in; verified it against real recorded values from real content files; added new automated test coverage confirming it round-trips correctly; tested it live end to end against a running server. It works exactly as expected — a genuine, previously-missing piece of real data, now correctly restored. **It did not fix the resting-pose difference.** The values it supplies are, like the animated motion measured in the previous milestone, only a few tens of degrees at most — real, correct, and still not the missing quarter-turn this defect needs.

## What's next

- **The resting-pose defect — an open call for help.** Every per-clip animation data source both relevant file formats can contain has now been read, checked directly against the platform's own original production code, and — where a real gap existed — fixed. None of it, alone or combined, explains why this project's character stands with its arms held away from its body instead of resting at its sides. What's left points outside per-clip animation data entirely: whether this project's own understanding of a character's base rest configuration (independent of any specific clip) is itself subtly wrong, whether the real game applies some further override this investigation hasn't yet located, or a structural difference not yet identified. This is, by a clear margin, the most thoroughly investigated open defect in this project's history, and the internally-visible leads are exhausted. If anyone with real hands-on knowledge of this platform's animation pipeline, its rest-pose conventions, or its original content-authoring tools recognizes this specific symptom, that's exactly the kind of outside knowledge this project doesn't have another way to reach — genuinely welcome, and would be credited. Walking's own separate, previously-open stride-length question remains closed out clean (see the earlier milestone) — the recorded animation data, played back and combined correctly, appears to genuinely contain that range of motion, very likely not a defect in this project's own reimplementation.
- **Crafting.** A closer look at what crafting actually requires under the hood turned up a genuine surprise: an earlier assumption about what was blocking it turned out to be wrong, and the real path there is shorter than expected. Still real work ahead — a dedicated interaction system, and the ability to carry and hand over items — but not the large undertaking it was thought to be.
- **Combat protocol decode.** Still a large, mostly-unexplored surface, intentionally deferred until the above is in place.
