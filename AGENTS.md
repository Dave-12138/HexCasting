# HexCasting (Dave-12138/HexCasting)

## Project
- Purpose: minimal repo (GitHub `Dave-12138/HexCasting`, branch `build`, origin remote above) hosting an aggregate CI for two upstream Minecraft mod repos: `FallingColors/HexMod` and `FallingColors/PAUCAL` (its dependency library).
- The upstream sources are **not** in this repo and must not be assumed present locally — the workflow checks them out at runtime. Earlier reference clones under `HexMod/` / `PAUCAL/` were removed.
- Only content: `.github/workflows/build.yml` (the deliverable) plus this file. Git history: single commit `init`; repo was `git init`-ed at the workspace root.

## Commands
- There is nothing to build locally: this repo has no sources, only a GitHub Actions workflow.
- YAML sanity: no YAML parser is installed in the container (no pyyaml/ruby); structural errors surface only when GitHub parses the workflow at run start.
- Re-clone an upstream for reference: `git clone --branch 1.21 https://github.com/FallingColors/HexMod.git` / `.../PAUCAL.git`.
- Container cannot `git push` (no credentials) — committing/pushing is done by the developer.

## Architecture
- `.github/workflows/build.yml`: manual (`workflow_dispatch`) aggregate build of both upstream repos, each in its own parallel job:
  1. `actions/checkout@v7` remote repo into `PAUCAL` / `HexMod`, `ref` from input (default `1.21`), **`fetch-depth: 0`** (see Pitfalls);
  2. `actions/setup-java@v6` temurin 21 + `gradle/actions/setup-gradle@v6`;
  3. `./gradlew build` (matches each upstream repo's own CI: HexMod `.github/workflows/pr.yml`, both Jenkinsfiles);
  4. stage jars from `Common|Fabric|Neoforge/build/libs` into `dist`, dropping `-sources/-javadoc/-shadow/-dev*` jars; upload as `paucal-build` / `hexmod-build` (`upload-artifact@v7`, 30 days).
- Optional `publish-release` job: runs only when inputs `publish_release == 'true'` **and** `release_tag != ''`; `needs` both build jobs; downloads both artifacts; `softprops/action-gh-release@v3` creates/updates the release for the input tag and attaches all jars (`release/paucal/**`, `release/hexmod/**`). Release is created in **this** repo (GITHUB_TOKEN cannot write to the upstream repos).
- Upstream build facts (verified against branch `1.21` heads, session-2025-09): both are Architectury Gradle multi-module projects (`Common`, `Fabric`, `Neoforge`), JDK 21, wrapper committed per repo.
  - HexMod (`gradle.properties`) pins `paucalVersion=0.7.1-pre-27`, `minecraftVersion=1.21.1`; consumes PAUCAL as `at.petra-k:paucal:<ver>+1.21.1-{common,fabric,neoforge}` from `https://maven.blamejared.com` (also declares `mavenLocal()`); builds standalone without a local PAUCAL checkout.
  - PAUCAL (`gradle.properties`): `mod_version=0.7.1`, `minecraft_version=1.21.1`, maven group `at.petra-k`; the `-pre-N` suffix in published versions comes from Jenkins `BUILD_NUMBER` via the external `at.petra-k.pkpcpbp.PKPlugin`.

## Conventions
- `workflow_dispatch` inputs: `hexmod-ref` / `paucal-ref` (default `1.21`), `publish_release` (boolean, default false), `release_tag` (string, default empty).
- Action versions kept at/above skill minimums to avoid Node 20 deprecation warnings: checkout@v7, setup-java@v6, setup-gradle@v6, upload/download-artifact@v7, softprops/action-gh-release@v3.
- Keep `fetch-depth: 0` on every checkout step — do not remove it (see Pitfalls).

## Pitfalls
- Shallow checkout breaks the build: Gradle configuration of `:Common` runs git range queries (`HEAD~..HEAD`); with the default `fetch-depth: 1` this fails with `fatal: ambiguous argument 'HEAD~..HEAD'`. HexMod's own `pr.yml` also uses `fetch-depth: 0`.
- A local PAUCAL `publishToMavenLocal` does **not** reliably satisfy HexMod's pinned `0.7.1-pre-27` coordinate (PKPlugin/Jenkins versioning), so don't try to force a composite local-PAUCAL build.
- Release publishing needs `permissions: contents: write` on the job (top level is `read`); rerunning the same tag updates the existing release and appends assets (softprops v3 behavior).
- `publish_release=true` with empty `release_tag` silently skips the release job (guard condition), it does not fail the run.
- Do not run full Gradle builds of the upstream repos inside this container: each pulls hundreds of MB to GBs of Minecraft dependencies, and artifacts are not the deliverable.
- `.github/workflows/build.yml` was originally a corrupt binary placeholder (text readers reject it); if it ever looks broken again, delete and rewrite rather than editing bytes.

## Maintenance
- Update this file in place whenever a repo-specific command, convention, or pitfall is discovered; remove entries that go stale.
- Upstream facts above were verified when the branch-`1.21` clones were present; re-verify against fresh clones before trusting them after significant upstream changes.
