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

Authors retain `build` in service Compose files for local development. VAD and
the release workflow derive the published image tag from the service package and
build context:

```yaml
services:
  notes-app:
    build: ./app
    ports:
      - "127.0.0.1:10001:10001"
```

Set `configuration.service_image_registry` in `game.yaml` to the single GHCR
package players will pull, for example
`ghcr.io/YOUR_GITHUB_OWNER/YOUR_GAME_REPOSITORY`. The author Compose file is
directly runnable locally. VAD produces the player copy by replacing buildable
services with image references such as `notes-app`, removing local port
publishing and local network attachments, then joining the services to its
generated VPN namespace. The generated team bundle pulls its small WireGuard
attachment from `ghcr.io/vidner/vad-vpn:latest`; the team server never builds it.

Normal pushes do not build images. A pushed `release-*` tag runs the included
workflow, which:

1. discovers every `services/*/compose.yaml` dynamically;
2. injects image tags for buildable services; and
3. builds and pushes those images into one GHCR package: `<repository>`.

No personal access token is stored. GitHub Actions authenticates with its
short-lived `GITHUB_TOKEN`.

## GitHub setup

Before the first release:

1. Enable GitHub Actions and allow the workflow `packages: write` permission.
2. Set `configuration.service_image_registry` in `game.yaml` to the lowercase
   GHCR package name for the game repository.
3. Protect the `release-*` tag pattern with a repository ruleset.

Publish a tested commit with a deterministic tag:

```sh
git tag "release-$(git rev-parse --short=12 HEAD)"
git push origin --tags
```

The first run creates `<repository>` as a private package. When the release is
ready for players, change that package to **public** in its package settings.
Public GHCR images can then be pulled anonymously; GitHub does not allow a
public package to be changed back to private.

Players receive an image-only Compose bundle and start it normally:

```sh
cd team-bundle/server
docker compose up -d
```

## Checker SDK

Checker images install the public `vad-checker` SDK from a pinned release of
[`vidner/vad-sdk`](https://github.com/vidner/vad-sdk). Pin the version in every
checker Dockerfile and upgrade it deliberately.

Install the same SDK release in your authoring environment, then run service
integration tests from the repository root:

```sh
python -m pip install https://github.com/vidner/vad-sdk/archive/refs/tags/v0.2.1.zip
python -m vad_checker.integration <service-id>
```

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
