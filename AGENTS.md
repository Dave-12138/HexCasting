# AGENTS.md

## Workspace layout

- `HexMod/` — reference clone of https://github.com/FallingColors/HexMod (branch `1.21`).
- `PAUCAL/` — reference clone of https://github.com/FallingColors/PAUCAL (branch `1.21`), HexMod's dependency library.
- `.github/workflows/build.yml` — aggregate CI: manual (`workflow_dispatch`) build of both repos, checked out from GitHub (workspace clones are reference only, not required at workflow runtime).

## Verified build facts

- Both are Architectury Gradle multi-module projects: `Common`, `Fabric`, `Neoforge`; built with JDK 21 (`actions/setup-java`, temurin 21) and `./gradlew build`. Gradle wrapper is committed in each repo.
- HexMod (1.21) pins `paucalVersion=0.7.1-pre-27` in `HexMod/gradle.properties` and consumes PAUCAL as `at.petra-k:paucal:<ver>+1.21.1-{common,fabric,neoforge}` from `https://maven.blamejared.com` (hexcasting.fabric/neoforge also declares `mavenLocal()`). Its own `HexMod/.github/workflows/pr.yml` builds standalone without a local PAUCAL checkout.
- PAUCAL (1.21): `mod_version=0.7.1`, `minecraft_version=1.21.1`, group `at.petrak` (published as `at.petra-k`); versioning driven by external `at.petra-k.pkpcpbp.PKPlugin` + Jenkins `BUILD_NUMBER` (hence `-pre-N` in artifact versions), so a local `publishToMavenLocal` does not reliably satisfy HexMod's pinned version.
- Both repos archive jars under `Common|Fabric|Neoforge/build/libs`; reference Jenkins pipelines archive `Common/Fabric/Neoforge/build/libs` jars there.
- CI pitfall: checkout must use `fetch-depth: 0` — the Gradle configuration of `:Common` runs git range queries (`HEAD~..HEAD`), which fail on a shallow (default `fetch-depth: 1`) clone with `fatal: ambiguous argument 'HEAD~..HEAD'`. HexMod's own `pr.yml` also checks out with `fetch-depth: 0`.

## Environment constraints

- Docker container: cannot push (no credentials), cannot restart dsh, avoid host-side pnpm builds (see `~/.dsh/AGENTS.md`).