# @deepseek-ai/dsh-client-ui-brand-official

English | [中文](README.zh.md)

This package fills `sidebar.brand.name` only when `DSH_CLIENT_BUILD_PROFILE` is `official`. Other builds load the plugin but register no occupant, leaving the shell fallback visible: the same product name followed by the build's 7-character `DSH_CLIENT_COMMIT_HASH` badge, so a local build is never mistaken for a release.

The occupant is the product name as text. The shipped brand carries no mark and no artwork, and `sidebar.brand.mark` and `conversation.hero.brand.mark` stay unoccupied for a deployment package that wants one. The registration installs through `slots.inject()`, so the package works whether its row activates before or after the sidebar declarer, withdraws its occupant when the declaration collapses, and leaves no partial brand mix during HMR. It retains no runtime state. The node half is an empty Loader seat, and the browser title remains a build-environment concern outside this package.

## Model Experience

None, as the package contributes browser presentation only; nothing here reaches a model request.

#### KV Cache effect

None; this package neither assembles nor sends a provider request.

## Known Limitations and Deferred Work

- **The package supplies one occupant** — alternative presentation belongs in another Cordis package occupying the same slot.
- **The browser title is independent** — `DSH_CLIENT_TITLE` selects title text at build time rather than through a UI slot.
