# Implementation plan: USD and the dynamic top 200

Status: proposed implementation. The public repository currently contains documentation and directory scaffolding only. Commands, source blocks, tests, and expected outputs below are instructions for future implementation, not executed results.

This revision replaces the earlier EUR / BTC-ETH-SOL design. Do not implement the old fixed-three-asset modules. All prices, portfolio values, trade prices, and fees use USD.

## Goal

Build an ESP32-S3 voice device that displays USD prices for the dynamic top 200 cryptocurrencies, visualizes a recorded portfolio, and documents completed purchases and sales after device confirmation.

## Current context and assumptions

### Confirmed requirements

- USD, not EUR.
- Support the first 200 cryptocurrencies, interpreted as the provider's top 200 by market capitalization.
- Include Bitcoin when present and show it first in the market browser, without falsifying its provider rank.
- A public repository under `timomoebes`: `hermes-crypto-gadget`.
- Initial publication may contain README, project folders, and technical planning.
- Display, microphone, voice interaction, portfolio visualization, and a purchase/sale journal.
- No custom enclosure in the first milestone.

### Proposed defaults and open decisions

- M5Stack CoreS3 K128, not CoreS3 SE. Keep the manufacturer's housing; do not design a printed enclosure yet.
- Separate Linux host for Hermes, STT/TTS, market data, and SQLite.
- CoinGecko `coins/markets`, five-minute refresh, no always-on microphone.
- Include stablecoins and wrapped assets if they belong to the provider's top 200. Do not silently filter categories.
- The proposed first version has no private keys or trading. The meaning of a real wallet remains unconfirmed. An external hardware-wallet signer needs a separate security plan.
- No portfolio or market implementation exists in the current scaffold.
- Public source licensing has not been chosen. Do not assume a public repository means permissive licensing.

### Working layout

All paths below are relative to the project repository:

```text
README.md
.gitignore
docs/architecture.md
docs/security.md
docs/implementation-plan.md
src/hermes_crypto_gadget/
integrations/hermes/
firmware/
tests/
.deps/                         # ignored upstream checkouts
runtime/                       # ignored personal configuration and data
```

The private agent-workspace plan is a local mirror, not another source of public credentials or business context.

### Upstream starting points

- Gadget SDK: `Adolanium/hermes-gadget-sdk`, commit `ffb1a43c02b9cdb4ad87c7132d367cccd5177dae`.
- Hermes: `NousResearch/hermes-agent`, compatibility-test commit `2eceb0275501b09febdec88d08b80d1fff730cf8`.
- SDK reference paths: `plugin/adapter.py`, `plugin/hub.py`, `plugin/tools.py`, `plugin/__init__.py`, `firmware/core/src/app.cpp`, `firmware/core/include/hg/app.hpp`, `firmware/tests/test_app.cpp`, `tests/test_gateway_e2e.py`, `tests/test_adapter_hermes.py`, `tests/fakes/fake_openai.py`, `docs/supported-hardware.md`, `docs/hardware-validation.md`, `firmware/esp32/platformio.ini`.
- Official Hermes documentation: https://hermes-agent.nousresearch.com/docs/
- CoinGecko endpoint documentation: https://docs.coingecko.com/demo/reference/coins-markets
- Hardware documentation: https://docs.m5stack.com/en/core/CoreS3

The selected Gadget SDK is a community project. Existing source and a successful build do not establish physical hardware reliability.

## Architecture / proposed approach

Keep the 200-coin catalog, retained coin-ID registry, USD journal, and polling service on the host, and send only the selected display page to the ESP32 over authenticated WSS. Reuse the Gadget SDK's pairing and voice path, adding device-scoped read/view/draft tools and a deterministic, non-modal background display. Retain holdings when rankings change, price out-of-catalog held assets in one additional bundled request, and require a current-session device confirmation before writing a journal event.

## Required data contracts

