# Authoring VAD services and checkers

One VAD challenge has two deliberately separate packages:

```text
services/<service-id>/     Player-facing service and author build context
checkers/<service-id>/     Organizer-only availability and flag checker
```

The shared directory name is the stable service ID. It must be lowercase and
match `^[a-z][a-z0-9_-]{0,63}$`. It appears in game configuration, APIs,
checker jobs, metrics, and scoring data, so do not change it after a game starts.

For every active team, VAD runs an isolated service instance. Players and
checkers reach its declared TCP ports through the tournament WireGuard relay.

## Checker jobs

Each checker handles three job types:

| Job | Purpose | Successful behaviour |
| --- | --- | --- |
| `PUT` | Place a new flag in one named store. | Return minimal public state and checker-only private state. |
| `GET` | Verify one previously placed flag. | Recover and compare the exact supplied flag. |
| `CHECK` | Exercise ordinary functionality. | Confirm legitimate features still work without testing the vulnerability. |

A store is a distinct flag location or retrieval contract. A service can expose
multiple stores and vulnerabilities.

## 1. Create the service package

Start with:

```text
services/notes/
├── manifest.yaml
├── compose.yaml
├── app/
│   ├── Dockerfile
│   └── ...
└── README.md
```

Example `manifest.yaml`:

```yaml
id: notes

stores:
  - private_notes
  - shared_drafts

ports:
  - 10001

checker_timeout: 10s

bundle:
  - compose.yaml
```

Rules:

- `id` must equal the directory name.
- Store IDs follow the same lowercase identifier rule and must remain stable.
- Ports must be valid and unique across every enabled service in the game.
- Checker timeout must be greater than zero and no more than one minute.
- `bundle` is mandatory and deny-by-default. It supports exact paths and
  recursive `directory/**` entries and must include `compose.yaml`.
- Never bundle `manifest.yaml`, checker code, exploits, credentials, private
  notes, or organizer-only files.

VAD rejects selected symlinks, traversal, unmatched entries, files over 16 MiB,
packages over 128 MiB, and a selected bundle set over 256 MiB.

## 2. Write the service Compose fragment

Keep `build` for author development and declare the image players will pull:

```yaml
services:
  notes-app:
    image: ${VAD_IMAGE_REGISTRY:-ghcr.io/YOUR_GITHUB_OWNER/YOUR_GAME_REPOSITORY-release}:notes-app${VAD_IMAGE_SUFFIX:-}
    build: ./app
    environment:
      NOTES_DATA_DIR: /data
    volumes:
      - notes-data:/data
    network_mode: service:vpn

volumes:
  notes-data:
```

Replace the owner and repository placeholders before release. The included
workflow overrides `VAD_IMAGE_REGISTRY` and `VAD_IMAGE_SUFFIX` while building;
the literal default is what player Compose uses after VAD strips `build`.

Compose rules:

- Every service container, database, and sidecar uses
  `network_mode: service:vpn`.
- Do not use `ports` or `expose`; the manifest is the only public-port contract.
- Prefix Compose service names with the stable service ID.
- Containers sharing the VPN namespace communicate over `127.0.0.1` and must
  listen on distinct ports.
- Use named volumes for state that must survive restarts.
- Do not use privileged mode, host networking, host PID, or the Docker socket.
- Leave image-only dependencies such as `redis:7.2-alpine` on their upstream
  image rather than republishing them.

Players are expected to perform filesystem and image-layer analysis. Do not rely
on deleted layers, environment variables, or obscurity to protect organizer
secrets. Anything added during an image build can be recovered.

## 3. Create the checker package

Create the matching package:

```text
checkers/notes/
├── compose.yaml
├── Dockerfile
└── src/
    └── main.py
```

The checker Compose file contains only its build:

```yaml
services:
  checker:
    build:
      context: ../..
      dockerfile: checkers/notes/Dockerfile
    restart: unless-stopped
```

VAD's deployment overlay supplies the VPN sidecar, `network_mode`, NATS URL,
service ID, durable name, and tunnel configuration.

Example Python Dockerfile:

```dockerfile
FROM python:3.13-alpine
WORKDIR /app
RUN pip install --no-cache-dir https://github.com/vidner/vad-sdk/archive/refs/tags/v0.1.0.zip
COPY checkers/notes/src /app
USER nobody
CMD ["python", "main.py"]
```

Pin the SDK and all additional dependencies. Checker images remain private and
must never be shipped to teams.

## 4. Implement `PUT`, `GET`, and `CHECK`

The Python SDK provides `Context`, `State`, `Outcome`, and `serve`:

