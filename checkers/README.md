# Checker packages

Create one private organizer-only checker package per challenge service:

```text
checkers/<service-id>/
├── compose.yaml
├── Dockerfile
└── src/
    └── main.py
```

The directory name must match `services/<service-id>` and the service ID enabled
in `game.yaml`. Read the complete
[service and checker authoring guide](../docs/service-checker-authoring.md)
before implementing a checker.

The package Compose file defines only the checker build:

```yaml
services:
  checker:
    build:
      context: ../..
      dockerfile: checkers/notes/Dockerfile
    restart: unless-stopped
```

VAD's checker deployment supplies the VPN sidecar, NATS connection, service ID,
durable name, and tunnel configuration. Do not duplicate those settings or add a
second VPN container.

Install the public checker SDK from a pinned `vad-sdk` release. Checkers must
implement `PUT`, `GET`, and `CHECK`, use `context.host` as the target, preserve
old flags for their configured lifetime, and return only minimal attacker-useful
state as public state.

From the game repository root, run the SDK-owned integration contract with:

```sh
python -m vad_checker.integration <service-id>
```

Checker sources are never included in player bundles. Keep the generated game
repository private until your disclosure policy permits publishing checker
logic, intended vulnerabilities, and exploits.