- Asset identity: exact provider `id`. Never use a symbol as a primary key and never silently repair a provider ID.
- Active catalog: exactly 200 distinct IDs. Replace atomically only after complete validation.
- Retained registry: all previously accepted IDs and their latest metadata. Rank membership changes do not delete registry or journal rows.
- Quote: `asset_id`, `price_usd` as a decimal string or null, `updated_at` as a UTC timestamp or null, `status` as `current`, `stale`, `unavailable`, or `error`.
- Journal event: `id`, `kind` (`OPEN`, `BUY`, `SELL`), `asset_id`, `quantity`, `price_usd`, `fee_usd`, `occurred_at`, `confirmed_device_id`, and schema version.
- `OPEN` has no assumed acquisition price. `BUY`/`SELL` require the actual completed trade's USD unit price and explicitly confirmed USD fee.
- No float arithmetic for accounting. Store exact decimal strings and use a bounded high-precision decimal context.
- The database declares `currency=USD`. Reject an EUR database rather than relabeling its contents.
- A missing held-asset quote makes the complete portfolio total unknown. An available subtotal must be explicitly called a subtotal.
- Initially cap the recorded portfolio at 200 distinct retained IDs. This bounds an out-of-catalog quote request below the provider's documented 250-ID maximum.
- Unknown catalog symbols and ambiguous ticker matches require clarification. No model-generated coin ID can bypass the retained registry.
- Journal corrections/voids are a separate tested workflow. Until that exists, the prototype journal is not authoritative bookkeeping.

## Step-by-step tasks

Each numbered substep is a 2–5-minute focused action. Builds, dependency downloads, API setup, and physical soak tests take additional waiting time. Do not pretend an integration workstream is one five-minute task.

### 1. Establish the source and test environment

**1.1** Verify the approved root and repository, without changing another repository:

```bash
git remote -v
git status --short
git branch --show-current
```

Expected after the initial publication: `origin` is `https://github.com/timomoebes/hermes-crypto-gadget.git`, branch `main`, and an empty status. Stop if there is unrelated uncommitted work.

**1.2** Create `pyproject.toml` during implementation, not in the documentation scaffold:

```toml
[build-system]
requires = ["setuptools>=75"]
build-backend = "setuptools.build_meta"

[project]
name = "hermes-crypto-gadget"
version = "0.1.0"
description = "Host-side USD market and portfolio modules for an ESP32 Hermes Gadget"
requires-python = ">=3.11"
dependencies = []

[project.optional-dependencies]
dev = ["pytest>=8,<9"]

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
pythonpath = ["src", "."]
testpaths = ["tests"]
```

**1.3** Create `src/hermes_crypto_gadget/__init__.py`, `tests/__init__.py`, and `tests/fixtures/__init__.py`, each with only a package comment. Then install the separate project environment:

```bash
uv sync --extra dev --python 3.11
uv run python --version
```

Expected: successful sync and a Python 3.11 version. Commit the actual `uv.lock`; do not invent resolved dependency versions.

**1.4** Commit only the environment files after reviewing the exact diff:

```bash
git diff --check
git add pyproject.toml uv.lock src/hermes_crypto_gadget/__init__.py tests/__init__.py tests/fixtures/__init__.py
git commit -m 'chore: establish isolated host test environment'
```

### 2. Validate the entire 200-coin USD catalog, test first

**2.1** Create the explicitly synthetic fixture `tests/fixtures/market_rows.py`:

```python
from decimal import Decimal


def synthetic_market_rows(count=200):
    return [
        {
            "id": "bitcoin" if rank == 1 else f"synthetic-coin-{rank:03d}",
            "symbol": "btc" if rank == 1 else f"c{rank}",
            "name": "Synthetic Bitcoin" if rank == 1 else f"Synthetic coin {rank}",
            "market_cap_rank": rank,
            "current_price": Decimal("100.125"),
            "price_change_percentage_24h": None,
            "last_updated": "2026-10-01T12:00:00Z",
        }
        for rank in range(1, count + 1)
    ]
```

These are fabricated test fixtures, not observed prices or a real top-200 list. Never publish them as live data.

**2.2** Create `tests/test_market.py`:

