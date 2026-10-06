# Hermes Crypto Gadget

**USD crypto prices, a personal portfolio, and voice interaction on an ESP32-S3 device.**

Status: **planning and repository scaffold only**. No application, firmware changes, live price feed, or hardware validation is shipped yet.

## What we are building

- A dynamic catalog of the **top 200 cryptocurrencies by market capitalization**, starting with Bitcoin when it is present in that catalog.
- Coin prices, holdings valuations, and recorded trade prices in **USD**.
- A compact market browser, individual coin view, portfolio allocation view, and transaction journal.
- Push-to-talk voice interaction through [Hermes Agent](https://github.com/NousResearch/hermes-agent) and the community [Hermes Gadget SDK](https://github.com/Adolanium/hermes-gadget-sdk).
- Device confirmation before saving documentation of a completed purchase, sale, or opening balance.

The 200-coin catalog is fetched dynamically. It is not a hardcoded list and does not make 200 separate price requests. Stable provider coin IDs identify assets; ticker symbols are not unique.

## Wallet boundary

The proposed first version is a **portfolio companion**, not a hardware wallet. It does not hold private keys, seed phrases, exchange trading credentials, or execute trades.

An actual wallet is a separate security project. The recommended route is an established external hardware-wallet signer, with explicit user approval and no secrets entering the agent. The intended wallet scope still needs confirmation.

## Proposed hardware

**M5Stack CoreS3 K128**, with an ESP32-S3, touchscreen, dual microphones, and speaker. A separate Linux host runs Hermes, speech services, market-data requests, and the private portfolio database.

The CoreS3 profile exists in the selected Gadget SDK, but physical validation remains outstanding. The smaller StickS3 is not assumed compatible with that profile.

No custom enclosure or 3D-print design in the first milestone. The manufacturer's existing housing can remain in place.

## Market data

Planned provider: [CoinGecko](https://www.coingecko.com/).

Planned catalog request:

```text
GET /api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=200&page=1&sparkline=false&price_change_percentage=24h
```

Default refresh: **every five minutes**, with visible source, timestamp, and stale-data state. This is not an exchange trading feed. Provider quotas and attribution terms must be checked before continuous use.

Coins leaving the top 200 must not erase holdings or journal entries. Held assets outside the current catalog need a separate, bundled price request or an explicit unavailable/stale valuation.

## Project layout

```text
.
├── docs/
│   ├── architecture.md
│   ├── implementation-plan.md
│   └── security.md
├── src/hermes_crypto_gadget/    # planned host-side market and portfolio modules
├── integrations/hermes/        # planned device-scoped Hermes plugin integration
├── firmware/                   # planned patches against the upstream Gadget SDK
└── tests/                      # planned deterministic tests and gateway integration tests
```

The directories currently contain documentation only. No build or test result is claimed.

## Development milestones

- [x] Define USD and dynamic top-200 requirements.
- [x] Publish the initial repository and project structure.
- [ ] Implement and test catalog validation, coin-ID registry, and bounded USD price polling.
- [ ] Implement and test the decimal portfolio journal and paginated display models.
- [ ] Integrate the real Hermes Gateway and Gadget SDK simulator.
- [ ] Validate TLS, tool restrictions, speech providers, and device confirmations.
- [ ] Flash and validate an explicitly approved physical board.

See the [implementation plan](docs/implementation-plan.md), [architecture](docs/architecture.md), and [security boundaries](docs/security.md).

## Privacy and safety

Keep API keys, Wi-Fi credentials, recordings, portfolio databases, device identities, and runtime configuration outside Git. Do not use a public repository as your portfolio backup.

This project provides no investment advice, custody service, trade execution, or tax report. Public availability is not a security audit or a hardware-wallet certification.

## Upstream projects

- [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
- [Adolanium/hermes-gadget-sdk](https://github.com/Adolanium/hermes-gadget-sdk)
- [M5Stack CoreS3 documentation](https://docs.m5stack.com/en/core/CoreS3)
- [CoinGecko coins/markets documentation](https://docs.coingecko.com/demo/reference/coins-markets)

This is an independent project. It is not an official product or endorsement from Nous Research, M5Stack, or CoinGecko.

No third-party source has been vendored in this initial scaffold. A project license has not yet been selected; a public repository alone does not grant an open-source license.
