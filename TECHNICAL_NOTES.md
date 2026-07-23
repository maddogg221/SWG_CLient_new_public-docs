# Technical Notes

A handful of illustrative notes and small fragments — enough to show real engineering rigor is behind this project, not a full walkthrough of how anything actually works. Deliberately incomplete by design.

## Verification discipline

Nothing gets marked "done" on the strength of it compiling or looking plausible against source code alone. The project's own working rule, applied consistently throughout:

> The wire is the source of truth — not the source code. Reference source shows what *can* exist on the wire; only real, captured traffic confirms what a live server actually sends.

In practice this means every decoder has, wherever possible, a permanent automated test built from **real bytes captured off a live connection** — not hand-invented sample data. The general shape of that testing style:

```cpp
TEST_CASE("<some message type>::parse - real captured payload") {
    auto buffer = bytesFromRealCapture(/* ... */);

    auto decoded = SomeMessageType::parse(buffer);

    CHECK(decoded.someField == /* the real, observed value */);
    CHECK(buffer.remaining() == 0); // no leftover, unexplained bytes
}
```

That last assertion — "no leftover bytes" — is treated as important as the field checks themselves: a decoder that reads the *wrong* number of bytes will usually still produce plausible-looking values, right up until the next message in the stream gets misaligned and everything downstream breaks. Catching that immediately, at the single point of failure, rather than as a confusing symptom three messages later, has repeatedly been worth the extra rigor.

## When source and reality disagree, reality wins

More than once, careful reading of reference source material led to a wrong conclusion that only real captured traffic exposed — including, this cycle, a case where a wire-format detail that looked ambiguous from source alone turned out to be fully resolvable once every layer that touches a message before this project's own code sees it was traced in full, not just the layer that seemed most relevant at first glance. The fix, once found, was confirmed three independent ways before being trusted — a deliberate, repeated pattern in how this project treats any conclusion it can't directly observe on the wire.

## A live cross-check that caught a real bug

The terrain generation work (see Progress) needed one thing no source-reading could provide: a real, independently-known "at this exact map position, the ground is at this exact height" fact to check the computed output against. That came from a live server telling the connecting client its own spawn coordinates directly — a value this project had been receiving and printing for a long time, but had never actually *used* for anything before.

Putting it to use immediately paid off: the raw values didn't line up with the computed terrain height at all, off by orders of magnitude, on a position independently confirmed to be standing on ordinary open ground. Rather than assume the newly-written terrain code was wrong, the reference source for that particular network message was re-examined end to end — and it turned out an internal helper on the *reference* side used two of its own axis labels in a way that didn't match the meaning this project (correctly, per every other position-carrying message already validated by rendering real objects on screen) had assumed. One message type had inherited that mismatch by naming its own fields after the reference helper's labels instead of their actual meaning — a mislabeling that had sat unnoticed because that particular value had only ever been printed to a log, never fed into anything that would have visibly disagreed with reality.

Fixed, then re-verified against two independent live servers: computed height matched real ground height to well under a meter both times, with a deliberately-included "standing inside a building" sample confirmed to *not* match — proof the test itself was discriminating, not just always agreeing.

## A small, honest example of graphics-side rigor

The rendering pipeline's real-geometry shading uses a standard, well-known technique (half-lambert shading) rather than anything exotic — the point was never to reinvent graphics, only to get real objects on screen correctly and quickly:

```hlsl
float shade = 0.5 + 0.5 * saturate(dot(normalize(worldNormal), lightDir));
```

Small, boring, and correct — which is the actual goal at this stage of the project.

## A compressed-format ambiguity that only got resolved by reading the real client's own code

Character animation data is stored using a well-known compression trick for unit quaternions (often called "smallest three"): rather than store all four rotation components, the encoder drops whichever one has the largest magnitude and stores only the other three, quantized. On decode, the dropped component is reconstructed mathematically from the other three via `sqrt(1 - x² - y² - z²)` — a square root, which has two possible signs, only one of which is the rotation the encoder actually intended.

