# Architecture

Status: proposed, not implemented.

## Responsibilities

```text
CoinGecko HTTPS -> host market service -> private SQLite cache and coin registry
                                          |
CoreS3 <-------- authenticated WSS --------+------ Hermes Gadget adapter
 display, touch, microphone, speaker               |
                                                   +-- deterministic display updates
                                                   +-- scoped read/view/draft tools
                                                   +-- existing speech/model pipeline
                                                   +-- confirmed journal writes
```

The ESP32 is a terminal, not the market-data service or signing device. It receives only the selected view, not the entire 200-coin response. Model inference, portfolio accounting, and provider credentials stay on the host.

## Catalog and identity

- Catalog: the first 200 results of CoinGecko's USD market-capitalization-descending query.
- Validate exactly 200 distinct provider IDs before replacing the current catalog.
- Preserve the returned provider rank. Do not fabricate continuous ranks or silently substitute ticker symbols.
- Keep Bitcoin first in the market browser when present, without changing its stored provider rank or adding a 201st asset.
- Stable identity: CoinGecko `id`. Display symbol/name are metadata, never database keys.
- Preserve known asset metadata and recorded holdings when catalog membership changes.
- Stablecoins and wrapped assets remain included if the provider includes them. No hidden category filter.

## Prices and arithmetic

- All money fields are USD. No silent conversion of old EUR data.
- Parse provider JSON numbers as `Decimal`, not binary floats.
- Store exact decimal strings. Round only display values.
- A null/missing price remains unknown, not zero.
- Preserve per-coin timestamps and quote status. An unavailable held asset makes the complete portfolio total unknown; show any available subtotal as a subtotal.
- Poll one 200-row catalog request every 300 seconds. An additional bundled request covers previously recorded, held IDs outside the top 200.
- Never poll once per coin. Bound extra IDs and response bytes.

## Display

- CoreS3 target: 320 x 240 pixels.
- Market page: four coin rows, page label, and source/age footer. The 200-coin catalog therefore has 50 pages.
- Views: `portfolio`, `markets`, `coin`, `journal`.
- Voice commands select pages and coin IDs through device-scoped tools.
- Do not repurpose SDK cancel/confirmation gestures for navigation without a separate tested gesture change.
- Default page starts at Bitcoin when present.
- Background updates cannot cover a conversation, confirmation, settings, or firmware-update screen.
- Full journal confirmation values must fit without clipping. Reject an oversized confirmation instead of shortening a price or quantity.

## Persistence

Private host SQLite storage contains:

- retained asset identity and metadata;
- current validated catalog and quote cache;
- immutable, physically confirmed `OPEN`, `BUY`, and `SELL` journal events;
- explicit schema version and `USD` currency marker.

Never erase journal rows during a ranking refresh. Never infer initial cost basis. Never accept a sale above the recorded balance. A production-grade journal also needs a tested correction/void workflow before it can be treated as authoritative bookkeeping.

## Upstream references

Starting SDK commit: `ffb1a43c02b9cdb4ad87c7132d367cccd5177dae`.

Starting Hermes compatibility-test commit from that SDK: `2eceb0275501b09febdec88d08b80d1fff730cf8`.

Store upstream checkouts under ignored `.deps/`. Do not modify a running personal Hermes installation or copy its credentials. Any copied upstream source must retain the applicable license and notices.
