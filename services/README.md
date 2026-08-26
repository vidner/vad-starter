# Service packages

Create one directory per player-facing challenge service:

```text
services/<service-id>/
├── manifest.yaml
├── compose.yaml
├── app/
│   ├── Dockerfile
│   └── ...
└── README.md
```

The directory name is the stable service ID and must match the corresponding
`checkers/<service-id>` directory and the entry in `game.yaml`.

Read the complete [service and checker authoring guide](../docs/service-checker-authoring.md)
before adding a package.

## Player bundle boundary

The required `bundle` list in `manifest.yaml` is deny-by-default. It normally
contains only `compose.yaml`:

```yaml
id: notes
authors:
  - alice
stores:
  - private_notes
ports:
  - 10001
checker_timeout: 10s
bundle:
  - compose.yaml
```

`compose.yaml` is an ordinary standalone project for author testing. Publish
local-only host ports there and run it with `docker compose up --build`.

When VAD assembles a player bundle, it replaces buildable services with image
references from `configuration.service_image_registry`, removes local port and
network settings, and places every container in the generated VPN service's
network namespace. Compose service names remain available for
container-to-container connections. Declare the TCP ports reachable through the
tournament VPN in the manifest, and prefix Compose service names with the stable
service ID so they remain unique across the game.

Never include checker code, exploits, credentials, organizer notes, or signing
secrets in the bundle or service image. Treat every image layer as player-visible.
