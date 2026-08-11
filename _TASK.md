# Bootstrap an admin and pin settings from configuration

## Background

A fresh Tourney instance can only be initialised through the web `/setup`
flow: a person opens the site, creates the admin account, and fills in
settings. That works when someone deploys one instance by hand. Operators who deploy
instances automatically (containers, CI, scripted deploys) have no way to
create the admin or apply settings without that manual step. The goal
of this task is to make a fresh instance fully usable from configuration
alone.

Two configuration systems already exist, and you will build on both:

| | File / environment config | Runtime config |
|---|---|---|
| Storage | `config.ini`, overridable by environment variables | the `Configs` table |
| Read with | `get_app_config("KEY")` | `get_config("key", default)` |
| Written by | the operator, at deploy time | the admin panel, at runtime |

## What to build

**1. A preset admin.** Config that lets an operator declare admin credentials
in advance, so the instance creates that admin automatically instead of
requiring the web setup:

- New config keys `PRESET_ADMIN_NAME`, `PRESET_ADMIN_EMAIL`,
  `PRESET_ADMIN_PASSWORD`, and an optional `PRESET_ADMIN_TOKEN` (an API
  token), exposed through `config.ini` like the existing keys.
- When those credentials are set and a **login** arrives whose identifier
  matches the preset name or email and whose password matches, the instance
  creates a verified admin account (once) and logs it in.
- A request that presents `PRESET_ADMIN_TOKEN` as its API token authenticates
  as that same preset admin, creating the account if it does not exist yet.
- **Security rule: never promote an existing non-admin account.** If a user
  matching the preset identity already exists but is not an admin, do not
  turn them into one; refuse instead.

**2. Preset configuration values.** A `PRESET_CONFIGS` config key holding a
JSON object of runtime config key/value pairs. These values take precedence
over what is stored in the database when `get_config` resolves a key, so an
operator can pin settings from the deployment.

**3. Built-in defaults for core settings**, so the app still works when web
setup was skipped and `get_config` runs against an empty `Configs` table.
Cover exactly these keys (a database value must still override them):

| key | default |
|---|---|
| `ctf_name` | `"Tourney"` |
| `user_mode` | `"users"` |
| `ctf_theme` | the `DEFAULT_THEME` constant |
| `challenge_visibility` | `"private"` |
| `registration_visibility` | `"public"` |
| `score_visibility` | `"public"` |
| `account_visibility` | `"public"` |

## Current behaviour

To see it in the running app, follow "Running the app locally" in
`_ORIENTATION.md`, then:

1. Start a clean instance (if you have run the app before, first delete both
   `Tourney/tourney.db` and the `.data/` directory). Visit any page: every
   route, `/login` and the whole API included, redirects to `/setup`. A
   fresh instance cannot be used, or even logged into, until a person
   completes the wizard.
2. Open the wizard's tabs before submitting. The ones this task builds on:
   General (event name), Mode, Settings (the visibility levels),
   Administration (admin name, email, password), and Style. Part 1 of
   "What to build" makes the
   Administration step unnecessary; part 3 provides defaults for the core
   settings. Whether setup has been completed is
   itself recorded as a runtime config value, like the rest of the wizard's
   answers.
3. Finish the wizard. You are logged in as the admin; `/admin/config` is
   where runtime config is edited today. `PRESET_CONFIGS` pins the same
   kind of keys from the deployment instead.
4. Log out and try `/login`. A wrong password re-renders the login page
   (HTTP 200) with "Your username or password is incorrect"; the right one
   answers with a redirect (HTTP 302). The preset login reuses this exact
   contract, including rendering the no-promotion message on the login
   page.
5. In `/settings`, open "Access Tokens" and generate a token. Send it as
   `Authorization: Token <value>` with a JSON content type (the header is
   ignored on non-JSON requests) to an admin-only endpoint such as
   `GET /api/v1/challenges/types`: 200 with a valid token, 401 or 403
   without. `PRESET_ADMIN_TOKEN` uses this same mechanism.

