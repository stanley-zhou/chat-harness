<!-- adr template version: "adr 3.12.0" -->
<!--
adr teaching map: the example is not the teaching — these are the up-links from this file to what teaches its form; load them rather than winging it. (Sibling: the template-version marker dates the mold this file was minted from.)
  skill: adr-authoring
  mcp: adrs4ai
  plugin: adr@adrs4ai
-->

# ADR-0002: Stack and deploy under the Actions lock

- **Date**: 2026-11-12
- **Iteration**: 1
- **Status**: Draft
- **Deciders**: Yuxiang Zhou, Heddle 5.5 (Claude Opus 5.5)

**TL;DR**: Branch-based Pages deploys turn out to be Actions runs too, so no deploy path is known to escape the billing lock; choose the stack on its merits, picking the shape that goes live with a single settings change once the lock lifts.

---

## Originating Context

**Source**: `Human dump`, the founding message, 2026-11-12: *"GitHub Pages only (no backend)"*, plus a constraint I could not have known:

> GitHub Actions on my account is currently blocked ("account is locked due to a billing issue"), so an Actions-based Pages deploy won't start until I fix billing. Factor that into any deploy decision.

There is also a precedent. In `wordle-with-records`, ADR-0003 chose the official Actions deploy *before* the lock was known. Its first run died in 3 seconds with that annotation, and its follow-up question (QST-DEPLOY-BLOCKED there) recommended "wait", with "push built output to a branch" as the fallback. This ADR starts from what that episode learned.

**Agency Grant**: Explore and recommend. Yuxiang decides the stack.

---

## Explicitation

**What I understand**:

1. **The obvious way around the lock is probably not one.** "Deploy from a branch" looks Actions-free, but on today's GitHub it isn't: branch-sourced sites are built by a GitHub-run workflow called `pages-build-deployment`. I checked this on your own account rather than relying on docs. `pilot-light` is a branch-sourced site (`build_type: "legacy"`, source `main` `/`), and `gh run list -R stanley-zhou/pilot-light` shows each deploy as a `pages-build-deployment` run with event `dynamic`.
2. **Whether the lock blocks those runs is unverified, and the evidence available can't settle it.** pilot-light's last successful build was 2026-11-12 03:34 UTC. Wordle's blocked run was 2026-11-13 00:17 UTC, about 21 hours later, and I don't know when the lock began. So the honest position is that no GitHub Pages deploy method is known to work right now. Settling it takes one probe (Action Items), and since the probe pushes to GitHub, it waits for your OK.
3. **That changes what this decision is about.** If every path may be blocked, the deploy method can't be chosen for "works tonight". The decision becomes two things: (a) which stack is right on its merits, and (b) which shape goes live with the *least* work on the day the lock lifts, while being fully usable locally until then.
4. **The stack decides what a "deploy" is.** With no build step, deploying means serving the committed files, and a one-time Pages setting is the whole pipeline. With a build step, *something* has to run the build and publish its output: either Actions (blocked) or a manual local build pushed to a branch (which per item 1 probably triggers an Actions run anyway).
5. **You have a stated reason to want TypeScript.** Your answer to wordle's QST-TYPESCRIPT was that you're targeting AI engineering roles where TS is common, so learning it is part of the point. That deserves weight here and not a reflexive "keep it simple". Option A below keeps a typed path open without a build.

**Assumptions** (stated, not asked):

- **Local dev uses any static server**, e.g. `python -m http.server` (on this box, `python` and not `python3`, which is the Store stub) or `npx http-server`. ES modules refuse to load from `file://`, so double-clicking `index.html` won't work. OpenRouter accepts any origin (ADR-0001 verified `Access-Control-Allow-Origin: *`), so `localhost` can call it directly.
- **A `.nojekyll` file at the repo root** for any branch-served option. Without it, GitHub's branch build runs Jekyll over the whole repo, including `docs/adr/` and the skills, where any `{{ … }}` in markdown is parsed as Liquid and can fail the build for reasons unrelated to the app.
- **The site lives at `https://stanley-zhou.github.io/chat-harness/`**, a project site, matching how pilot-light and wordle are hosted. No custom domain (ADR-0001's follow-ups say what that would change).

**Confirm**: Yuxiang, mostly on item 2. If you know the lock started before 2026-11-12 03:34 UTC, pilot-light's build is evidence that branch deploys *survive* it, and that changes the cost of option C.

---

## Questions

### QST-STACK-DEPLOY: No-build static page, or a TypeScript build — and so which deploy path?
- Status: unanswered
- Why asking: Choosing a stack also chooses a deploy path, and your Actions account is locked for billing. I checked something the obvious workaround depends on: GitHub's "Deploy from a branch" is itself an Actions run (`pages-build-deployment`, visible in pilot-light's run list). So none of the options below is known to deploy *today*. That means the choice should rest on what the code needs, and on how little work it takes to go live once billing is fixed. In wordle you chose TypeScript to learn it for AI-engineering roles. That is a real input here, which is why this is a question and not a default.
- Need: pick a letter (or override — any shape answers)