```python
from decimal import Decimal
import pytest
from hermes_crypto_gadget.market import MARKET_URL, parse_catalog
from tests.fixtures.market_rows import synthetic_market_rows


def test_url_requests_one_usd_top_200_page():
    assert "vs_currency=usd" in MARKET_URL
    assert "order=market_cap_desc" in MARKET_URL
    assert "per_page=200" in MARKET_URL
    assert "page=1" in MARKET_URL


def test_accepts_exactly_200_and_preserves_provider_ids():
    catalog = parse_catalog(synthetic_market_rows())
    assert len(catalog) == 200
    assert len({coin["asset_id"] for coin in catalog}) == 200
    assert catalog[0]["asset_id"] == "bitcoin"
    assert catalog[-1]["asset_id"] == "synthetic-coin-200"
    assert catalog[0]["price_usd"] == "100.125"


@pytest.mark.parametrize("count", [0, 199, 201])
def test_rejects_incomplete_or_oversized_catalog(count):
    with pytest.raises(ValueError, match="200"):
        parse_catalog(synthetic_market_rows(count))


def test_rejects_duplicate_ids_but_not_duplicate_symbols():
    rows = synthetic_market_rows()
    rows[1]["symbol"] = rows[0]["symbol"]
    assert len(parse_catalog(rows)) == 200
    rows[1]["id"] = rows[0]["id"]
    with pytest.raises(ValueError, match="duplicate"):
        parse_catalog(rows)


@pytest.mark.parametrize("value", [True, -1, "NaN", Decimal("Infinity"), Decimal("1e-999999")])
def test_rejects_invalid_or_unbounded_prices(value):
    rows = synthetic_market_rows()
    rows[0]["current_price"] = value
    with pytest.raises(ValueError):
        parse_catalog(rows)


def test_null_price_and_timestamp_remain_unknown():
    rows = synthetic_market_rows()
    rows[0]["current_price"] = None
    rows[0]["last_updated"] = None
    coin = parse_catalog(rows)[0]
    assert coin["price_usd"] is None
    assert coin["updated_at"] is None
```

**2.3 RED:**

```bash
uv run pytest tests/test_market.py -q
```

Expected before implementation: collection failure because `hermes_crypto_gadget.market` does not exist. The invalid-price and duplicate-ID tests must also demonstrate behavioral failures before those validators are added.

**2.4 GREEN:** Create `src/hermes_crypto_gadget/market.py`:

```python
from datetime import datetime, timezone
from decimal import Decimal, InvalidOperation
import json
import urllib.request

MARKET_URL = (
    "https://api.coingecko.com/api/v3/coins/markets"
    "?vs_currency=usd&order=market_cap_desc&per_page=200&page=1"
    "&sparkline=false&price_change_percentage=24h"
)
MAX_BYTES = 512 * 1024


def exact_decimal(value, *, nonnegative=False):
    if isinstance(value, bool) or not isinstance(value, (int, Decimal)):
        raise ValueError("expected a JSON number")
    try:
        number = Decimal(value)
    except (InvalidOperation, ValueError):
        raise ValueError("invalid decimal") from None
    if not number.is_finite() or abs(number) > Decimal("1e18"):
        raise ValueError("invalid decimal range")
    if number.as_tuple().exponent < -36 or len(number.as_tuple().digits) > 54:
        raise ValueError("unbounded decimal precision")
    if nonnegative and number < 0:
        raise ValueError("negative price")
    return format(number, "f")


def timestamp(value):
    if value is None:
        return None
    if not isinstance(value, str) or len(value) > 40:
        raise ValueError("invalid timestamp")
    try:
        parsed = datetime.fromisoformat(value.replace("Z", "+00:00"))
    except ValueError:
        raise ValueError("invalid timestamp") from None
    if parsed.tzinfo is None or parsed.utcoffset() is None:
        raise ValueError("timestamp needs timezone")
    return int(parsed.astimezone(timezone.utc).timestamp())


def identity(value):
    if not isinstance(value, str) or not 1 <= len(value) <= 128:
        raise ValueError("invalid coin id")
    if value != value.strip() or not value.isprintable():
        raise ValueError("invalid coin id")
    return value


def parse_rows(rows):
    if not isinstance(rows, list):
        raise ValueError("expected a coin array")
    seen, result = set(), []
    for row in rows:
        if not isinstance(row, dict):
            raise ValueError("invalid coin row")
        asset_id = identity(row.get("id"))
        if asset_id in seen:
            raise ValueError("duplicate coin id")
        seen.add(asset_id)
        rank = row.get("market_cap_rank")
        if type(rank) is not int or rank < 1:
            raise ValueError("invalid provider rank")
        symbol, name = row.get("symbol"), row.get("name")
        if not isinstance(symbol, str) or not 1 <= len(symbol) <= 64 or not symbol.isprintable():
            raise ValueError("invalid symbol")
        if not isinstance(name, str) or not 1 <= len(name) <= 256 or not name.isprintable():
            raise ValueError("invalid name")
        price = row.get("current_price")
        change = row.get("price_change_percentage_24h")
        result.append(dict(
            asset_id=asset_id, symbol=symbol, name=name, provider_rank=rank,
            price_usd=None if price is None else exact_decimal(price, nonnegative=True),
            change_24h=None if change is None else exact_decimal(change),
            updated_at=timestamp(row.get("last_updated")),
        ))
    return result


def parse_catalog(rows):
    if not isinstance(rows, list) or len(rows) != 200:
        raise ValueError("catalog requires exactly 200 rows")
    return parse_rows(rows)


def read_json(url, api_key):
    if not api_key:
        raise RuntimeError("CoinGecko Demo API key is required")
    request = urllib.request.Request(url, headers={
        "x-cg-demo-api-key": api_key,
        "User-Agent": "HermesCryptoGadgetPrototype/1",
    })
    try:
        with urllib.request.urlopen(request, timeout=8) as response:
            body = response.read(MAX_BYTES + 1)
        if len(body) > MAX_BYTES:
            raise ValueError("provider response too large")
        return json.loads(body, parse_float=Decimal)
    except Exception:
        raise RuntimeError("market provider request failed") from None


def fetch_catalog(api_key):
    return parse_catalog(read_json(MARKET_URL, api_key))
```

