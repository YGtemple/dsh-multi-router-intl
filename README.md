<div align="right">

**[English](README.md)** · [简体中文](README.zh-CN.md)

</div>

# dsh-multi-router-intl

Route model requests across several clients — WorkBuddy (CN + international) and Trae — inside DeepSeek Harness, with a standalone international chat path.

<sub>TypeScript · no build step required to run · ships with dependency lock and change log</sub>

---

## Why this exists

WorkBuddy's international build would not go through. Same product, same model, but three things differ from the CN build:

| | CN build | International build |
|---|---|---|
| Login | one redirect | separate redirect + callback |
| Endpoint | one host | different host |
| Session refresh | tolerant | expires fast, needs proactive renewal |

The first fix was straightforward but wrong: two parallel implementations side by side. Every change had to be made twice.

What made the structure click was realising **this was never about WorkBuddy**. Strip the product names and it is always the same operation:

```
request arrives → decide which client handles it → forward → stream the response back
```

So the client became *data* instead of *code*.

---

## How it is put together

```
src/
├── clients.mjs          each client is one declarative entry
├── router.mjs           dispatch — knows no product by name
├── accounts.mjs         per-account state
├── workbuddy-login.mjs  the one flow that could not be shared
├── stream.mjs           unified streaming
├── config.mjs           configuration
└── manager.mjs          orchestration
```

**A client entry declares** what actually varies: endpoint, auth method, whether login is custom, and capability differences (available models, context window).

Everything else — routing, streaming, retries, session bookkeeping — is shared.

---

## Adding Trae

Trae was the first real test of whether the abstraction held.

If adding a client means editing `router.mjs`, the abstraction is wrong and should be fixed rather than patched with another branch.

Adding Trae required exactly one thing: an entry in `clients.mjs`. `router.mjs`, `stream.mjs` and `config.mjs` were untouched.

That is the criterion I now use — **the cost of adding an implementation is the only honest test of an abstraction.**

---

## The part that did not get abstracted

Login.

The international build has its own redirect and callback; Trae has a different scheme entirely. Forcing these into one shape would have produced a config format more complicated than the code it replaced.

So it stays in `workbuddy-login.mjs`, on its own. **Specialising where the domain genuinely differs beats uniform abstraction.**

---

## Layout

```
src/          19 modules — router, clients, accounts, streaming, auth
scripts/      build, upstream fetch, host probe, live check, preview,
              plus two guard scripts (client discovery, live reload)
tests/
docs/         reference comparison, screenshots,
              evolution-workbuddy-to-trae.md
upstream/     pinned upstream copy
upstream-lock.json   dependency lock
```

### Guard scripts

| Script | Checks |
|---|---|
| `check-client-discovery.mjs` | every client entry is picked up by the router |
| `check-live-reload.mjs` | hot reload keeps state consistent |

---

## Running it

```bash
npm install
node scripts/build.mjs
node scripts/host-probe.mjs     # confirm the host answers
node scripts/live-check.mjs     # end-to-end against a real account
```

`scripts/preview.mjs` renders a local preview of the router configuration.

---

## Further reading

- [docs/evolution-workbuddy-to-trae.md](docs/evolution-workbuddy-to-trae.md) — the migration in detail
- [docs/reference-comparison.md](docs/reference-comparison.md) — behaviour compared against reference implementations
- [CHANGELOG.md](CHANGELOG.md) — change history
- [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) — upstream attribution

---

## Scope

Personal tooling for connecting model providers inside DeepSeek Harness. Not affiliated with WorkBuddy, Trae or Qoder. Accounts and credentials stay local.

## License

MIT — see [LICENSE](LICENSE).