```python
from vad_checker import Context, Outcome, State, serve


class NotesError(Exception):
    def __init__(self, outcome: Outcome, detail_code: str):
        super().__init__(detail_code)
        self.outcome = outcome
        self.detail_code = detail_code


class NotesChecker:
    def base_url(self, context: Context) -> str:
        return f"http://{context.host}:10001"

    async def put(self, context: Context) -> State:
        assert context.store in {"private_notes", "shared_drafts"}
        assert context.flag is not None
        note_id, edit_token = await create_note(
            self.base_url(context), context.flag, context.idempotency_key
        )
        return State(
            public={"note_id": note_id},
            private={"edit_token": edit_token},
        )

    async def get(self, context: Context) -> None:
        assert context.flag and context.public and context.private
        value = await fetch_note(
            self.base_url(context),
            context.public["note_id"],
            context.private["edit_token"],
        )
        if value != context.flag:
            raise NotesError(Outcome.FLAG_MISSING, "flag_mismatch")

    async def check(self, context: Context) -> None:
        if not await ordinary_notes_flow(
            self.base_url(context), context.idempotency_key
        ):
            raise NotesError(Outcome.SERVICE_FAILURE, "ordinary_flow_failed")


serve(NotesChecker())
```

The example's application helpers are intentionally left for the challenge
author. `serve()` handles NATS, deadlines, retries, publishing, and
acknowledgements.

### State contract

| Field | `PUT` | `GET` | `CHECK` |
| --- | --- | --- | --- |
| `host`, `team`, `service`, `tick`, `idempotency_key` | present | present | present |
| `store` | present | present | absent |
| `flag` | new flag | exact existing flag | absent |
| `public` | absent | exact `PUT` public state | absent |
| `private` | absent | exact `PUT` private state | absent |

Successful `PUT` public and private states must each be JSON objects no larger
than 8 KiB. Public state is exposed to opponents through the target API; private
state is returned only to the matching future `GET`. Never put a flag, password,
bearer token, secret filesystem path, or checker-only credential in public state.

`GET` must compare the exact flag from its context. Missing/inaccessible flag
state normally maps to `FLAG_MISSING`; broken normal service behaviour maps to
`SERVICE_FAILURE`; checker bugs map to `CHECKER_FAILURE`.

Jobs can be retried. Make `PUT` and `CHECK` idempotent using
`context.idempotency_key` where possible. Never delete earlier flags, reset the
database, rotate global secrets, or assume a clean service instance.

`CHECK` must exercise legitimate functionality without testing the intended
vulnerability or reading flags. A defensive patch should pass `CHECK` if normal
functionality remains intact.

Checker stdout and stderr are collected by the organizer. Log useful detail
codes, but never intentionally print flags, private state, team tokens, or
credentials.

## 5. Enable the pair

Only after both directories exist, add the service ID to `game.yaml`:

```yaml
services:
  - notes
```

Platform bootstrap validates the manifest. Checker-node startup requires a
matching checker package for every enabled service. Changing IDs, stores, or
ports after registrations exist is a breaking game change.

## 6. Test and release

Before release:

1. Build the service and checker images locally.
2. Exercise every `PUT`, `GET`, and `CHECK` path, including retries with the same
   idempotency key.
3. Confirm the service retains every flag for at least `flag_lifetime_ticks`.
4. Start a generated team bundle on Linux and test every declared port from a
   player WireGuard peer.
5. Register multiple teams and verify isolated persistent data and distinct
   virtual target hosts.
6. Verify target API output contains only minimal public state.
7. Inspect the player ZIP and service image layers for accidental checker code,
   exploits, secrets, and organizer notes.
8. Test a real exploit and flag submission end to end.

Publish only a tested commit:

```sh
git tag "release-$(git rev-parse --short=12 HEAD)"
git push origin --tags
```

The tag-triggered workflow first builds every service into private staging. It
updates the release package only after all builds succeed. Normal branch pushes
do not consume image-building Actions minutes.

## Common failures

| Symptom | Usually means |
| --- | --- |
| Bootstrap rejects a package | Manifest ID, bundle allowlist, port, or Compose validation is wrong. |
| Service is unreachable | Wrong listen address/port or missing manifest port. |
| Compose rejects `ports`/`expose` | Remove them and use manifest ports plus `network_mode: service:vpn`. |
| Checker cannot connect | It hard-coded a target or ignored `context.host`. |
| Opponents cannot exploit a placed flag | Public state is insufficient to locate it. |
| Old `GET` jobs fail | The service overwrote or expired flag state too early. |
| Defensive patches break `CHECK` | The checker tests the vulnerability instead of ordinary functionality. |
| A secret appears in targets | It was returned in `State.public` rather than `State.private`. |

## Security review

Document privately before release:

- intended vulnerabilities and affected stores;
- exact public state required by an attacker;
- checker-only private state required for retrieval;
- normal functionality protected by `CHECK`;
- flag-retention behaviour; and
- every file intentionally exposed through the service image or bundle.

Keep checker implementations, exploits, and writeups private until the game
disclosure policy explicitly permits release.
