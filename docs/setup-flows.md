Provider setup flows
==================================

Music Assistant separates **interactive setup** from **runtime options**:

- **Setup flow** — everything needed to get a provider or player working: credentials, OAuth logins, pairing PINs, discovery choices. Authored as a small coroutine in `providers/<domain>/setup_flow.py`, driven step-by-step by the server, rendered by the frontend in a dialog. The values a flow collects are stored as the target's **`setup_data`**: encrypted at rest and never serialized over the API.
- **Options** — the runtime knobs (quality, behavior toggles, output settings). Declared by overriding `get_config_entries()` on the provider (or player) instance and stored as regular config values.

If your provider needs no interaction to set up (works out of the box, or only needs options), don't create a `setup_flow.py` — the add-provider UI then completes immediately. Providers that are auto-created by the server (builtin) must not have one.

## Quick start: a credential flow

```python
# providers/myprovider/setup_flow.py
from music_assistant_models.config_entries import ConfigEntry
from music_assistant_models.enums import ConfigEntryType

from music_assistant.models.setup_flow import SetupSession

_ENTRIES = [
    ConfigEntry(key="username", type=ConfigEntryType.STRING, label="Username", required=True),
    ConfigEntry(key="password", type=ConfigEntryType.SECURE_STRING, label="Password", required=True),
]


async def run_setup(session: SetupSession) -> None:
    """Run the interactive setup for this provider."""
    errors: dict[str, str] | None = None
    while True:
        values = await session.form(_ENTRIES, step_id="user", errors=errors, last_step=True)
        try:
            await _validate_login(str(values["username"]), str(values["password"]))
        except LoginFailed as err:
            # re-show the form with an error; the user can correct and retry
            errors = {"base": err.translation_key or str(err)}
            continue
        await session.finish(values)
        return
```

The engine calls `run_setup(session)` when the user starts the flow. Each `await session.form(...)` suspends the coroutine until the user submits; `session.finish(values)` persists the values as `setup_data` and creates (or reloads) the target. If loading fails, `finish` raises `SetupFlowError` — catch it and loop back to a form instead of dead-ending.

## The SetupSession API

| Call | Purpose |
|---|---|
| `await session.form(entries, step_id="user", errors=None, last_step=None, expires_in=None)` | Show a form, return the submitted values. `errors` maps entry keys (or `"base"`) to error slugs/text for a retry round. `last_step` controls the submit button label. |
| `await session.external(url, step_id="auth", expires_in=None)` | Send the user to an external URL (OAuth). Resumes when the flow's `callback_url` is hit; returns the merged GET/POST callback params. |
| `session.progress(step_id, text=None, pct=None, image=None)` | Fire-and-forget display-only progress step. Re-emit it to update text, percentage, or image in place. |
| `await session.progress_until(awaitable, step_id, text=None, image=None, expires_in=None)` | Show progress while awaiting something, with a single deadline that drives both the UI countdown and the enforcement. Raises `StepExpiredError` on the deadline. |
| `await session.finish(values)` | Persist `values` as the target's `setup_data`, create/reload the target, publish the FINISH step. Raises `SetupFlowError` when the target fails to load with these values. |
| `raise AbortFlow(reason)` | End the flow with a translated message (`setup_flow.abort.<reason>`). Use for unrecoverable situations. |

