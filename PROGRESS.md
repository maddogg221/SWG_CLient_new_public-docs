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

## What's next

- **Terrain and world rendering.** Everything rendered so far sits on a flat reference grid, not the game's real terrain. This is recognized as a prerequisite for crafting, combat, and travel to mean anything visually, and is a substantial undertaking in its own right.
- **The interaction-menu protocol, live-confirmed.** The mystery above is solved on paper; getting a real live capture to confirm it against actual traffic is the next concrete step.
- **Crafting and combat protocol decode.** Both are large, mostly-unexplored surfaces, intentionally deferred until the prerequisites above are in place.
