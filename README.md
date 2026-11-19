# mpgs: a test harness for Mastercard Gateway integrations

> **Experimental prototype, not an official Mastercard product and not Mastercard
> documentation.** It runs on the MTF test gateway with TEST merchant profiles only.
>
> - **Its aim:** help merchants integrate with the Mastercard Gateway faster and with
>   fewer errors.
> - **Its source:** publicly available Mastercard Gateway (MPGS) documentation.
> - **Your payment service provider comes first:** always follow your provider's
>   guidance; where it differs from this tool, your provider is right.
> - **Use at your own risk:** this is a prototype, provided as is, with no warranty or
>   support.

`mpgs` helps you, or your coding agent, build an integration with the Mastercard Gateway
and **prove it works** before go-live:

- **Integration paths in your terminal:** one buildable route per use case (for example
  "authorize and capture with 3-D Secure, card in hosted fields"), with the guides and
  API reference behind it. Search them in your own words.
- **A test harness for your own backend:** a path's core scenarios, one for each answer
  your backend must handle (approved, declined, 3-D Secure challenge failed, ...), run
  through your endpoints, then the gateway's record of each
  order is checked against the path.
- **A profile check:** what your merchant profile can do (currencies with a working
  acquirer link, tokenization, 3-D Secure), with the evidence for each finding.

**How it fits together:** you find the path for what you want to build, try it on your
profile, build your integration from it (you or your coding agent: `mpgs` does not write
it), then let `mpgs` check what you built.

## What you need

- macOS or Linux. **Windows is not supported.** [uv](https://docs.astral.sh/uv/) (or
  `pipx`) with Python 3.11 or later
- A **TEST merchant profile** on the MTF test gateway: its merchant ID (it starts with
  `TEST`) and its API password, from your payment service provider