## Getting started

- Trace one existing key through each configuration system: how a
  `config.ini` value ends up readable with `get_app_config`, and what
  `get_config` does today to resolve a runtime key. The Configuration section
  of `_ORIENTATION.md` describes both systems.
- Find where a login's password is checked, and where an API request's
  `Authorization` token is resolved to a user. The preset admin hooks into
  both places.
- `tests/test_config.py` shows how config behaviour is exercised from tests,
  including how file-config values are simulated in a test app.

## Acceptance criteria

The names and behaviour below are required exactly.

**Preset admin**

- A function `generate_preset_admin()` in `Tourney/utils/security/auth.py`
  that returns the admin `Users` object, or `None` when it cannot produce
  one (no preset configured, or the no-promotion case). This exact module
  path is required.
- It is idempotent: called again for an already-created preset admin, it
  returns the same row and does not create a second admin.
- Blank counts as unconfigured: the preset login and token paths are active
  only when `PRESET_ADMIN_NAME`, `PRESET_ADMIN_EMAIL`, and
  `PRESET_ADMIN_PASSWORD` are all non-empty. In particular, an empty preset
  password must never match an empty submitted password, and an empty
  `PRESET_ADMIN_TOKEN` disables the token path entirely; it must never match
  any presented token.
- The admin is created only by the login and token paths, never at startup:
  an instance that has served no matching login or token request has no
  preset admin row in `Users`.
- A created preset admin has `type == "admin"` and `verified is True`, and
  its stored password verifies against `PRESET_ADMIN_PASSWORD`.
- Logging in (`POST /login`) with the preset name or email and the preset
  password succeeds with a redirect (HTTP 302) to the challenges listing,
  the same destination as a normal login, creating the admin on first use. A
  preset login only reaches admin creation after the password has matched
  `PRESET_ADMIN_PASSWORD`; a non-matching password falls through to the
  normal login flow and its normal error message, whether or not the preset
  admin exists yet, and creates nothing.
- A request bearing a valid `PRESET_ADMIN_TOKEN` reaches admin-only API
  endpoints (HTTP 200), creating the admin if needed. Without a valid token,
  those endpoints return 401 or 403.
- When a preset admin cannot be created (the no-promotion case), surface
  exactly this message:
  `Preset admin user could not be created. Please contact an administrator`.
  The login attempt that hits this case renders the message in an HTTP 200
  response; it is not a redirect.

**Config resolution**

- `get_config(key, default)` resolves in this order: the `PRESET_CONFIGS`
  value, then the database value, then the caller's `default` argument if one
  was passed, then the built-in defaults, then `None`.
- Precedence is by key presence, not truthiness: a preset pinned to `False`,
  `0`, or `""` still wins over the database.
- A database value overrides a built-in default.
- **SAFE_MODE:** in safe mode the instance must not load `PRESET_CONFIGS`
  from `config.ini` (do not parse untrusted file config in safe mode). Apply
  that gate where the config file is loaded, not inside `get_config`:
  `get_config` always honours a `PRESET_CONFIGS` mapping that is already
  present on the app config. A `PRESET_CONFIGS` value that is already a
  mapping rather than a JSON string did not come from file parsing; leave it
  intact even in safe mode.

**General**

- The existing test suite still passes, with one deliberate exception.
  Implementing the built-in defaults changes the expectation of
  `test_api_config_delete_admin` in `tests/api/v1/test_config.py`: it deletes
  `ctf_name` and asserts `get_config("ctf_name") is None`, which the new
  `ctf_name` default makes impossible. Update that one test (for example, to
  exercise deletion through a key that has no built-in default) and note the
  change in `_THINKING.md`. Every other existing test must pass unmodified.

---

See [README.md](./README.md) for how to work this task: the process, the no-AI rule, keeping _THINKING.md, and how your work is evaluated.
