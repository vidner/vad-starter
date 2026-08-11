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
stores:
  - private_notes
ports:
  - 10001
checker_timeout: 10s
bundle:
  - compose.yaml
```

VAD removes `build` from Compose services that also declare `image`, so players
receive image-only definitions. Source remains in this author repository and in
whatever form the built image layers expose.

Every application container must use `network_mode: service:vpn`. Do not publish
ports with Compose `ports` or `expose`; declare reachable TCP ports only in the
manifest. Prefix Compose service names with the stable service ID so they remain
unique across the game.

Never include checker code, exploits, credentials, organizer notes, or signing
secrets in the bundle or service image. Treat every image layer as player-visible.