Options:
- **A — No-build static page**: `index.html` + plain ES modules + CSS, committed and served exactly as written, via Pages "Deploy from a branch" (`main`, `/`, with `.nojekyll`). There are no runtime dependencies. Tests run on the pure modules (the stream parser, storage) with Node's built-in `node --test`, which installs nothing. Types are optional but available: JSDoc annotations checked by `tsc --checkJs --noEmit` as a *dev-only* gate step that never touches what ships. Going live is a single Pages setting change, after which every push to `main` deploys.
- **B — Vite + TypeScript, Actions deploy**: wordle's shape. Real `.ts` files, bundling, and the official `upload-pages-artifact` → `deploy-pages` workflow. It's familiar from wordle and is the full TypeScript experience. Nothing goes live until the lock lifts, and there is a workflow file and a `node_modules` toolchain to maintain for a single page.
- **C — Vite + TypeScript, `dist/` pushed to a `gh-pages` branch**: build locally, commit the built output to a branch Pages serves. You get TypeScript and bundling without writing a workflow. Cost: built artifacts in git history, a manual deploy step that has to be remembered, nothing enforcing the gate before publish, and, per Explicitation item 1, the branch build is probably an Actions run too, so this probably doesn't escape the lock either.

**Recommendation**: (by Heddle 5.5, Claude Opus 5.5, 2026-11-12)

**A — no-build static page, with JSDoc types checked in the gate.** *Rationale*: The page is small: a message list, an input, a model picker, a settings panel, and one fetch with a streaming reader. Nothing in it needs a compiler. The lock removes B's and C's usual advantage, since neither ships sooner than A, while A goes live with the smallest change once billing is fixed: one Pages setting, no workflow, no artifacts. Zero runtime dependencies also helps ADR-0001: no third-party script on the page means nothing else running in the origin that holds your key. The TypeScript goal is served better than it looks, because `tsc --checkJs` *is* the TypeScript compiler. Message roles and stream states become checked unions, and `vue-tsc`'s role in wordle's gate is played here by `tsc`. What you lose is TS *syntax* and the bundler workflow. *Confidence*: 0.75. The lock evidence and the size of the app point clearly to A, but the learning goal is yours to weigh, and A teaches less of the TS toolchain than B does. *If wrong*: if writing actual `.ts` files and running a Vite build is part of what you want to practice, this flips to **B**. Under the lock B costs only waiting, since no option ships today. If the probe shows branch builds *do* run under the lock, nothing flips, but A is then live within minutes of the first commit.

**ANS:** (by Yuxiang Zhou)
[Fill this in]   <!-- literal placeholder — parser-significant, do not paraphrase -->

---

## Supporting Materials

### FIL: the evidence behind Explicitation items 1–2
- `gh api repos/stanley-zhou/pilot-light/pages`: `"build_type":"legacy"`, `"source":{"branch":"main","path":"/"}`, `"status":"built"`
- `gh run list -R stanley-zhou/pilot-light -L 3`: `pages build and deployment · pages-build-deployment · main · dynamic`, last `success` at `2026-11-12T03:34:07Z`
- `gh run list -R stanley-zhou/wordle-with-records -L 1`: `Deploy to Pages`, `failure` in 4s at `2026-11-13T00:17:48Z`, the billing-lock run
- `gh api repos/stanley-zhou/wordle-with-records/pages/builds`: `[]`, so wordle has never had a successful build

### COD: what the gate would grow into under A
```bash
# justfile `gate`, project slot — illustrative, lands with the first code commit
node --test tests/                 # stream parser, storage reads
npx -y -p typescript@5 tsc -p jsconfig.json --noEmit   # JSDoc types (checkJs)
git diff --cached | grep -E 'sk-or-[A-Za-z0-9-]{10,}' && exit 1 || true   # no staged key
```

---

## Decision

*Pending QST-STACK-DEPLOY.*

---

## Open Follow-ups

- [ ] Unverified assumption: whether the billing lock blocks `pages-build-deployment` runs. The probe in Action Items settles it.
- [ ] Concern: wordle-with-records' QST-DEPLOY-BLOCKED there is still unanswered. Its fallback (branch deploy) inherits the same finding as Explicitation item 1, which is worth carrying back to that record.

---

## Action Items

- [ ] Yuxiang: answer QST-STACK-DEPLOY - Owner: Yuxiang Zhou
- [ ] Probe, needs your OK because it pushes to GitHub: once a page exists, push `main`, set Pages to "Deploy from a branch", and see whether `pages-build-deployment` starts or fails with the billing annotation - Owner: Heddle 5.5
- [ ] Resolve the account lock at github.com/settings/billing - Owner: Yuxiang Zhou

---

## Iterations

### Iteration 1 (2026-11-12)
- Trigger: founding message + the Actions-lock constraint
- Contributors: Heddle 5.5 (Claude Opus 5.5)
- Changes: initial draft. Verified via `gh` that branch deploys are Actions runs. The lock's effect on them is flagged as unverified.
- Outcome: Draft, awaiting QST-STACK-DEPLOY

---

## Links

- Related ADRs: [ADR-0001](0001-openrouter-key-custody-in-a-shared-origin.md) (zero runtime dependencies = no third-party script on the key's origin), [ADR-0003](0003-rendering-model-output-safely.md) (option A's vendoring story for a markdown renderer)
- Precedent: `wordle-with-records` ADR-0003, QST-TYPESCRIPT (answered A) and QST-DEPLOY-BLOCKED (open)