Everything the user sees is translatable — see [Translations](#translations).

### Flow context

`session.context` describes what is being (re)configured:

- `kind` — `"setup"` (new) or `"reconfigure"` (existing target)
- `reason` — `"user"`, `"auth"` (re-auth needed) or `"error"` (target failed to load)
- `instance_id` / `player_id` — the existing target, for reconfigure and player flows
- `setup_data` — the decrypted values from a previous run, for prefill
- `values` — the target's current options values

A reconfigure flow is the same coroutine: branch on `session.context.kind` to prefill entries (set each entry's `default_value` from `context.setup_data`) or to skip steps that are already satisfied.

### Multi-step flows

Just call `session.form()` (or any other step) more than once. Collect values across steps in a local dict and pass the merged result to `finish`:

```python
async def run_setup(session: SetupSession) -> None:
    collected: dict[str, ConfigValueType] = {}
    collected.update(await session.form(_SERVER_ENTRIES, step_id="server"))
    if collected.get("use_auth"):
        collected.update(await session.form(_CRED_ENTRIES, step_id="credentials"))
    await session.finish(collected)
```

Steps can expire: pass `expires_in=<seconds>` and the frontend shows a countdown while the engine enforces the deadline (`StepExpiredError` — catch it to mint a fresh code and re-show the step, as pairing and device-code flows do).

### OAuth / external login

```python
async def run_setup(session: SetupSession) -> None:
    auth_url = build_authorize_url(redirect_uri=session.callback_url)
    params = await session.external(auth_url, step_id="authenticate")
    if "code" not in params:
        raise AbortFlow("auth_failed")
    tokens = await exchange_code(str(params["code"]))
    await session.finish({"refresh_token": tokens.refresh_token})
```

The frontend renders an "Open" button (plus the destination host as a trust cue) and waits. The external party — or a small bounce page you host — must redirect/POST to `session.callback_url` (`/setup_flow/callback/<flow_id>`); query and body parameters are handed back to your coroutine.

### Progress and inline images

`ConfigEntryType.IMAGE` entries and the `image=` argument accept a data URI, which the dialog renders inline — used for QR logins and device codes:

```python
creds = await session.progress_until(
    client.poll_until_confirmed(device),
    step_id="device_login",
    text="device_login",
    image=render_code_svg(device.user_code),   # data:image/svg+xml;... URI
    expires_in=float(device.expires_in),
)
```

## Player setup flows

Players use the same session API. A player that needs interaction (pairing PIN, password) overrides `run_setup_flow` on its `Player` class and sets `needs_setup`/`setup_reason` so the UI shows a *Setup required* chip:

```python
class MyPlayer(Player):
    async def run_setup_flow(self, session: SetupSession) -> None:
        pin_entry = [ConfigEntry(key="pin", type=ConfigEntryType.STRING, label="PIN", required=True)]
        await self._start_pairing()
        values = await session.form(pin_entry, step_id="pair", expires_in=60)
        await self._finish_pairing(str(values["pin"]))
        await session.finish({"credentials": self._credentials})
```

Wrapper players (universal players, native players wrapping protocol children) need no code: the engine delegates to the child that needs setup, or shows a picker when several do. The collected values persist to the child's own config.

Set `setup_reason` to a short slug (`"pairing_required"`, `"password_required"`, `"auth_expired"`) — it resolves against the provider's translations and is shown on the chip.

## Translations

All flow-facing text lives in the provider's `strings.json`:

```json
{
  "setup_flow": {
    "user": { "title": "Connect your account", "description": "Enter your credentials." },
    "device_login": { "progress_text": "Enter the code on the device page" },
    "abort": { "auth_failed": "Authentication failed." }
  },
  "errors": {
    "invalid_credentials": "Invalid username or password."
  }
}
```

- `setup_flow.<step_id>.title` / `.description` — form and external steps
- `setup_flow.<step_id>.progress_text` — progress steps (the `text=` argument is the key)
- `setup_flow.abort.<reason>` — `AbortFlow` reasons
- `errors.<slug>` — error slugs passed via `errors=` (owner-first resolution: your provider's file wins, then common)

## Runtime options

Options are declared on the provider (or player) instance — the method takes no arguments and reads whatever it needs from `self`:

```python
class MyProvider(MusicProvider):
    async def get_config_entries(self) -> tuple[ConfigEntry, ...]:
        """Return the runtime options for this provider instance."""
        return (
            ConfigEntry(key="quality", type=ConfigEntryType.STRING, label="Quality", options=...),
        )

    async def handle_config_action(self, action: str) -> tuple[ConfigEntry, ...]:
        """Handle a one-shot action button press and re-render the entries."""
        if action == "clear_cache":
            await self._clear_cache()
            return await self.get_config_entries()
        return await super().handle_config_action(action)
```

`ConfigEntryType.ACTION` entries render as buttons; a press arrives via `config/providers/invoke_action` → `handle_config_action(action)`. Actions are one-shot side effects on saved state — anything that needs input or multiple steps belongs in the setup flow. An action that must open a browser page returns a `ConfigEntryType.URL` entry in its response; the frontend opens it once and drops it from the form.

## API commands

| Command | Purpose |
|---|---|
| `config/providers/setup {provider_domain}` | Start the setup flow for a new provider instance |
| `config/providers/reconfigure {instance_id}` | Re-run the flow for an existing instance (also used for re-auth) |
| `config/players/setup {player_id}` | Start (or re-run) a player's setup flow |
| `config/flows/submit {flow_id, values}` | Submit the active step's values |
| `config/flows/get {flow_id}` | Re-fetch the current step (reconnects) |
| `config/flows/abort {flow_id}` | Abort a running flow |
| `config/providers/get_entries {instance_id}` | Options entries for an existing instance |
| `config/players/get_entries {player_id}` | Options entries for a player |
| `config/providers/invoke_action {instance_id, action}` | Run an options action button |
| `config/players/invoke_action {player_id, action}` | Run a player options action button |

Flow steps are pushed as `SETUP_FLOW_UPDATED` events (translated per connection); sessions expire after 15 minutes of inactivity.

## Migrating older providers

If you maintain an out-of-tree provider, note what the setup-flows redesign removed:

- **`AuthenticationHelper` and the `AUTH_SESSION` event are gone.** Use `session.external()` — it provides the callback route and the browser-open UX.
- **Actions can no longer tunnel through `get_entries`.** `get_config_entries` takes no arguments; button presses arrive via `handle_config_action`. Multi-step "action flows" (intermediate `values`, `session_id` plumbing) are setup flows now.
- **Setup values moved out of the options namespace.** Credentials collected by a flow live in `setup_data` (read via `self.get_setup_value(key)`), not in config values.
