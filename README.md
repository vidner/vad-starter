# VAD game starter

A GitHub repository template for authoring an attack-defense game for
[VAD](https://github.com/vidner/vad). Each challenge consists of a player-facing
service and a private checker with the same stable ID.

```text
game.yaml   Game identity, timing, networks, and enabled service IDs
services/   Challenge sources and player Compose definitions
checkers/   Organizer-only checker implementations
docs/       Service and checker authoring guide
.github/    Tag-triggered GHCR build and release workflow
```

This public template intentionally contains no real challenge, checker, exploit,
credential, or flag material.

## Start a game repository

1. Select **Use this template** on GitHub and create a new repository.
2. Make the generated repository **private** while a game is under development.
   Checker logic, intended vulnerabilities, exploits, and organizer notes can
   reveal the challenge design.
3. Replace the example identity and settings in [`game.yaml`](game.yaml).
4. Create matching `services/<id>` and `checkers/<id>` packages using the
   [authoring guide](docs/service-checker-authoring.md).
5. Add the ID to `game.yaml` only after both packages exist.

The service ID is the shared directory name. It must be lowercase and match
`^[a-z][a-z0-9_-]{0,63}$`.

## Service images

Authors retain `build` in service Compose files for local development. Each
buildable service also declares the public image players will pull:

```yaml
services:
  notes-app:
    image: ${VAD_IMAGE_REGISTRY:-ghcr.io/YOUR_GITHUB_OWNER/YOUR_GAME_REPOSITORY-release}:notes-app${VAD_IMAGE_SUFFIX:-}
    build: ./app
    network_mode: service:vpn
```

Replace `YOUR_GITHUB_OWNER/YOUR_GAME_REPOSITORY` in every service Compose file
with the lowercase owner and generated repository name. VAD removes `build` from
the player copy, leaving only the published image reference. The generated team
bundle still builds its small WireGuard attachment locally.

Normal pushes do not build images. A pushed `release-*` tag runs the included
workflow, which:

1. discovers every `services/*/compose.yaml` dynamically;
2. builds commit-addressed images into a private `<repository>-staging` GHCR
   package; and
3. promotes them into `<repository>-release` only if every build succeeds.

No personal access token is stored. GitHub Actions authenticates with its
short-lived `GITHUB_TOKEN`.

## GitHub setup

Before the first release:

1. Enable GitHub Actions and allow the workflow `packages: write` permission.
2. Create an `image-release` environment and add the desired deployment
   protection or required reviewers.
3. Protect the `release-*` tag pattern with a repository ruleset.

Publish a tested commit with a deterministic tag:

```sh
git tag "release-$(git rev-parse --short=12 HEAD)"
git push origin --tags
```

The first run creates `<repository>-staging` and `<repository>-release` as
private packages. Keep staging private. When the release is ready for players,
change only `<repository>-release` to **public** in its package settings. Public
GHCR images can then be pulled anonymously; GitHub does not allow a public
package to be changed back to private.

Players receive an image-only Compose bundle and start it normally:

```sh
cd team-bundle/server
docker compose up -d
```

## Checker SDK

Checker images install the public `vad-checker` SDK from a pinned release of
[`vidner/vad-sdk`](https://github.com/vidner/vad-sdk). Pin the version in every
checker Dockerfile and upgrade it deliberately.

## Security boundary

Assume players can inspect every service image, filesystem layer, environment
value, and bundled file. Never put checker code, exploits, organizer tokens,
cloud credentials, signing secrets, or unpublished writeups into a service image
or its manifest `bundle` list.

The flag HMAC secret and runtime infrastructure credentials belong in the VAD
deployment environment, never in this repository.

## License

The starter scaffolding is available under the [MIT License](LICENSE). Challenge
authors should review whether their generated repository needs different
licensing or disclosure terms.
