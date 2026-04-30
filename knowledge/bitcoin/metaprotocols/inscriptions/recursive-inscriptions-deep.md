# Recursive Inscriptions - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/metaprotocols/inscriptions`.
> Canonical source: https://docs.ordinals.com/inscriptions/recursion.html
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/metaprotocols/inscriptions/SKILL.md

## Concept

Recursive inscriptions let one inscription reference the body of
another at render time. The ord client exposes an HTTP endpoint
`/r/inscription/<id>` and `/content/<id>` that JS, HTML, or SVG
inscriptions can fetch. The mechanism turns Bitcoin into a content-
addressed asset bus: an HTML inscription becomes 200 bytes that pulls
in 5 MB of pre-inscribed JavaScript libraries, fonts, or 3D models.
Cost falls dramatically and large libraries are amortised across
thousands of dependent inscriptions.

## Walkthrough / mechanics

Two ways to recurse:

1. **HTTP fetch from inscribed JS/HTML**. The body is normal text/
   html and uses `fetch("/content/<id>")` or `<script src=...>`. The
   ord wallet renderer resolves these against its local index.
2. **Delegate field**. Field 6 (`delegate`) replaces the body with
   the body of the referenced inscription. The delegating inscription
   has empty body and inherits content_type from the delegate. Useful
   for collections where 10_000 NFTs share one image.

Inscription IDs are `<reveal_txid>i<index>`, e.g.
`6fb976ab49dcec017f1e201e84395983204ae1a7c2abf7ced0a85d692e442799i0`.
The index is the position of the envelope inside the witness (0 for
the first envelope of the first input).

ord exposes additional endpoints:

```
/content/<id>           raw body bytes
/r/inscription/<id>     JSON metadata
/r/sat/<sat>            metadata for the sat owning the inscription
/r/blockheight          current chain tip (used by clock inscriptions)
/r/blockhash/<height>   hash of a specific block
/r/children/<id>        list of inscriptions claiming <id> as parent
/r/metadata/<id>        CBOR metadata field
```

## Worked example  (encoded data, real txs, configs)

A minimal recursive HTML inscription that loads p5.js once and runs
sketches:

```html
<!doctype html>
<html><body>
<script src="/content/2dbdf9ebbec6be793fd16ae9b797c7cf968ab2427166aaf390b90b71778266abi0"></script>
<script>
new p5((p) => {
  p.setup = () => p.createCanvas(400, 400);
  p.draw  = () => p.ellipse(p.mouseX, p.mouseY, 50, 50);
});
</script>
</body></html>
```

The HTML is around 230 bytes; the p5.js inscription it references is
roughly 760 KB but exists once. A collection of 1_000 generative-art
inscriptions each pays for 230 bytes of HTML rather than 1_000 *
760 KB of bundled JS. At 30 sat/vB that is 1_725 sats per piece
instead of ~5.7 M sats per piece.

A delegate-style example. Suppose you inscribe a master image at id
`MASTER`. A 10_000-piece collection then mints 10_000 inscriptions
each with envelope:

```
OP_FALSE OP_IF
  "ord"
  PUSH tag=6 (delegate)
  PUSH <MASTER bytes>
OP_ENDIF
```

No body push at all. ord renders each child inscription with the
master's bytes and content_type. The collection cost is 10_000 small
envelopes (~80 bytes each) plus one master, instead of 10_000 copies.

## Trade-offs / pitfalls

- Renderers other than ord (third-party explorers) must implement
  `/r/...` semantics consistently or recursive content breaks. The
  spec is informal; mainstream explorers have caught up but edge
  endpoints (e.g. `/r/sat/...`) sometimes diverge.
- Delegate cycles (A delegates to B, B delegates to A) must be
  rejected. ord caps delegation depth.
- Cursed inscriptions: pre-jubilee inscriptions with negative IDs
  may be referenced; ensure your renderer handles them.
- An inscription that fetches `/r/blockheight` becomes a "live"
  artifact whose appearance changes over time. Marketplaces relying
  on cached previews see stale visuals.
- Censorship resistance: recursive content lets a small malicious
  payload bring in arbitrary other inscribed bytes. Filtering at the
  Knots layer is harder because the witness only contains a hash-
  like reference.
- Browser-side: `/r/...` URLs are served by the local ord node. If a
  user opens raw HTML outside ord, recursion fails. Always render
  inside an ord-aware viewer.

## References

- Recursion docs: https://docs.ordinals.com/inscriptions/recursion.html
- Delegate field: https://docs.ordinals.com/inscriptions/delegate.html
- ord endpoints: https://github.com/ordinals/ord/blob/master/docs/src/inscriptions/recursion.md