**2.5** Run the same command, expect all parametrized tests to pass, then commit:

```bash
uv run pytest tests/test_market.py -q
git diff --check
git add src/hermes_crypto_gadget/market.py tests/test_market.py tests/fixtures/market_rows.py
git commit -m 'feat: validate a bounded 200-coin USD market catalog'
```

### 3. Prove that all 200 coins are accessible on the display

**3.1** Create `tests/test_pages.py`:

```python
import pytest
from hermes_crypto_gadget.market import parse_catalog
from hermes_crypto_gadget.pages import market_page, usd
from tests.fixtures.market_rows import synthetic_market_rows


def test_all_fifty_pages_cover_exactly_200_ids_once():
    coins = parse_catalog(synthetic_market_rows())
    pages = [market_page(coins, page) for page in range(1, 51)]
    collected = [coin["asset_id"] for page in pages for coin in page["coins"]]
    assert len(pages) == 50
    assert len(collected) == len(set(collected)) == 200
    assert set(collected) == {coin["asset_id"] for coin in coins}
    assert collected[0] == "bitcoin"
    assert all(len(page["coins"]) == 4 for page in pages)
    assert all(len(line) <= 26 for page in pages for line in page["lines"])


@pytest.mark.parametrize("page", [0, 51, True, "1"])
def test_page_range_is_checked(page):
    with pytest.raises(ValueError):
        market_page(parse_catalog(synthetic_market_rows()), page)


def test_small_nonzero_price_never_displays_as_zero():
    assert usd("0.000000000001") == "<$0.00000001"
    assert usd(None) == "unknown"
    assert usd("0") == "$0.00"
```

**3.2 RED:**

```bash
uv run pytest tests/test_pages.py -q
```

Expected: missing `pages` module. After adding the function skeleton, demonstrate failed page coverage/range checks before their implementations.

**3.3 GREEN:** Create `src/hermes_crypto_gadget/pages.py`:

