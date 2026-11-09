# e-pet client app

The phone plays the cat. This repository is that client, and it has no app code yet.

It relies on the [e-pet-backend](https://github.com/iravan/e-pet-backend) server. The server keeps one shared seed and the hunger clock, including cats whose owners are offline. It does not simulate a cat. This app does.

## What the server provides

The contract lives in that repository's `DESIGN.md`. This app uses it as follows.

| Call | Use |
| --- | --- |
| `GET /api/stream-seed` | SSE. One seed about every second. A new connection receives the current seed first |
| `POST /api/cats` | Create the cat. Fullness starts at 100 |
| `GET /api/cats/{id}` | Read name, fullness, and `ALIVE` or `DEAD` |
| `POST /api/cats/{id}/feed` | Add fullness, capped at 100. A dead cat is refused |
| `POST /api/cats/{id}/revive` | Bring a dead cat back |
| `POST /api/cats/{id}/verify` | Claim a lucky pickup. The server answers `202` and rolls against the current seed |

Each SSE `data` line is `{"seed":"<16 hex chars>","timestamp":<unix seconds>,"type":"TRNG"}` or `"type":"LOCAL_CSPRNG_FALLBACK"`. A comment line `: ping` is only a keep-alive. `{"error":"client_lag"}` means this connection fell behind. Keep the last good seed and continue.

A disconnected phone keeps its last seed until the stream returns.

## What this app does

Salt the shared seed with the cat id and roll behaviour locally:

```text
personal = SHA-256(utf8(seed_hex) ‖ utf8(cat_id))
roll = first 8 bytes of SHA-256(personal ‖ event_name ‖ counter) as u64, mod 100
```

`counter` starts at 0 for each seed. Walk the weights in the order below.

| Event | Weights | On the phone |
| --- | --- | --- |
| `FOOD_EVENT` (食) | sleep 50, groom 30, walk 15, zoomies 5 | locked 30 minutes |
| `POOP_EVENT` (痾) | zoomies 60, walk 25, groom 10, sleep 5 | locked 5 minutes |
| `WAKE_EVENT` (瞓醒) | hungry 60, walk 25, zoomies 10, groom 5 | hungry until a feed succeeds |
| Idle, every second | walk 10, sit 85, hungry 5 | until one of the events above |

Hunger is not rolled here. The server drains fullness. This app shows the cat from `GET /api/cats/{id}` and sends feed and revive.

A lucky pickup is `POST /api/cats/{id}/verify` with the timestamp of the seed in hand, `triggered_event` `LUCKY_ITEM_PICKUP`, and an `action_nonce` from 1 to 10000. The server stores the verdict. This app does not grant an item.
