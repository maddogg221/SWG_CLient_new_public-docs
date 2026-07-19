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

## A small, honest example of graphics-side rigor

The rendering pipeline's real-geometry shading uses a standard, well-known technique (half-lambert shading) rather than anything exotic — the point was never to reinvent graphics, only to get real objects on screen correctly and quickly:

```hlsl
float shade = 0.5 + 0.5 * saturate(dot(normalize(worldNormal), lightDir));
```

Small, boring, and correct — which is the actual goal at this stage of the project.

## What's deliberately not shown here

- The actual message/field layouts this project has decoded (that's the protocol reverse-engineering work itself, and the whole point of keeping the implementation private).
- The internal module structure and how the pieces connect.
- Real server details, credentials, or any information about the infrastructure this project is tested against.