```python
from decimal import Decimal


def usd(value):
    if value is None:
        return "unknown"
    number = Decimal(value)
    if not number.is_finite() or number < 0:
        raise ValueError("invalid USD display value")
    if number == 0:
        return "$0.00"
    if number < Decimal("0.00000001"):
        return "<$0.00000001"
    if number >= Decimal("1000000000"):
        return f"${number / Decimal('1000000000'):.2f}B"
    if number >= 1:
        return f"${number:.2f}"
    return ("$" + format(number, ".8f")).rstrip("0").rstrip(".")


def market_page(coins, page):
    if len(coins) != 200 or len({coin["asset_id"] for coin in coins}) != 200:
        raise ValueError("expected a validated 200-coin catalog")
    if type(page) is not int or not 1 <= page <= 50:
        raise ValueError("page must be an integer in 1..50")
    ordered = [coin for coin in coins if coin["asset_id"] == "bitcoin"]
    ordered += [coin for coin in coins if coin["asset_id"] != "bitcoin"]
    selected = ordered[(page - 1) * 4:page * 4]
    lines = []
    for coin in selected:
        symbol = coin["symbol"].encode("ascii", "replace").decode().upper()[:6]
        label = f"{coin['provider_rank']} {symbol} {usd(coin['price_usd'])}"
        lines.append(label if len(label) <= 26 else f"{coin['provider_rank']} {symbol} view coin")
    return dict(page=page, pages=50, coins=selected, lines=lines, currency="USD")
```

**3.4** Run and commit:

```bash
uv run pytest tests/test_pages.py tests/test_market.py -q
git diff --check
git add src/hermes_crypto_gadget/pages.py tests/test_pages.py
git commit -m 'feat: paginate all 200 assets into fifty device pages'
```

The test calculates completeness programmatically. Seeing page 1 and Bitcoin alone is not an acceptance test for 200-coin support.

### 4. Implement the retained registry and USD journal in small TDD slices

Files: `src/hermes_crypto_gadget/store.py`, `tests/test_store.py`.

Implement each numbered behavior as a separate failing test, minimal implementation, rerun, and local commit. The data contracts above are mandatory. Do not reuse the old three-symbol `ASSETS` constant or EUR field names.

**4.1 Registry:** `replace_catalog(catalog)` runs one SQLite transaction, upserts each asset by exact ID, replaces the active-membership table, and never deletes known asset or event rows. Test two synthetic 200-ID catalogs, with one replacement ID in the second; the retained registry has 201 IDs and the active catalog still has 200.

**4.2 Journal identity:** Reject unregistered IDs, allow different IDs with the same symbol, and reject a symbol supplied where an ID is required. Persist `currency=USD` and a schema version. An EUR marker causes a clear error, not a migration-by-renaming.

**4.3 Amount validation:** Accept bounded positive quantity strings, actual USD trade-price strings, and explicitly supplied nonnegative USD fees. Reject float, NaN, Infinity, negative fee, missing required trade fields, and unknown fields. `OPEN` stores `price_usd=null`, `fee_usd=0`. Use `decimal.localcontext(prec=80)` in accounting paths and never normalize through a default-precision arithmetic operation that drops digits.

**4.4 Persistence and balances:** Append confirmed events atomically; calculate balances by `asset_id`; reject overselling inside the same write transaction. Reopening the DB produces identical events and balances. Event IDs are idempotency keys, not ticker hashes.

**4.5 Backup:** Use SQLite backup into a new private target, explicitly close connections, and compare the entire restored registry, journal, and quote cache. Do not overwrite a live DB during a restore test.

Command for every slice:

```bash
uv run pytest tests/test_store.py -q
git diff --check
git add src/hermes_crypto_gadget/store.py tests/test_store.py
git commit -m 'feat: add the verified USD registry or journal behavior'
```

RED is the missing behavior/assertion; GREEN is every test in the file passing. Use a meaningful commit subject for the actual slice rather than blindly copying the generic subject.

### 5. Poll and value holdings without 200 requests

Files: `src/hermes_crypto_gadget/polling.py`, `src/hermes_crypto_gadget/portfolio.py`, `tests/test_polling.py`, `tests/test_portfolio.py`.

**5.1 Poll schedule:** Add an injectable monotonic clock and provider function. At startup fetch one catalog. Schedule the next attempt no sooner than 300 seconds after the previous request begins. Provider errors preserve the last accepted snapshot and mark it stale/error. Do not loop immediately on 429 or wake a model for refresh.

