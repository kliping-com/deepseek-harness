# Agent Note: Kliping product brand, without a mark

Status: implemented

English | [中文](2026-08-24-kliping-product-brand.zh.md)

## Problem

The shipped product presented DeepSeek's whale artwork and the `DSH Local Build` label in the sidebar brand row, the same whale plus `Into the Unknown` in the blank-session hero, `DeepSeek Harness` in the browser title and install manifest, and `DeepSeek Harness` in the three system-prompt sentences that tell a model what it is running inside. A deployment that ships under its own name therefore leaked the upstream brand into every surface a person and a model can see, and the shell's own fallbacks — not just the optional official-brand package — carried the artwork.

## Decision

The shipped brand is the product name `Kliping` as text, with no mark anywhere.

The sidebar brand row renders the product name and the seven-character build revision; the mark slot stays declared and unoccupied. The collapsed rail keeps the panel toggle alone, because a rail whose resting state was the mark would otherwise render an invisible button. The blank-session hero renders the same product name beside its `Preview` badge, and its headline is a flex line rather than fixed grid tracks, which would reserve a column and a gap for a mark nothing occupies.

Neither surface wraps its mark slot in a box of its own. The slot renderer emits its occupant under a `display: contents` element, which generates no box at all while the slot is unoccupied, so the flex row simply has one item fewer; a shell-owned wrapper would instead stand as a zero-width flex item and still collect the row's gap. An occupant therefore owns its own box and receives the requested size.

[`client-ui-brand-official`](../../../../packages/client/ui-brand-official/README.md) now fills `sidebar.brand.name` alone under the `official` build profile, so an official build shows the product name without the build revision while every other build keeps the shell fallback that carries it. Its dependencies on the conversation and primitives packages are withdrawn with the occupants they served, and `FishLogo`/`BrandWordmark` are deleted from [`client-ui-primitives`](../../../../packages/client/ui-primitives/README.md) so the Web client ships no brand artwork at all. The documentation site keeps its own wordmark, because it publishes the upstream documentation corpus rather than the product.

The browser title, the PWA manifest name, the official build profile's `DSH_CLIENT_TITLE`, and the favicon (a neutral letter monogram that keeps the tested light/dark rule) carry the same name. The three model-visible identity sentences — the [system-prompt](../../../../packages/core/system-prompt/README.md) opener, the app-boot checkout sentence, and the web-app GUI sentence — name Kliping, and every recorded snapshot moves with them. The GUI welcome notice names Kliping and bumps its version so an acknowledged notice is shown once more.

Package names, the `dsh` command, `DSH_*` variables, `$DSH_HOME`, the `@deepseek-ai` npm scope, and the DeepSeek model-provider name are deliberately unchanged: the first four are technical identifiers, and the last names the actual API vendor.

## Alternatives considered

**Keep the whale and change only the label.** Rejected because the request was a brand replacement, and a mark is the half a person recognizes first; keeping it would have made the text change read as a typo rather than a rebrand.

**Delete `client-ui-brand-official` entirely.** Its remaining difference from the shell fallback is one build-revision badge, which invited deletion. Rejected because the package is the worked example of the declaration-aware brand-slot registration, and removing it would have touched the web-app bundle, both client catalogs, the module graph, and every tsconfig aggregate for a cosmetic gain.

**Rename the npm scope and the `dsh` command in the same change.** Rejected as a separate mechanical project: the scope reaches thousands of files, the lockfile, tsconfig references, and generated catalogs, and the pre-release stance already permits doing it freely later, when a rescope is the whole change rather than a passenger on a brand change.

**Leave the model-visible identity alone.** Rejected because the system prompt states the product's name to the model on every request; a rebrand that stops at the pixels leaves the agent introducing itself under the old name.

## Consequences

The shell now ships no brand artwork at all, so a deployment that wants a mark occupies `sidebar.brand.mark` or `conversation.hero.brand.mark` rather than replacing a fallback. Both slots stay declared for exactly that.

The collapsed rail lost its mark-to-panel-icon hover swap along with the mark, and the CSS rules for it are gone; the rail is now a plain toggle in both resting and hover states.

Thirty recorded snapshot artifacts moved with the source strings. They were edited in lockstep rather than re-recorded, because re-recording needs a provider key; replay compares regenerated output against them, so both sides changed together.

The welcome-notice version bump means every user who already acknowledged the notice sees it once more.

## Testing

Unit coverage moves with the behavior: the sidebar spec asserts the product name, the build revision, and that the brand row carries no `svg`; the hero spec asserts the product name and that the mark slot is rendered without a fallback; the brand-official spec asserts one occupant and text-only rendering. `pnpm run typecheck`, `pnpm run verify-translation-pairing`, and the affected package suites pass. The browser end-to-end and snapshot-replay suites need a complete `pnpm run build` and were left to CI.
