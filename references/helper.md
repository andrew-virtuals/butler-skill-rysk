# The Rysk helper

Write this to a file and run it with `python3`. Stdlib only - no key, no account,
no signature. Inventory and quotes are public reads.

```sh
python3 /tmp/rysk.py inventory
python3 /tmp/rysk.py screen --collateral 5000 --tenor monthly
python3 /tmp/rysk.py quote --asset ETH --collateral 5000 --tenor monthly
python3 /tmp/rysk.py quote --asset ETH --collateral 5000 --delta 0.20
```

Exit codes: `0` the read succeeded, `1` it did not.

Every quote comes back with `tradeable`, a `reasons` list when that is false, and a
`notes` list of things worth saying either way. **Never offer a quote that is not
`tradeable`**, always read the `notes`, and never soften either - they are written
for an owner already.

It contains a small WebSocket client because Rysk is RFQ over a socket and the
container ships no websocket package. That is the only reason it is here; nothing
else in the file is unusual.

```python
"""Read Rysk V12 and price an option before anyone sells one.

Rysk is an RFQ venue: you ask, real market makers answer, and the answer is a
signed quote good for sixty seconds. There is no order book to screen, so what
this helper does instead is turn an intent - "earn on $5,000 of USDC, ETH, a
month out" - into a quote with every number already fixed, and refuse the ones
an owner should not be shown.

Two things make Rysk different from a cash-settled venue, and both matter more
than the yield:

  * It settles **physically**. An assigned put hands the owner the asset, so
    "you now own 2.04 ETH at $2,150" is simply true. On a cash-settled venue it
    is not, and that gap has to be papered over with a separate spot buy.
  * Quotes die in **sixty seconds**. Anything that needs a human to read and
    tap will miss that window, so the signing path needs a session key rather
    than an approval card.

Everything here is a public read: inventory and quotes need no key, no account
and no signature. Selling does, and that path is not in this file.
"""

import argparse
import base64
import json
import os
import socket
import ssl
import struct
import sys
import time

HOST = "v12.rysk.finance"
TAKER_PATH = "/taker"
GAS_URL = "https://v12.rysk.finance/api/gas"

#: Rysk's own units. Strikes are 8dp, everything else is 18dp.
E8 = 10 ** 8
E18 = 10 ** 18

#: hyperliquid mainnet; 998 is their testnet.
CHAIN_ID = 999

TENOR_DAYS = {"weekly": 7, "monthly": 30, "quarterly": 90}

# --- the gates -------------------------------------------------------------
# Rysk has no spread to screen, so these are about the promise the owner is
# making, not about execution quality.

#: How likely assignment is, as delta. Past this an owner is being sold an
#: option that will probably be exercised, which is a different conversation
#: from "earn on idle cash".
MAX_ASSIGNMENT_DELTA = 0.35

#: Below this the premium does not pay for the attention it costs.
MIN_APR_PCT = 4.0

#: Rysk takes fixed size steps that differ per asset and are not published in
#: the inventory - ETH moves in 0.5, BTC in 0.05, SOL in 10, HYPE in 50. Rather
#: than hard-code a table that will drift, the helper walks this ladder and
#: lets the venue say which step it accepts.
SIZE_STEPS = (0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5, 10, 50, 100, 500, 1000)

#: A quote is signed and dies at `validUntil`. Anything closer than this to
#: expiring cannot survive a round trip, let alone a human.
MIN_QUOTE_SECONDS_LEFT = 20


class RyskError(RuntimeError):
    pass


# --- a websocket, in the standard library ----------------------------------
# Rysk is RFQ over a socket, and the container ships no websocket package, so
# this is the handshake and two frame operations and nothing else.


def _connect(timeout=25):
    raw = socket.create_connection((HOST, 443), timeout=timeout)
    sock = ssl.create_default_context().wrap_socket(raw, server_hostname=HOST)
    key = base64.b64encode(os.urandom(16)).decode()
    sock.send((
        "GET %s HTTP/1.1\r\nHost: %s\r\nUpgrade: websocket\r\nConnection: Upgrade\r\n"
        "Sec-WebSocket-Key: %s\r\nSec-WebSocket-Version: 13\r\n"
        "Origin: https://app.rysk.finance\r\n\r\n" % (TAKER_PATH, HOST, key)
    ).encode())
    head = b""
    while b"\r\n\r\n" not in head:
        chunk = sock.recv(4096)
        if not chunk:
            raise RyskError("Rysk closed the connection during the handshake")
        head += chunk
    first = head.split(b"\r\n", 1)[0].decode(errors="replace")
    if "101" not in first:
        raise RyskError("Rysk did not accept the connection: %s" % first)
    return sock


def _send(sock, payload):
    data = json.dumps(payload).encode()
    mask = os.urandom(4)
    size = len(data)
    header = b"\x81"
    if size < 126:
        header += bytes([0x80 | size])
    elif size < 65536:
        header += bytes([0x80 | 126]) + struct.pack(">H", size)
    else:
        header += bytes([0x80 | 127]) + struct.pack(">Q", size)
    sock.send(header + mask + bytes(b ^ mask[i % 4] for i, b in enumerate(data)))


def _recv(sock, timeout=25):
    sock.settimeout(timeout)
    head = sock.recv(2)
    if len(head) < 2:
        raise RyskError("Rysk sent nothing back")
    size = head[1] & 0x7F
    if size == 126:
        size = struct.unpack(">H", sock.recv(2))[0]
    elif size == 127:
        size = struct.unpack(">Q", sock.recv(8))[0]
    buf = b""
    while len(buf) < size:
        part = sock.recv(min(65536, size - len(buf)))
        if not part:
            break
        buf += part
    return buf.decode(errors="replace")


def _call(payload, want_id, tries=3, timeout=25):
    """One request/response over a fresh socket, retried on a bad minute."""
    last = None
    for attempt in range(tries):
        sock = None
        try:
            sock = _connect()
            _send(sock, payload)
            deadline = time.time() + timeout
            while time.time() < deadline:
                body = json.loads(_recv(sock, timeout=max(2, deadline - time.time())))
                if body.get("id") != want_id:
                    continue
                if body.get("error"):
                    raise RyskError(str(body["error"])[:200])
                return body.get("result")
            raise RyskError("Rysk did not answer in %ds" % timeout)
        except RyskError:
            raise
        except Exception as exc:  # noqa: BLE001 - retried, then surfaced
            last = exc
            if attempt < tries - 1:
                time.sleep(1.5 * (attempt + 1))
        finally:
            if sock is not None:
                try:
                    sock.close()
                except Exception:
                    pass
    raise RyskError("could not reach Rysk: %s" % last)


# --- reads -----------------------------------------------------------------


def inventory():
    """Every asset Rysk offers, with its strikes, expiries and indicative APY."""
    return _call({"id": "inventory", "jsonrpc": "2.0", "method": "inventory"}, "inventory")


def _puts(inv, asset):
    out = []
    for combo in (inv.get(asset, {}).get("combinations") or {}).values():
        if combo.get("isPut"):
            out.append(combo)
    return out


def _pick(combos, tenor, otm_pct, target_delta):
    """The listed strike closest to what was asked for, at the nearest expiry."""
    want_days = TENOR_DAYS.get(tenor, TENOR_DAYS["monthly"])
    live = [c for c in combos if (c.get("timeToExpiryDays") or 0) >= 1]
    if not live:
        return None
    best_exp = min({round(c["timeToExpiryDays"]) for c in live},
                   key=lambda d: abs(d - want_days))
    at_exp = [c for c in live if round(c["timeToExpiryDays"]) == best_exp]
    if target_delta is not None:
        return min(at_exp, key=lambda c: abs(abs(c.get("delta") or 0) - abs(target_delta)))
    return min(at_exp, key=lambda c: abs(
        (c["strike"] / (c.get("index") or 1) - 1) * 100 + otm_pct))


def quote(asset, collateral_usd, tenor="monthly", otm_pct=10.0, target_delta=None):
    """Ask real makers what they will pay for one cash-secured put.

    The returned price is signed and expires; `valid_for_seconds` is how long
    is actually left, not how long it was issued for.
    """
    asset = asset.upper()
    inv = inventory()
    if asset not in inv:
        raise RyskError("Rysk does not list %s. It offers: %s"
                        % (asset, ", ".join(sorted(inv))))
    combos = _puts(inv, asset)
    if not combos:
        raise RyskError("Rysk lists %s but is offering no puts on it right now" % asset)

    combo = _pick(combos, tenor, otm_pct, target_delta)
    if combo is None:
        raise RyskError("no %s put with an expiry more than a day out" % asset)
    products = combo.get("products") or []
    if not products:
        raise RyskError("no tradeable product for that %s strike" % asset)
    product = products[0]

    strike = float(combo["strike"])
    index = float(combo.get("index") or 0)
    # A cash-secured put is collateralised at the strike, so the size the
    # owner can afford falls straight out of what they are willing to commit.
    wanted = collateral_usd / strike if strike else 0

    def ask(size):
        return _call({
            "id": "request", "jsonrpc": "2.0", "method": "request",
            "params": {
                "asset": product["asset"], "assetName": asset, "chainId": CHAIN_ID,
                "expiry": combo["expiration_timestamp"], "isPut": True,
                "quantity": str(int(round(size * E18))),
                "strike": str(int(round(strike * E8))),
                "taker": "0x" + "0" * 40,
                "usd": product["strikeAsset"],
                "collateralAsset": product["collateralAsset"],
            },
        }, "request", tries=1, timeout=15)

    # Largest step that still divides what the owner can afford, walking up
    # until the venue stops complaining about the increment or the minimum.
    result, used, last = None, 0.0, None
    for step in [s for s in SIZE_STEPS if s <= max(wanted, SIZE_STEPS[0])][::-1]:
        size = int(wanted / step) * step
        if size <= 0:
            continue
        try:
            result, used = ask(round(size, 8)), size
            break
        except RyskError as exc:
            last = exc
            continue
    if result is None:
        raise RyskError(
            "Rysk would not price $%s of %s: %s. Its sizes step per asset - ETH in 0.5, "
            "BTC in 0.05, SOL in 10, HYPE in 50 - so a small amount may be under the "
            "minimum." % (format(collateral_usd, ",.0f"), asset,
                          str(last) if last else "no maker answered"))

    # `price` is PER UNIT, not the total - the same figure comes back for 0.5,
    # 1 and 1.5 of the same option. Multiplying is the whole premium.
    unit_price = float(result.get("price") or 0) / E18
    filled = float(result.get("quantity") or 0) / E18 or used
    premium = unit_price * filled
    collateral = strike * filled
    days = float(combo.get("timeToExpiryDays") or 0)
    valid_until = int(result.get("validUntil") or 0)
    left = valid_until - int(time.time())
    apr = (premium / collateral * 365 / days * 100) if collateral and days else 0.0

    q = {
        "venue": "rysk",
        "product": "cash_secured_put",
        "settlement": "physical",
        "asset": asset,
        "chain_id": CHAIN_ID,
        "strike": strike,
        "index_price": round(index, 4),
        "pct_otm": round((strike / index - 1) * 100, 2) if index else None,
        "expiry": combo.get("expiry"),
        "expiry_ts": combo["expiration_timestamp"],
        "days_to_expiry": round(days, 2),
        "size": round(filled, 8),
        "delta": combo.get("delta"),
        "premium_usd": round(premium, 2),
        "premium_per_unit": round(unit_price, 4),
        "collateral": {"asset": "USD", "amount": round(collateral, 2)},
        "apr_pct": round(apr, 2),
        "breakeven": round(strike - unit_price, 2),
        "maker": result.get("maker"),
        "valid_until": valid_until,
        "valid_for_seconds": left,
        "quote_nonce": result.get("nonce"),
        "quote_signature": result.get("signature"),
        "tradeable": False,
        "reasons": [],
        "notes": [],
    }

    if filled <= 0:
        q["reasons"].append("the maker quoted no size - nobody will write that option now")
        return q
    # A short fit is worth saying, never worth hiding the note over: Rysk's
    # strikes and size steps are fixed, so a round dollar amount rarely lands
    # exactly. Refusing here would bury whichever asset pays best.
    if collateral_usd and collateral < collateral_usd * 0.95:
        q["notes"].append(
            "this puts $%s of the $%s to work - Rysk's strikes and sizes step, so a "
            "round amount rarely lands exactly"
            % (format(collateral, ",.0f"), format(collateral_usd, ",.0f")))
    if abs(combo.get("delta") or 0) > MAX_ASSIGNMENT_DELTA:
        q["reasons"].append(
            "this is close enough to the money that it will probably be exercised "
            "(delta %.2f) - a likely purchase, not idle cash earning"
            % abs(combo["delta"]))
    if apr < MIN_APR_PCT:
        q["reasons"].append(
            "%.1f%% a year does not pay for locking the money up until %s"
            % (apr, combo.get("expiry")))
    if left < MIN_QUOTE_SECONDS_LEFT:
        q["reasons"].append(
            "this quote has %ds left and dies at %d - re-quote rather than act on it"
            % (max(left, 0), valid_until))

    q["tradeable"] = not q["reasons"]
    return q


def screen(collateral_usd, tenor="monthly", otm_pct=10.0):
    """What Rysk will actually pay across everything it lists, right now."""
    inv = inventory()
    rows = []
    for asset in sorted(inv):
        try:
            rows.append(quote(asset, collateral_usd, tenor, otm_pct))
        except RyskError as exc:
            rows.append({"asset": asset, "venue": "rysk", "tradeable": False,
                         "reasons": [str(exc)]})
    rows.sort(key=lambda r: (not r.get("tradeable"), -(r.get("apr_pct") or 0)))
    return rows


def main():
    p = argparse.ArgumentParser(description=__doc__,
                                formatter_class=argparse.RawDescriptionHelpFormatter)
    sub = p.add_subparsers(dest="cmd", required=True)
    sub.add_parser("inventory", help="assets, strikes and expiries Rysk offers")
    s = sub.add_parser("screen", help="what every listed asset pays right now")
    s.add_argument("--collateral", type=float, required=True)
    s.add_argument("--tenor", default="monthly", choices=sorted(TENOR_DAYS))
    s.add_argument("--otm-pct", type=float, default=10.0)
    q = sub.add_parser("quote", help="one signed quote for a cash-secured put")
    q.add_argument("--asset", required=True)
    q.add_argument("--collateral", type=float, required=True)
    q.add_argument("--tenor", default="monthly", choices=sorted(TENOR_DAYS))
    q.add_argument("--otm-pct", type=float, default=10.0)
    q.add_argument("--delta", type=float, default=None)
    a = p.parse_args()

    if a.cmd == "inventory":
        inv = inventory()
        out = {}
        for asset, body in inv.items():
            combos = list((body.get("combinations") or {}).values())
            out[asset] = {
                "strikes": len(combos),
                "puts": sum(1 for c in combos if c.get("isPut")),
                "calls": sum(1 for c in combos if not c.get("isPut")),
                "expiries": sorted({c.get("expiry") for c in combos if c.get("expiry")}),
            }
        print(json.dumps(out, indent=2))
    elif a.cmd == "screen":
        print(json.dumps(screen(a.collateral, a.tenor, a.otm_pct), indent=2))
    else:
        print(json.dumps(quote(a.asset, a.collateral, a.tenor, a.otm_pct, a.delta), indent=2))


if __name__ == "__main__":
    try:
        main()
    except RyskError as exc:
        print("error: %s" % exc, file=sys.stderr)
        sys.exit(1)
    except Exception as exc:  # noqa: BLE001 - one line for the skill to read
        print("error: %s" % exc, file=sys.stderr)
        sys.exit(1)
```