**5.2 Held-asset extras:** Read retained positive-balance IDs absent from the current catalog. With no extras, make exactly one total request. With extras, make exactly one additional `coins/markets` request with URL-encoded `ids`, `vs_currency=usd`, `per_page=250`, `page=1`, and `sparkline=false`. Reject more than 200 held extra IDs. Validate returned IDs against the requested set; mark missing extra quotes unavailable. Never turn a partial response into a false current valuation.

**5.3 Budget:** A five-minute catalog schedule has 8,640 calls per 30-day continuous period. An always-needed extra bundle makes that 17,280. These are scheduling calculations, not a provider-plan entitlement. Confirm actual quota and attribution requirements before live polling.

**5.4 Portfolio valuation:** Multiply each recorded decimal quantity by that ID's current or explicitly stale quote. An unknown quote makes `total_usd=null`; expose `available_subtotal_usd` separately. Preserve exact arithmetic; format only the device view. An asset leaving the top 200 stays in holdings and journal.

**5.5 Timestamp status:** A quote with no timestamp, a future timestamp beyond 60 seconds, a failed fetch, or age above 900 seconds is not current. Carry quote-level status into the coin detail and portfolio summary. Validate future timestamps using an injected wall clock.

Commands:

```bash
uv run pytest tests/test_polling.py tests/test_portfolio.py -q
git diff --check
git add src/hermes_crypto_gadget/polling.py src/hermes_crypto_gadget/portfolio.py tests/test_polling.py tests/test_portfolio.py
git commit -m 'feat: add bundled polling and honest USD portfolio valuations'
```

Expected test assertions: exact request count 1 or 2, no early retry, retained IDs after rank changes, correct decimal total, null total with one unavailable held asset, and unchanged cache after errors. Write these tests before each behavior.

### 6. Integrate Hermes without changing the running personal installation

Paths: `.deps/gadget-sdk`, `.deps/hermes-agent`, `integrations/hermes/gadget/`, `tests/test_controller.py`, `tests/test_gateway_e2e.py`.

**6.1** Clone and pin the upstreams:

```bash
git clone https://github.com/Adolanium/hermes-gadget-sdk.git .deps/gadget-sdk
git -C .deps/gadget-sdk checkout --detach ffb1a43c02b9cdb4ad87c7132d367cccd5177dae
git clone https://github.com/NousResearch/hermes-agent.git .deps/hermes-agent
git -C .deps/hermes-agent checkout --detach 2eceb0275501b09febdec88d08b80d1fff730cf8
```

Expected: exact commit outputs from `git -C .deps/gadget-sdk rev-parse HEAD` and `git -C .deps/hermes-agent rev-parse HEAD`. Both checkouts remain ignored. If they already exist, inspect instead of overwriting.

**6.2** Read the pinned upstream fixture and adapter signatures before writing wrappers. Copy the upstream plugin into `integrations/hermes/gadget/` only with its applicable license and notices. Keep the upstream's voice, HMAC identity, pairing, and prompt behavior. Create `portfolio_adapter.py`, `portfolio_controller.py`, and `portfolio_tools.py` in that copied plugin.

**6.3 Tools:** Register `portfolio_read`, `portfolio_view`, and `portfolio_record_draft` in toolset `gadget`. `portfolio_view` takes `view` (`portfolio`, `markets`, `coin`, `journal`), plus `page` (1..50) or an exact registered `asset_id` as appropriate. Reject unused/unknown parameters. Resolve ambiguous spoken symbols by asking for the coin name/rank, not guessing.

**6.4 Confirmation:** Bind each draft to configured device ID, current connection session, and a 60-second expiry. No commit tool exists. Only `on_prompt_reply` from the matching paired session can call the store's append path. A reconnect, cancellation, `/new`, competing SDK prompt, or expiry discards the draft. Duplicate replies write at most one event.

**6.5 Background:** The controller sends the currently selected page, with USD currency, source, quote age, and stale state. It does not send all 200 rows to the ESP32. Read and write the store on the controller's event loop; offload only blocking provider I/O. Polling must not create agent turns.

