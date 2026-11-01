# ADR 0001: Order quantity format

- **Status:** Accepted
- **Date:** 2026-11-01
- **Decided by:** Ondřej Ivan

## Context

The app places buy orders with `POST /api/v1/orders`. The first draft of the contract sent the quantity as a JSON integer (`"quantity": 5`), so only whole units could be bought.

Several facts make the format of the quantity a decision worth recording now:

- **Fractional shares are common in retail investment apps.** A single unit of a broad ETF costs hundreds of crowns or more, so a user who invests 1,000 CZK a month cannot spread it over several funds in whole units.
- **The contract is hard to change later.** The Android app stays in users' phones for months and cannot be forced to update. Changing the type of `quantity` from a number to a string would break every installed version.
- **Markets may differ.** One market or execution partner may support fractions, another only whole units, and this may change over time.
- **Money already uses decimal strings.** Amounts are `{ "amount": "2650.00", "currency": "CZK" }` so that no client parses them as floating point.

## Options

| Option | Pros | Cons |
|---|---|---|
| **1. Integer quantity** (`"quantity": 5`) | Simplest for the client and the server | No fractions; adding them later is a breaking change that needs `/api/v2` or a parallel field |
| **2. Decimal string with a per-instrument minimum and step** (`"quantity": "2.5"`, `minQuantity`, `quantityStep`) | Fractions without a special format; can be enabled per market or instrument as data; `quantityStep: "1"` keeps whole units where needed; same approach as money | Clients parse decimals; reservation and settlement need explicit rounding rules; the server validates the step |
| **3. Decimal JSON number** (`"quantity": 2.5`) | Looks natural in JSON | Many parsers turn it into a binary floating-point number; rejected for the same reason as money |
| **4. Order by amount** (`"amount": { "amount": "1000.00", "currency": "CZK" }`) | Matches how people invest regularly ("1,000 CZK a month") | The quantity is known only after execution; settlement must compute and round it; a different request shape and more complex reservation |

## Decision

**Option 2.** `quantity` is a decimal string with at most 6 decimal places. Each instrument in `GET /api/v1/instruments` states `minQuantity` and `quantityStep`. An order whose quantity is below the minimum or not a multiple of the step is rejected with `422` and code `QUANTITY_NOT_ALLOWED`.

Six decimal places are enough for fractions of an ETF unit and leave room for partners with finer precision; the actual precision of an instrument is set by its `quantityStep`.

Ordering by amount (option 4) is **out of scope for now**. If it comes, it must be added without breaking old clients: a new optional request field next to `quantity`, and the `Order` response must keep `quantity` non-null for orders placed by quantity, for example by adding a separate field for the requested amount instead of making `quantity` nullable.

## Business impact

- Users can invest small regular amounts and buy expensive ETFs, which lowers the entry barrier and supports regular investing.
- Fractions can be turned on for one market or instrument at a time, without an app release.
- In a real platform, fractional shares need an execution partner that supports them, or internal aggregation of fractions into whole units, and clear information for clients about how fractions are held. The demo simulates execution, so this is outside its scope.

## Cost

- **Now:** low. Nothing is implemented yet; only the contract and the architecture document change.
- **Implementation:** `BigDecimal` for quantities everywhere, `NUMERIC` columns in `orders` and `position`, a step check (`quantity mod quantityStep == 0`), the reserved amount rounded up to the currency's decimal places and an explicit rounding rule for the settled cost.
- **Cost of deciding later:** a breaking change for every installed app version, a new major version of the API and months of running both versions.

## Consequences

- Clients treat quantities like money: parse as decimals, never as `Double`, and format for display by locale.
- The idempotency fingerprint uses the normalized quantity, so `"5"` and `"5.0"` are the same request.
- market-service provides `minQuantity` and `quantityStep` with each instrument of a market.
- Tests cover quantities at the minimum, off the step and with more than 6 decimal places.