- The release: the wheel `mpgs_cli-<version>-py3-none-any.whl`, from the
  [releases](https://github.com/marekjanos/mpgs-cli-prototype/releases). It carries the CLI
  and the documentation it serves.
- Only for the 3-D Secure challenge scenario: Google Chrome and the
  [`agent-browser`](https://www.npmjs.com/package/agent-browser) CLI

## 1. Install

```bash
uv tool install https://github.com/marekjanos/mpgs-cli-prototype/releases/download/v0.5.0/mpgs_cli-0.5.0-py3-none-any.whl
# (pipx install works the same way; or download the wheel and install the file)
mpgs --version        # the CLI version and the documentation release it carries
```

The first command unpacks the documentation into `~/.mpgs`; there is nothing else to
download. Later, `mpgs update` brings a newer `mpgs` and newer documentation.

## 2. Give it your TEST credentials

The CLI reads your merchant ID and API password, in this order, from:

1. Environment variables: `MPGS_MERCHANT_ID` and `MPGS_API_PASSWORD`
2. An `.env` file your backend already uses: `mpgs --env-file .env --env-keys MY_MID,MY_PASSWORD …`
3. Your operating system's password store (macOS Keychain, Linux Secret Service): `mpgs login TESTYOURMID` asks for the password once and stores
   it; then set only `MPGS_MERCHANT_ID`. On macOS, the first read may show a dialog asking
   whether Python may use the item: choose "Always Allow". On a Linux machine with no
   password store (a server, a container), use 1 or 2 instead.

A merchant ID without the `TEST` prefix is refused before any call, and the CLI calls
`https://mtf.gateway.mastercard.com` only. The password never goes into a log or file.

Check it works, and see what your profile can do:

```bash
export MPGS_MERCHANT_ID=TESTYOURMID
mpgs doctor
```

## 3. Find your path

Say what you want to build, in your own words:

```bash
mpgs path find "authorize and capture with 3ds, using hosted session"
mpgs path find "recurring payments with a stored card"
mpgs path find 3ds
mpgs path find                   # what you can search by, with counts
```

The first line says how your words were read (`Understood: Authorize + Capture ·
3-D Secure · Hosted Session`). Each result starts with the **path ID**, the name every
other command takes, then its title and what it does. `mpgs path list` shows every path.

## 4. Read it

```bash
mpgs path get <path ID>               # an overview: the steps, where each runs, what is next
mpgs path get <path ID> --full        # the whole path
mpgs path get <path ID> --out path.md # the whole path, as a file for your agent
```

A path gives you, for each step: which call it is (or what runs in the payer's browser or
your app), the fields the request needs and why, an example request, what to read from
the response and what to do with it. Then the test cards for every outcome, and a
server-only way to test it. Links in a path open with `mpgs docs get <link>`.

The documentation behind the paths (the guides, the API reference, the JavaScript SDK
reference) is searchable too:

```bash
mpgs docs                                          # what is in it, how to search it
mpgs docs find "3ds challenge"                     # pages, best match first
mpgs docs grep "storedOnFile"                      # exact text, by page
mpgs docs field order.amount --operation transaction/pay   # one API field
mpgs docs get guides/supportedFeatures/testAndGoLive       # a page
```

## 5. Try the path on your profile, before you write code

```bash
mpgs path run <path ID>
```

The CLI runs this path with your TEST merchant ID on the MTF test gateway: it sends the
path's own example requests, so there is nothing to write or set up, just run it. It
tells you three things before you build anything:

- **Whether your profile can do this path at all** (acquirer link, 3-D Secure, tokens).
  A failure here is your profile or your PSP's configuration, not your code:
  `mpgs doctor` and the explanation under the failed step say which.
- **What every request and response looks like on your profile.** The output names each
  order; `mpgs logs --order <order ID> --full` shows every request and response.
- **The reference your integration is compared with:** the gateway's record of each
  transaction is saved, and step 7 checks your orders against it field by field. It
  belongs to this version of the path: after a documentation update, run it again.

## 6. Build your integration

`mpgs` does not write your integration: you, or your coding agent, build it from the
path. Each step that runs on your server becomes one call in your backend, sending what
the step's example request sends; each step in the payer's browser or your app becomes a
change there, with the code the path gives.

For a coding agent, the CLI prints the instruction to paste:

```bash
mpgs path prompt <path ID>
```

It tells the agent to read the path and build the integration from it (the server calls,
and your checkout page or app wired to them, so a payer can pay), look things up in the
documentation, use your TEST merchant ID on the test gateway without printing the
password, and prove the result with `mpgs check`.

Before step 7, your backend must be able to take a payment on the test gateway with your
TEST merchant ID: it creates the order, and you know its order ID.

## 7. Check what you built

Check one order your code created:

```bash
mpgs check <path ID> --order <your order ID>
```

It reads the gateway's record of the order and checks it against the path: each step's
success rules, and every field the path sends, against the reference from step 5. Then
let the CLI run the path's core scenarios through your backend:

```bash
mpgs adapter init <path ID>     # writes mpgs.toml and mpgs-adapter.md
# map the path's steps onto your own endpoints in mpgs.toml (mpgs-adapter.md explains how)
mpgs adapter check <path ID>    # backend reachable, {env:…} set, nothing paid; names unmapped steps
mpgs test <path ID>             # the core scenarios: approve, decline, …
```

`mpgs.toml` says how your backend is already called: which endpoint performs which path
step, where the card, order ID and amount go (placeholders such as `{card}`, `{orderId}`),
where your response names the order, and where it carries the 3-D Secure redirect. You
write no test scripts. A step can also be a command (`exec = "…"`) that prints one JSON
object.

Every check says what it could not see ("blind"): calls the gateway rejected before they
created a transaction, and steps that run in the payer's browser.

Each step's command ends with a `Next:` line: what to run after it.

## Command reference

Every command takes `--json` (before the command: `mpgs --json path get <id>`), and
`mpgs <command> --help` shows its options with examples. Exit codes: `0` pass, `1` a test
or check failed, `2` usage, `3` configuration (credentials, documentation, `mpgs.toml`),
`4` the gateway or its firewall refused, `5` not available in the prototype.

**Coding agents:** `mpgs help agents` is their guide (the rules, the route, the JSON
shapes); `mpgs path prompt <id>` points them to it.

Which commands call the test gateway:

| Command | Calls the gateway | Creates orders |
|---|---|---|
| `doctor`, `path run`, `test` | yes | yes |
| `check` | yes, reads only | no |
| `path find / list / get / prompt`, `docs`, `logs`, `evidence`, `help` | no | no |

### `mpgs path find "<words>"` · `path list`

Searches the paths in your own words: integration method (direct API, hosted session,
hosted checkout), flow (pay, authorize and capture), capabilities (3-D Secure, token,
recurring, Apple Pay, Google Pay, Click to Pay, network token, add a card, mobile app,
…) and add-ons (aggregator, airline, …). With no words, it shows what you can search by.

```
$ mpgs path find "pay with apple pay"
Understood: Pay · Apple Pay
3 paths

Direct API (3)
  pay-agreement-apple-pay-merchant-decrypted-direct
    Recurring Apple Pay (CIT, then MITs): you decrypt and store the device card number
    …
  pay-apple-pay-direct
    Pay with Apple Pay, decrypted by the gateway
    Take an Apple Pay payment in one Pay: send Apple's encrypted payment token as it is,
    and the gateway decrypts it with the certificate you uploaded.
  …

Next: mpgs path get pay-agreement-apple-pay-merchant-decrypted-direct
```

`--tag` and `--method` filter exactly (for scripts and agents).

### `mpgs path get <id>`

An overview: the steps, where each runs (your server, the payer's browser, your app)
and which call it makes, the add-ons, the API pages behind the steps, and what to run
next. `--full` shows the whole path, `--out FILE` writes it, `--format json` as JSON.

```
$ mpgs path get pay-3ds-session-direct
pay-3ds-session-direct
Pay with 3-D Secure
Integration method: Hosted Session
…
Steps
   1. Create Session                     your server  POST session/create-session
   3. Collect the card in the hosted fields   payer's browser
  10. Pay                                your server  PUT transaction/pay
```

### `mpgs path run <id>`

The CLI runs the path with your TEST merchant ID on the MTF test gateway, using the
path's own example requests, and saves the gateway's record as the reference for
`check`. `--scenario challenge` runs a 3-D Secure challenge card through the gateway's
test page (needs Google Chrome and `agent-browser`); `--addon <id>` adds an add-on the
path lists (repeat it for several), and saves the reference for the path with them:
give `check`, `adapter init` and `test` the same add-ons.

```
$ mpgs path run pay-apple-pay-merchant-decrypted-direct
Ran pay-apple-pay-merchant-decrypted-direct with TESTYOURMID on the MTF test gateway.

PASS  1. Pay   order mpgs-ref-1794565573 · transaction pay-1 · APPROVED

Passed: your profile can do this path. …
See every request and response: mpgs logs --order mpgs-ref-1794565573 --full
```

### `mpgs path prompt <id>`

Prints the instruction to paste into your coding agent: read the path and the CLI's
agent guide, build the integration (the server calls, and your checkout page or app wired
to them, so a payer can pay), use your TEST merchant ID without printing the password,
prove it with `mpgs check`.

### `mpgs docs`

The documentation behind the paths: `find "<words>"` (pages, best match first),
`grep "<exact text>"` (by page, the pages with most matches first), `field <field>
--operation <group/operation>` (one API field's type, existence and description; without
`--operation`, every operation that has it), `get <page>` (a page; links copied from a
path work as they are). With no arguments, it shows what is there.

### `mpgs login <TESTMID>`

Asks for the API password once and stores it in your OS password store.

### `mpgs doctor`

What your profile can do: one Payment Options Inquiry (its configuration), then a test
transaction per check (a Verify per currency, a token, a 3-D Secure authentication).
`--currency X` checks other currencies, `--all-currencies` every configured one.

```
$ mpgs doctor
From your profile's configuration (Payment Options Inquiry, 1 call)
  payment operations        AUTHORIZE, PURCHASE
  currencies (configured)   AED, AUD, … USD
    Configured is not the same as usable: a currency works only where your
    acquirer link is configured for it.
  …
Checked with a test transaction
  acquirer link USD         yes            Verify approved
  tokenization              yes            Create Token: status VALID
  3-D Secure                3DS2 2.2.0     Initiate Authentication
```

### `mpgs check <id> --order <order ID>`

Reads the gateway's record of each order you name (no payment is made) and checks it:
the path's success rules for each transaction, and every field the path sends against
the reference from `path run`. Your payer's own data (name, billing and shipping
address, browser details) must be there, but its values are theirs and are not compared;
so are the terms of an agreement (`agreement.type` UNSCHEDULED where the path's sample is
RECURRING, when your amounts vary). Each transaction is checked against its own step:
on a path that makes two payments (the payer's, then a later one you take on your own),
the first payment is checked against step 1 and the second against step 2, so name every
order your code made for the path. Exit `1` when a check fails.

With `--addon <id>` (repeatable; the add-ons you ran `path run --addon` with), the add-on's
fields are checked too: one with a fixed value (account funding's `order.purchaseType`)
must match; the rest are your data (a booking reference, a passenger) and must be there,
whatever their values. When none of an add-on's fields is on the order, the check fails:
your integration does not send it. A step an add-on adds (Payment Options Inquiry) is
listed as blind when it is not a transaction on the order.

```
$ mpgs check pay-3ds-token-direct --order my-order-123
PASS  step 4 AUTHENTICATION response.gatewayRecommendation expected PROCEED got PROCEED
PASS  step 7 PAYMENT result expected SUCCESS got SUCCESS
PASS  step 7 PAYMENT response.gatewayCode expected APPROVED got APPROVED

Blind (25): what the gateway's record of the order cannot show (not a pass, not a fail)
  - calls the gateway rejected before they created a transaction
  - steps 3, 5 run in the payer's browser
…
Passed.
```

### `mpgs adapter init <id>` · `adapter check`

`init` writes `mpgs.toml` (which of your endpoints performs which path step) and
`mpgs-adapter.md` (how to fill it). `init <id> --addon <id>` adds the add-on's steps and
records the add-ons in `mpgs.toml` (`addons = ["airline"]`), so `adapter check` and
`test` use them without the flag. `check` sees that `base_url` answers, each `exec` command
exists and each `{env:NAME}` is set in your shell (unset, the CLI would send it empty);
it calls no endpoint and pays nothing: `mpgs test` is the first to call them.

On a Hosted Session path, the payer types the card into the gateway's fields. `mpgs test`
does that for them in one of two ways: it creates a session with the test card and
passes it to your backend as `{sessionId}`, or, when your backend creates the session
itself (to set the order's amount), you name that session with `session_id = "$.…"` on
the step that returns it, and `mpgs test` types into it (the card, or only the security
code when the card is stored) before the next step.

Each `[step.<name>]` is named after the path step it performs (as `init` writes them), or
says so with `path_step = 2` (a number or ID, or a list when one endpoint does several).
A scenario ends where the path says it does: a decline at the sign-up stops there, and
the steps tied to a later path step are not called. Some start at a later step (a
declined merchant-initiated payment, a declined card change): the steps before it run with
the approve card, and the scenario fails when no step of `mpgs.toml` performs the step it
starts at. `adapter check <id>` names the path steps `mpgs.toml` does not map.

### `mpgs test <id>`

Runs each scenario through your backend as `mpgs.toml` maps it, creating orders on the
test gateway, then checks each order. The scenarios are the path's own, each verified on
the MTF test gateway: `mpgs path get <id>` lists them with the card, expiry and result.
`approve` is checked as `check` does, and fails when a step of your backend answered an
HTTP error (the order's record cannot show every step, such as a token update); a decline scenario passes only when the gateway
gave its own answer (`decline` DECLINED, `expired-card` EXPIRED_CARD, `timed-out`
TIMED_OUT and so on) and, with `declined` in `mpgs.toml`, your backend reported it (the
first step that reports a decline ends the scenario: later steps are not called); a
scenario that must stop the payment (a 3-D Secure challenge the payer fails) passes when
nothing was charged. A scenario marked `*` works on the test gateway but is not in the
integration guides. `challenge` (a 3-D Secure challenge card) is added on paths with
3-D Secure. A scenario that starts at a later step (`mpgs path get` shows "from step
2") runs the steps before it with the approve card. Scenarios that send a field the path
does not are not run here. By default `mpgs test` runs the path's core scenarios:
one for each way your backend must handle an answer (approved, declined, an expired card,
3-D Secure rejected, and so on). `--all` adds the others, which take the same branch as a
core one with another card or answer (timed out and acquirer error are handled as an
expired card is). `--scenario` runs one. The add-ons are those in `mpgs.toml`, or
`--addon <id>` (repeatable) in their place: each scenario then runs the path with them,
and `approve` checks their fields as `check --addon` does.

Hosted Checkout paths are not tested yet: `mpgs test` cannot pay on the gateway's
payment page, so it answers exit `5` with that reason. Pay on your checkout in a browser
with the path's test cards (`mpgs path get <id>` lists them).

```
$ mpgs test pay-3ds-token-direct
PASS approve                order mpgs-approve-1794…
PASS decline                order mpgs-decline-1794…
PASS expired-card           order mpgs-expired-card-1794…
…
PASS challenge              order mpgs-challenge-1794…

Every scenario passed: your integration does what the path says on the test gateway.
```

### `mpgs logs` · `mpgs evidence`

`logs` lists the calls the CLI made (`--order <id>`, `--last n`), with gateway errors
explained; `--full` adds each request and response, card data masked. `evidence` adds
up your runs (`--since YYYY-MM-DD`).

### `mpgs update` · `mpgs --version`

`update` first looks for a newer `mpgs` in the
[releases](https://github.com/marekjanos/mpgs-cli-prototype/releases). When there is one,
it downloads the wheel, checks it against its published checksum and size, and installs
it with the tool you installed this one with (`uv tool install --force`, or `pipx install
--force`), on the same Python. The new `mpgs` then fetches the documentation (the paths,
guides and API reference), checked the same way, so documentation that needs the new
version installs in the same run. `--no-cli` updates only the documentation and keeps your
`mpgs`. An `mpgs` not installed with uv or pipx (from source, say) is not replaced:
`update` says a newer one is out, and you install it as in step 1. `--version` names the
CLI and the documentation release.

A path that needs a newer `mpgs` than yours is still listed and readable, but `path run`
and `test` refuse it: "This path needs a newer mpgs: run `mpgs update`." (exit `5`).

## What it stores

Everything lives in `~/.mpgs` (or `$MPGS_HOME`): the documentation, `log.jsonl` (every
call the CLI made, card numbers and security codes masked), `runs/` (a summary per run,
no card data) and `reference/`. Delete it at any time; the next command unpacks the
documentation again.

The CLI sends requests to the MTF test gateway, in `mpgs test` to your own backend, and
in `mpgs update` to its GitHub releases. No telemetry. Only the test cards from the paths' test tables are used.

It checks HTTPS certificates against your operating system's certificate store, as your
browser does, so it works behind a company proxy that inspects HTTPS wherever your browser
works.

## Troubleshooting

| You see | Meaning |
|---|---|
| `… is not a TEST merchant profile` | Use your TEST merchant ID |
| `Invalid credentials` | Wrong API password or merchant ID |
| `No valid Merchant Acquirer Relationship available` | Your profile has no acquirer for that currency and card: `mpgs doctor --all-currencies` shows which currencies work; ask your PSP |
| HTTP 400 "Application Security Module Event Detected" (exit 4) | The gateway's firewall: too many calls too fast. Wait about 10 minutes; run at most two clients at once |
| `no OS password store available` | Use `MPGS_API_PASSWORD` or `--env-file` instead of `mpgs login` |
| `… does not answer: is the backend running?` | `mpgs adapter check` cannot reach `base_url` in `mpgs.toml` |
| `{env:NAME} is not set in this shell` | `mpgs.toml` uses a variable your shell lacks: export it before `mpgs test`, or the CLI sends it empty |
| `the gateway asked for a 3-D Secure challenge … names no answer` (`mpgs test`) | The scenario's card is challenged on this path but the scenario has no answer for it: the path's test data, not your backend. `mpgs update` may bring a fix |
| `no documentation installed` | Reinstall the CLI (`uv tool install --force ./mpgs_cli-…whl`) |
| `certificate is not trusted` or `certificate verify failed` | Your network inspects HTTPS with a root certificate your operating system does not trust: ask your IT team to install it system-wide |
| `the download of … stopped after 0 of … bytes` (`mpgs update`) | A network filter, such as a company proxy, blocked the file. Your documentation is unchanged: try another network |
| `invalid peer certificate: UnknownIssuer` (`uv tool install`) | Your network inspects HTTPS: add `--system-certs` (older uv: `--native-tls`) so uv uses your operating system's certificates |
| `uv could not install mpgs …` (`mpgs update`, exit `3`) | Your `mpgs` and documentation are unchanged: install the release as in step 1, or `mpgs update --no-cli` for the documentation only |

## Uninstall

```bash
uv tool uninstall mpgs-cli      # or: pipx uninstall mpgs-cli
rm -rf ~/.mpgs                  # optional: documentation, logs, run summaries
```