General background research into this technique suggested the decoder needs some extra signal to pick the correct sign — and a first implementation was built on that assumption, disambiguating frame-to-frame using a global optimization pass that picked whichever sign kept a whole animation channel's motion smoothest. It mostly worked, but not always: certain limbs would intermittently snap into a wildly wrong orientation for a frame or two, in a way no amount of tuning the smoothness heuristic fully eliminated.

The actual answer turned out not to be a smarter disambiguation algorithm at all. Reading the relevant decode routine directly in the original game client's own compiled code showed it does no sign check whatsoever — it takes the plain, unconditionally non-negative square root, every time, full stop. The encoder guarantees this works by choosing, at export time, whichever overall sign of the quaternion (`q` or `-q` — both represent the identical rotation) makes the dropped component come out non-negative, so the decoder never has anything to guess. The general "smallest three" technique doesn't inherently require this convention, but this particular content pipeline does — a detail no amount of reasoning about the compression scheme in the abstract was going to surface, and that no amount of statistical tuning against sample data fully compensated for either. Only reading the real, authoritative implementation settled it, and once applied, an entire category of intermittent bad-frame artifacts went away at once rather than needing to be chased individually.

Finding that routine at all, inside an executable with no symbols and no source, is its own small story. The way in wasn't guessing at addresses or scanning for byte patterns first — it was searching the binary for human-readable debug strings the original developers had left baked into it (an internal class name logged during setup, in this case), then reading outward from that one confirmed landmark: what constructs this, what else references its name, what does the code actually do a few calls downstream. Most of what that trail turns up first is boilerplate — constructors, reference counting, object-pool bookkeeping — that has to be recognized and set aside rather than mistaken for the real logic. The actual arithmetic, when it was finally reached, was a handful of floating-point instructions doing exactly the multiply-add-square-root sequence the math predicted, which is its own satisfying confirmation of being in the right place. Slow, methodical, and a genuinely different kind of puzzle from the network-protocol reverse engineering this project otherwise does daily — closer to archaeology than to decoding a spec.

## Finding a live graphics-API address in a binary with no import table to read

A recurring problem when reading a compiled game client directly: modern graphics APIs like Direct3D are accessed entirely through interface objects handed out at runtime — there's no fixed address for "the function that draws a triangle" sitting in the executable waiting to be found by name. The object implementing the graphics device doesn't exist until the game creates one, and every method call on it goes through a table of function pointers whose *layout* is a documented, stable public contract, but whose actual in-memory *location* is different every time the program runs.

The general technique that worked, reusable for any similar API: catch the one moment the graphics device gets created (this only happens once, early, so the debugger has to be attached from process launch rather than to an already-running instance), capture the object handed back at that exact moment, then read its function-pointer table directly out of memory using the *public, documented* method ordering for that interface — no internal knowledge of this specific game required, just the same publicly available interface contract any graphics programmer would already have. From there, the address of any method on that interface — draw calls, state changes, resource creation, anything — is a single, deterministic memory read away, and stays valid for the rest of that run.

This turned a category of "there's no symbol for this, so it's effectively unreachable" problems into "there's no symbol for this, but the address is still fully computable" — a meaningfully different, much more tractable situation.

## Ruling things out is still real progress

