# Contributing to tai42-tools-stripe

`tai42-tools-stripe` is a **Stripe payment tools** plugin for the TAI ecosystem:
manifest-loaded Checkout Session tools that mint hosted payment links, answer a
paid ask from a Stripe webhook, reconcile lost payments from Stripe's own
session list, and mint a flexible-amount non-blocking payment link. Each tool
registers through the `tai42_app` handle from `tai42_contract.app` and is loaded
by the host from the manifest's `tools[].module` field by dynamic import. The
hard rule (the plugin rule): **it registers through `tai42-contract` and never
imports the skeleton.**

## Ground rules

- **No skeleton import — ever.** The package is contract-facing; the ban is
  enforced by ruff (`flake8-tidy-imports`), so a stray import fails lint:
  ```bash
  grep -rn "tai42_skeleton" src/   # must be empty
  ```
- **Loud errors.** No swallowed exceptions, silent fallbacks, or silent
  truncation. A bad argument, a missing or empty secret, a non-2xx from Stripe
  or the callback door, an exhausted retry ladder, or a page-ceiling overrun all
  raise loudly rather than returning a clean-looking partial result.
- **The secrets never leave the environment.** `STRIPE_SECRET_KEY` and
  `TAI_BRIDGE_CALLBACK_SECRET` are read from settings at call time and never
  carried in a tool argument, a fixture, or a test. No key value is ever echoed
  into a raised message or a log line.
- **Direct REST, one pinned version.** All Stripe traffic is direct REST through
  tai42-kit's curl client (no `stripe` SDK); the `Stripe-Version` header is
  pinned in code. Redirects are off on every call so a 302 can never replay the
  `Authorization` or `X-TAI-Bridge-Secret` header to another host.
- **The callback door is origin-pinned.** Every answer POST target is validated
  by exact-origin match against `INTERACTIONS_PUBLIC_BASE_URL` before it is
  dialed; a foreign host, scheme, port, or path-traversal is refused.
- **Livemode is asserted against the key.** A session's `livemode` must agree
  with the configured key's mode (`sk_live_`/`rk_live_` → live,
  `sk_test_`/`rk_test_` → test); a mismatch raises.
- **The bridge and reconcile tools are not agent/user tools.** They hold the
  bridge secret and answer payment asks. `create_stripe_payment_link` is
  deployment-fenced to deterministic flow callers only. Never expose any of them
  on an agent or user toolset.
- **Typed package** (`py.typed`). Pyright runs clean.

## Layout

- `src/tai42_tools_stripe/tools/` — the registered tool entrypoints
  (`create_stripe_checkout`, `confirm_stripe_payment`,
  `reconcile_stripe_payments`, `create_stripe_payment_link`).
- `src/tai42_tools_stripe/_internal/tools/stripe_client.py` — the shared Stripe
  REST client, livemode assert, SSRF pin, answer builder, and callback-door retry.
- `tests/` mirrors `src/`.

## Naming

PyPI is a flat namespace with no owner in the path, so distributions carry the
`tai42-` prefix. GitHub repositories keep their `tai-` names, because the
`tai42ai` organisation already namespaces them. Import packages follow the
distribution.

| Surface | Form |
| --- | --- |
| Distribution — PyPI, `pip install`, dependency pins | `tai42-<name>` |
| Import package | `tai42_<name>` |
| GitHub repository | `tai-<name>` |

So a dependency is declared as `tai42-<name>` while its repository is named
`tai-<name>`, and both spellings are correct in their own context.

Some surfaces are deliberately neither, and must not be renamed: the `tai` CLI
command (`tai42` is an alias), the Prometheus metric namespace (`tai_tool_*`),
`TAI_*` environment variables, and the `tai-plugin.yml` descriptor filename.

## Dev

```bash
uv venv --python 3.13
uv pip install --no-sources --extra dev --editable .
uv run --no-sync pytest --cov --cov-report=term-missing
uv run --no-sync ruff check .
uv run --no-sync ruff format --check .
uv run --no-sync pyright
```

`make dev` installs the sibling `tai-contract` and `tai-kit` repos as editable
installs for local cross-repo development.

Before any commit, run a secret scan over `src/` and `tests/` (e.g.
`detect-secrets scan`).

## License

By contributing you agree your contributions are licensed under Apache-2.0.