**6.6 Capability gate:** Run an isolated `HERMES_HOME` under ignored `runtime/`, enable only the Gadget platform, set `platform_toolsets.gadget: [gadget, no_mcp]`, and disable general terminal/file/code/delegation tools. Never copy personal credentials, memories, or another profile.

**6.7 Real integration tests:** Reuse the pinned upstream's real Gateway subprocess, simulator, pairing helper, and fake OpenAI peer. Do not replace the Gateway with a mock. Seed exactly 200 clearly synthetic catalog rows in a temporary store. The actual registered-tool catalog must contain the three portfolio tools and no generic write/terminal/MCP capabilities. Assert page 50 exposes IDs 197..200 and that every page has been covered by the separate page-completeness test.

**6.8 Additional E2E checks:** Drive read, market page selection, coin details, and draft tools through the actual tool registry; confirm through the real simulated C++ core; replay an old reply; try a foreign paired device; disconnect and reconnect mid-draft; send `/new` while a draft is open; run one synthetic STT/TTS turn. Only a valid owner confirmation changes the DB. Provider failure never erases its old cache.

Future verification command after installing the pinned Hermes test environment:

```bash
HERMES_AGENT_DIR=.deps/hermes-agent HERMES_GADGET_E2E=1 uv run pytest tests/test_gateway_e2e.py -q -rs
```

Expected: all named E2E assertions pass, exit code 0, and **zero skipped tests**. Missing physical hardware is not an excuse to skip the real Gateway/simulator path. Pin-specific fixture wrappers must be inspected and tested during integration; this document does not claim they already work.

### 7. Apply minimal firmware patches, with C++ tests first

Paths: `firmware/patches/portfolio-display.patch`, upstream `firmware/tests/test_app.cpp`, `firmware/core/include/hg/app.hpp`, `firmware/core/src/app.cpp`, and `docs/protocol.md`.

**7.1** Run the unmodified pinned SDK baseline:

```bash
.deps/hermes-agent/.venv/bin/hermes-gadget build-sim --test
```

Install the SDK editable into that separate environment first; run the command from `.deps/gadget-sdk`. Expected: all existing core tests pass. No firmware claim follows from a successful host test.

**7.2 RED:** Add C++ `Rig` tests proving a `display` frame with `background=true` updates an idle text page, does not cover Listening/Thinking/Reply/Prompt/Settings/OTA, and becomes `ALT` after 30 seconds without a host update or after a stale quote.

**7.3 GREEN:** Add a bounded RAM-only portfolio background to `App::h_display` and use it only in the existing Ready branch of `App::update_model`. Do not create a modal card or wake a sleeping display on each refresh. Clear private background data in `App::h_unpaired`. Preserve frames without `background` unchanged.

**7.4 Confirmation layout:** Give `pfolio-` prompts the existing non-hero text layout with readable price, quantity, USD fee, UTC time, and Yes/No hint. Reject values that exceed the verified layout. Do not shorten confirmation numbers. Add tests for non-clipping model text and later verify actual rendered pixels.

**7.5** Save the reviewed diff as `firmware/patches/portfolio-display.patch`, document application against the exact pinned SDK, and rerun the full core suite. Preserve third-party notices when distributing patched source. No SDK source edits outside `.deps/` occur without an intentional vendoring plan.

### 8. Hardware, TLS, and privacy gates

**8.1** Confirm the exact CoreS3 model/revision and the approved USB device. Purchase, flashing, real portfolio data, speech-provider costs, and private-data processing need explicit scope approval.

**8.2** Build the CoreS3 profile in the pinned SDK's `firmware/esp32` directory:

```bash
pio run -e m5stack-cores3
pio device list
```

Expected: build `SUCCESS` and the actual approved device/port identified. Record the tested PlatformIO/platform/library versions; do not substitute an unverified version pin.

**8.3** Flash only the explicitly approved port, using `pio run -e m5stack-cores3 -t upload --upload-port ACTUAL_APPROVED_PORT`. The placeholder is not an executable default. No trial GPIO changes or guessed USB target.

**8.4** Configure a separate Gadget runtime with a reachable WSS hostname, trusted certificate chain, and no certificate-verification bypass. Loopback WS is allowed only for artificial simulator tests. Verify actual peer certificate validation on the device before live voice or holdings.