A specific, real question came up while investigating how animated characters actually get drawn: does this client compute skinned character poses on the CPU (composing each bone's position and writing final vertex data before anything reaches the graphics card), or does it hand raw pose data to the graphics card and let it do that math on every vertex during rendering? The two approaches lead to genuinely different places to look for how a character's final shape gets built each frame, so getting this right mattered before going further.

The direct way to test it: watch every place the client uploads per-vertex transform data to the graphics card during a real play session, and see what's actually being uploaded. Doing this live turned up a real, if slightly deflating, pattern at first — three different, independently traced call sites, each fully followed back to its origin, and each turned out to be something else entirely: one was ordinary camera/viewport setup, one was a level-of-detail selection routine picking among nearby quality tiers based on view distance, and one was a generic per-object "this became newly visible, refresh its pending state" mechanism that applies uniformly to every kind of object the engine renders — buildings and trees included, nothing specific to characters at all.

None of those three were the answer, but ruling each one out *was* the answer forming: a real, named class in the client's own code — one whose very name describes exactly this ("software" being the standard industry term for CPU-side computation, contrasted with the graphics card doing the equivalent work) — turned out to be responsible for skinned characters specifically, and tracing one of its real operations through to completion led all the way to an actual, ordinary "draw these triangles" graphics-card call, using geometry that had already been fully assembled by the time it got there. That's about as direct a confirmation as reverse engineering this kind of thing gets: not an inference from absence, but watching the real handoff happen.

The methodological point worth keeping, independent of the specific answer: a wrong-looking result that's been fully, honestly traced to its actual cause is worth more than a plausible-sounding guess left unverified, even when — especially when — the traced result isn't the one being hoped for going in.

## An experimental fix that a live-code read proved unnecessary

A real, visible animation artifact on articulated character geometry — a flat, collapsing seam at certain joints — led to trying a well-known alternative blending technique specifically designed to eliminate that class of artifact, on the theory that the more common, simpler blending approach was the inherent cause. It was a reasonable theory: the artifact matched a textbook description of that simpler approach's known failure mode closely enough to justify the experiment.

Reading the real client's own compiled code settled it differently. The official implementation uses the plain, simpler blending approach — not the more sophisticated alternative the experiment had switched to. If the simpler approach's known failure mode were really the cause, the official client would show the same artifact under the same conditions, and it doesn't. That's a clean, falsifiable test: the theory predicted something checkable, and reading the real implementation checked it directly rather than continuing to guess. The experimental change was reverted in favor of matching the confirmed real technique, and the search for the actual cause narrowed to something else — not the blending family, but somewhere in the underlying per-joint data feeding it.

The broader point: an experimental fix that make a symptom look different isn't the same as a fix that addresses the real cause, and the fastest way to tell the two apart, when the ground truth is available at all, is to go read it directly rather than keep iterating on the symptom.

## Knowing when to stop chasing a root cause and fix the data instead

A follow-up investigation into that same joint artifact went looking for ground truth more directly — tracing how the official client actually manages the memory it writes finished per-vertex data into, frame by frame. That trace turned up a lot of real, working detail: a resource-pool allocator identified by name via a debug string baked into the binary, the specific method responsible for handing back a writable buffer, and a ring/bump-allocator pattern managing where in a shared arena each frame's data actually lands.

What it didn't turn up, after a long session, was the literal values being written — the actual numbers needed to compare directly against this project's own computed output for the same joints. Getting there would have meant reliably capturing several different pieces of fast-changing, live process state at the exact same instant, by hand, through a slow manual read-and-record workflow — and two separate attempts at that hit the same wall: values read a few steps apart in time turned out to be already inconsistent with each other by the time all of them were in hand.

Faced with that, the practical call was to stop pushing on the reverse-engineering thread for the moment and go fix the actual defect a different way: directly reworking the affected geometry's own per-vertex weighting using existing, community-built asset tooling, rather than continuing to chase the official client's exact internal arithmetic. This doesn't require ever learning the precise root cause — it repairs the specific broken data directly, and can be checked against the same live rendering used throughout. If a cleaner way to capture multiple pieces of live state atomically turns up later (scripted, rather than manual, capture is the obvious next tool for that), the deeper question is still worth returning to — but shipping a real fix didn't need to wait on it.

## What's deliberately not shown here

- The actual message/field layouts this project has decoded (that's the protocol reverse-engineering work itself, and the whole point of keeping the implementation private).
- The internal module structure and how the pieces connect.
- Real server details, credentials, or any information about the infrastructure this project is tested against.
