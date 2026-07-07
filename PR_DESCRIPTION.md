# fix: kline empty result (u-cookie + count arg) and northbound_shareholding dict wrapping; pin fastmcp<3

## Summary

This PR fixes three runtime bugs that made two tools unusable, and pins `fastmcp` to the 2.x line so the project installs and imports cleanly.

- `kline` always returned an empty result — from two independent causes (a wrong positional argument, and a missing `u` cookie the Xueqiu endpoint uniquely requires).
- `northbound_shareholding_sh` / `northbound_shareholding_sz` crashed under fastmcp because they returned a bare `list` where fastmcp requires a `dict`.
- `fastmcp>=2.1.2` (unpinned) resolves to `fastmcp` 3.x, which relocates `FastMCP` out of the top-level namespace and breaks `from fastmcp import FastMCP`.

All changes are confined to `main.py` and `pyproject.toml`. No public tool signatures change, and existing `.env` token formats remain backward compatible.

---

## Fix 1 — `kline` passed `days` into the `period` argument

**Symptom:** `kline(stock_code, days)` returned an empty envelope (`{"items": [], "items_size": 0}`) for every symbol, with no error.

**Root cause:** `pysnowball`'s signature is `kline(symbol, period='day', count=284)`. The original call passed `days` positionally, so it landed in `period` — making `period` a number (invalid) rather than setting the candle count.

**Fix:** call by keyword — fix `period` to `"day"` and pass `days` as `count`.

```python
# before
result = ball.kline(stock_code, days)

# after
result = ball.kline(stock_code, period="day", count=days)
```

---

## Fix 2 — `kline` endpoint requires a `u` (uid) cookie

**Symptom:** even after Fix 1, `kline` still returned an empty `items` list — HTTP 200, `error_code: 0`, no error message.

**Root cause:** Xueqiu's `kline.json` endpoint uniquely requires a `u` (uid) cookie to be present. Without it the endpoint silently returns an empty envelope. All other endpoints (`quotec`, `quote_detail`, `capital`, …) work without `u`, which masks the requirement. The value is not validated — any placeholder (e.g. `u=1`) satisfies it.

**Fix:** a small token loader auto-appends `u=1` to `XUEQIU_TOKEN` when no `u=` is already present. Tokens that already contain a real `u=` are left untouched, so this is fully backward compatible.

```python
# before
ball.set_token(os.getenv("XUEQIU_TOKEN"))

# after
def _load_token():
    """Read XUEQIU_TOKEN and append a placeholder `u` cookie if missing.

    Xueqiu's kline.json endpoint requires a `u` (uid) cookie; without it it
    silently returns an empty {"items": []} envelope (HTTP 200, error_code 0).
    The value is not validated, so u=1 is enough. Tokens that already carry a
    real u= are left unchanged.
    """
    raw = os.getenv("XUEQIU_TOKEN") or ""
    if not re.search(r"(^|;|\s)u=", raw):
        raw = raw.rstrip().rstrip(";") + "; u=1;"
    return raw

ball.set_token(_load_token())
```

---

## Fix 3 — `northbound_shareholding_sh/sz` returned a bare list

**Symptom:** calling either tool raised
`structured_content must be a dict or None. Got list`.

**Root cause:** both tool functions returned a Python `list`, but fastmcp requires tool return values to be a `dict` (or `None`).

**Fix:** wrap the list in `{"data": ...}`.

```python
# before  (both northbound_shareholding_sh and _sz)
return process_data(result)

# after
return {"data": process_data(result)}
```

---

## Note 4 — pin `fastmcp>=2.1.2,<3`

`dependencies` previously specified `fastmcp>=2.1.2` (unbounded). Resolving that today pulls in `fastmcp` 3.x, which moves `FastMCP` out of the top-level namespace and breaks `from fastmcp import FastMCP`. (Separately, the older `fastmcp` 2.1.2 + `mcp` 1.6.0 pair fails at import with `ImportError: CreateMessageResultWithTools`.) Pinning to the 2.x line keeps the current import and API working.

```toml
# before
"fastmcp>=2.1.2",

# after
"fastmcp>=2.1.2,<3",
```

---

## Testing

- **kline:** `kline("SH600000")` now returns real candle data (e.g. 10 candles for a 10-day request) instead of an empty list.
- **northbound_shareholding:** `northbound_shareholding_sz()` returns a dict — `{"data": [...]}` — with ~2150 rows, and no longer raises the `structured_content` error.
- **server boot:** the server starts cleanly and logs `Starting MCP server 'Xueqiu MCP'`, confirming the fastmcp import works under the pin.

---

## Notes for maintainers

- The `u`-cookie requirement for `kline.json` is undocumented upstream and easy to miss, because every other endpoint works without it. The auto-append only adds a placeholder when `u=` is absent, so existing `.env` files with a real `u=` are unaffected.
- No secrets are included in this PR; all token references use placeholders.
