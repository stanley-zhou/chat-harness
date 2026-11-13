<!-- adr template version: "adr 3.12.0" -->
<!--
adr teaching map: the example is not the teaching — these are the up-links from this file to what teaches its form; load them rather than winging it. (Sibling: the template-version marker dates the mold this file was minted from.)
  skill: adr-authoring
  mcp: adrs4ai
  plugin: adr@adrs4ai
-->

# ADR-0001: OpenRouter key custody in a shared origin

- **Date**: 2026-11-12
- **Iteration**: 1
- **Status**: Draft
- **Deciders**: Yuxiang Zhou, Heddle 5.5 (Claude Opus 5.5)

**TL;DR**: Where the page keeps the OpenRouter key decides who else can read it, and on `stanley-zhou.github.io` "who else" includes every other project site you publish there.

---

## Originating Context

**Source**: `Human dump`, the founding message, 2026-11-12:

> I want to build a simple harness: a chat page interface online
> - GitHub Pages only (no backend)
> - OpenRouter API keys through CORS on the front-end
> - storage in local storage
>
> Never ask me to paste my OpenRouter key into this terminal, and never commit a key. The key is entered only in the running page.

That message fixes most of the design: no server, so the browser holds a bearer credential and spends it directly. This ADR covers the one part it leaves open, which is how the page *holds* the key between uses.

**Agency Grant**: Explore and recommend. Yuxiang decides custody; I implement whatever is chosen.

---

## Explicitation

**What I understand**:

1. **Every GitHub Pages project site on your account shares one origin.** Browser storage is partitioned by origin (scheme + host + port), not by path. `https://stanley-zhou.github.io/chat-harness/` and `https://stanley-zhou.github.io/pilot-light/` are *the same origin*, so they read and write the same localStorage. This is not hypothetical here. `pilot-light` is live at that origin (`gh api repos/stanley-zhou/pilot-light/pages` → `html_url: https://stanley-zhou.github.io/pilot-light/`), and `wordle-with-records` is configured to deploy there and already persists to `wordle-with-records:board:v1` (`src/game/persist.ts:23` in that repo). As things stand today, a key this page puts in localStorage can be read by any script any of those sites ever runs. That includes their dependencies, and any CDN script one of them adds later.
2. **Conversations are covered by the same fact.** Whatever you say to a model and store in localStorage can be read by sibling sites too. That is less severe than the key, because it is not spendable, but it is private.
3. **The quota is shared as well.** Browsers give each origin a single localStorage budget of roughly 5 MB, so long chat histories here compete with pilot-light's and wordle's storage. That does not block v1, but it is why conversations should be stored under a namespaced, versioned prefix (see Assumptions).
4. **The other way the key leaks is script running on *this* page**, meaning model output rendered as HTML. That is ADR-0003, and the two decisions constrain each other: the more readily the key can be read, the more rendering safety matters.
5. **What OpenRouter itself allows was checked, not assumed** (2026-11-12, preflight sent with a generic `Origin: https://example.github.io` and no key). `OPTIONS /api/v1/chat/completions` → `204`, `Access-Control-Allow-Origin: *`, with `Authorization` among the allowed headers. `OPTIONS /api/v1/auth/keys` → same. OpenRouter's docs describe an OAuth PKCE flow (`callback_url`, `code_challenge`, exchange at `/api/v1/auth/keys`), so a page can *obtain* a key without anyone pasting one. That is option D below.

**Assumptions** (stated, not asked, because each is cheap to reverse):

- Every key this page writes is namespaced `chat-harness:` and versioned (`chat-harness:v1:conversations`), so it never collides with `wordle-with-records:*` or anything else on the origin, and a later schema change can migrate data instead of corrupting it.
- The key never appears in a URL, in `console.log`, or in an error message the page displays. A visible **Forget key** control clears it from every store.
- You use a **dedicated key with a credit limit** set in OpenRouter's key settings, not your main key. Whatever we choose below, this is what caps the cost of a leak at a number you picked.

**Confirm**: Yuxiang, on the shared-origin reading (item 1) above all. If you plan to move this to a custom domain, the threat model changes, and so does the recommendation below.

---

## Questions

### QST-KEY-CUSTODY: Where does the page keep the OpenRouter key between uses?
- Status: unanswered
- Why asking: The page has to hold a bearer credential, and every option trades convenience against exposure. The exposure here is unusual. Your GitHub Pages project sites (pilot-light, wordle-with-records, this one) all share the origin `https://stanley-zhou.github.io`, and browsers partition storage by origin, not by path. So anything stored in localStorage can be read by every script on every one of those sites. You also said "storage in local storage", and I can't tell from that whether you meant the key or only the conversations. Conversations go to localStorage under every option below; this question is only about the key.
- Need: pick a letter (or override — any shape answers)