**8.5** Physically verify 320x240 orientation, touch, microphone, speaker, pairing, USB recovery, all confirmation rows, and foreground/background priority. Verify all 200 IDs on the host and render page 1 and page 50 on the device. Do not require 200 firmware-resident icons or permanent balance storage.

**8.6** Run a two-hour voice/reconnect soak. Record resets, audio failures, and RAM trend in `docs/hardware-validation.md`. Use synthetic holdings in any public screenshots. Actual price-provider integration is a separate live check, never replaced with fabricated data.

## Tests / validation

Use RED -> GREEN -> focused diff review -> local commit for each behavior change. Pure documentation tasks do not need invented product tests.

Required automated evidence:

- Exactly 200 distinct live-provider IDs before an active-catalog update.
- Fifty pages collectively contain those same 200 IDs once each.
- USD request and USD money fields throughout; EUR DB rejected.
- Duplicate ticker symbols do not merge assets.
- No holdings disappear after rank 200 is crossed.
- One or two provider requests per interval, never 200.
- Invalid/partial/oversized responses preserve the old accepted cache.
- Null/missing quotes remain unknown; small nonzero prices never display as zero.
- Decimal accounting and journal persistence survive a restart.
- Overselling, foreign device, replay, expiry, and stale connection are rejected.
- Only physical owner confirmation enters the journal append path.
- Actual tool registry and actual Gateway/simulator path verified without skipped tests.
- Certificate validation, speech processing, layout readability, and physical hardware separately verified.

Final unit command:

```bash
uv run pytest tests/test_market.py tests/test_pages.py tests/test_store.py tests/test_polling.py tests/test_portfolio.py tests/test_controller.py -q
```

Expected: exit code 0 with all collected tests passing. No test count is claimed before those suites exist and run.

## GitHub publication and delivery

Repository: `https://github.com/timomoebes/hermes-crypto-gadget`.

Initial approved publication contains only README, architecture/security/implementation documentation, `.gitignore`, and directory READMEs. No source implementation, credentials, real holdings, runtime, or binaries.

Before each public push:

1. Inspect `git diff --cached --name-only` and the full staged diff.
2. Check secrets, private paths, recordings, identity files, and database artifacts.
3. Commit only the intended paths; never blindly stage the home directory.
4. Push to the exact approved `origin`.
5. Read GitHub visibility, default-branch HEAD, and published file content back through the API.
6. Compare the remote commit SHA with local `git rev-parse HEAD` before reporting success.

Do not push SDK clones, personal Hermes profiles, or the private agent's business/memory folders. No CI-green or functional-device claim in the README before real evidence exists.

## Risks, tradeoffs, and open questions

- The 200-coin interpretation is market-capitalization order, not alphabetical or a permanent frozen list.
- Provider ranks can have gaps. Preserve the reported rank and count distinct entries, not an assumed range of exactly 1..200.
- The catalog may contain stablecoins and wrapped assets. Removing them changes the requirement and needs a decision.
- A previously held asset outside the catalog may be delisted or unpriceable. The correct result is unavailable/stale, not an invented zero valuation.
- Symbol collisions, speech errors, decimal precision, and fiat conversion require explicit ID and USD validation.
- Five-minute prices are suitable for display, not execution guarantees. Provider quota and terms remain a live gate.
- Self-hosting Hermes does not automatically keep speech/model processing local.
- Host SQLite privacy, backup, and correction policy remain operator responsibilities.
- CoreS3 hardware status is unverified until the named revision is tested.
- Wallet signing/keys, exchange import, P&L, tax handling, alerts, battery-life targets, and printed enclosure are out of the proposed first build.

## Definition of done

Initial repository milestone: public repo, verified remote commit and files, clear planned-vs-implemented status, safe ignore rules, USD/top-200 requirements, and documented structure.

Functional prototype milestone: implemented host code, real automated evidence, complete 200-ID access, scoped Gateway integration, certificate-verified voice/data transport, and an explicitly approved physical device passing the recorded hardware checks.

These are separate milestones. Publishing this scaffold completes only the initial repository milestone.