Options:
- **A — Remember in localStorage**: paste once per browser and it stays. This is the most convenient option. The key sits in plain text where every script on `stanley-zhou.github.io` can read it, across all your project sites and their dependencies, now and in the future.
- **B — Session by default, opt-in remember**: the key lives in sessionStorage, so it survives a reload but is gone when the tab closes. A **Remember on this device** checkbox, with a one-line warning, moves it to localStorage. Cost: you paste again in each new tab or session unless you opt in. One honest caveat: sessionStorage is shared by same-origin pages *in the same tab*, so following a link from the harness to `/pilot-light/` in that tab would expose it to pilot-light's scripts.
- **C — Passphrase-encrypted at rest**: the key is encrypted with AES-GCM (WebCrypto) under a key derived from a short passphrase (PBKDF2), and only ciphertext goes to localStorage. You type the passphrase once per session. Sibling sites can see only ciphertext. Cost: you have to remember a passphrase, there is noticeably more code, and it does nothing against script on *this* page while the key is unlocked (ADR-0003's territory).
- **D — Connect with OpenRouter (OAuth PKCE)**: instead of pasting, you click a button, approve on openrouter.ai, and the page receives a freshly minted key. This is about how the key is *acquired*, so it still needs one of A–C for storage. Cost: the callback URL must match where the page is served (local dev and Pages differ), and each connect mints another key, which you manage on openrouter.ai.

**Recommendation**: (by Heddle 5.5, Claude Opus 5.5, 2026-11-12)

**B — session by default, opt-in remember.** *Rationale*: The shared origin is the deciding fact. I verified it: pilot-light is live on `stanley-zhou.github.io`, and wordle already writes `wordle-with-records:board:v1` there. Given that, plain localStorage (A) means your OpenRouter key is readable by code that has nothing to do with this project. B costs one paste per session and lets you turn on A's convenience per device, as a deliberate choice instead of a default. C buys real protection against sibling sites, but at a complexity price that is out of proportion to a "simple harness", and it does nothing about the more likely leak, which is rendering (ADR-0003). D is a good feature but answers a different question: it can be added on top of B later without rework, because acquisition and storage are separate functions. Whichever letter you pick, the credit-limited dedicated key (Assumptions) is what actually bounds the damage. *Confidence*: 0.7, because the exposure is verified while the convenience cost is yours to weigh, and I can't see how often you'll open new tabs. That puts it in the "your call" band. *If wrong*: if re-pasting gets annoying within the first week *and* every script on your github.io origin is code you wrote, this flips to **A**. If you never want to handle the key string at all, add **D** on top of B.

**ANS:** (by Yuxiang Zhou)
[Fill this in]   <!-- literal placeholder — parser-significant, do not paraphrase -->

---

## Supporting Materials

### API: the CORS preflight that confirms the premise
**Request** (generic origin, no key, 2026-11-12):
```http
OPTIONS /api/v1/chat/completions HTTP/1.1
Host: openrouter.ai
Origin: https://example.github.io
Access-Control-Request-Method: POST
Access-Control-Request-Headers: authorization,content-type
```

**Response** (abridged):
```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Authorization,…,Content-Type,…,HTTP-Referer,…,X-Title,…
Access-Control-Allow-Methods: GET,OPTIONS,PATCH,DELETE,POST,PUT
```

`HTTP-Referer` and `X-Title` are allowed, so the page can also identify itself to OpenRouter's app attribution if you want it to. That is optional.

### FIL: evidence for the shared origin
- `gh api repos/stanley-zhou/pilot-light/pages`: `html_url` is on `stanley-zhou.github.io`, `status: built`
- `../wordle-with-records/src/game/persist.ts:23`: `const KEY = 'wordle-with-records:board:v1'`, which is localStorage on the same origin once deployed

---

## Decision

*Pending QST-KEY-CUSTODY.*

---

## Open Follow-ups

- [ ] Future task: if conversation histories outgrow the shared ~5 MB localStorage budget, move them to IndexedDB, which is also per-origin but has a far larger quota. The namespaced, versioned keys make that a migration rather than a rewrite.
- [ ] Concern: a custom domain or a dedicated user site would give the page its own origin and remove items 1–3 entirely. That is not proposed now, because the founding message says GitHub Pages and a domain costs money, but it is the structural fix if the threat model ever matters more.

---

## Action Items

- [ ] Yuxiang: answer QST-KEY-CUSTODY - Owner: Yuxiang Zhou
- [ ] Create a dedicated OpenRouter key with a credit limit before first use. This happens on openrouter.ai, and the key is never pasted anywhere but the running page - Owner: Yuxiang Zhou

---

## Iterations

### Iteration 1 (2026-11-12)
- Trigger: founding message
- Contributors: Heddle 5.5 (Claude Opus 5.5)
- Changes: initial draft. Shared origin verified via `gh api` and the wordle source. CORS and the PKCE flow verified against OpenRouter.
- Outcome: Draft, awaiting QST-KEY-CUSTODY

---

## Glossary (optional)

- **Origin**: scheme + host + port, the unit browsers use to partition storage. Path is not part of it, which is the whole problem here.
- **Bearer credential**: a secret that grants access to whoever presents it, with no further proof needed. An OpenRouter API key is one.

---

## Links

- Related ADRs: [ADR-0002](0002-stack-and-deploy-under-the-actions-lock.md) (stack: a no-dependency page puts no third-party script on the origin), [ADR-0003](0003-rendering-model-output-safely.md) (the same-page leak path)
