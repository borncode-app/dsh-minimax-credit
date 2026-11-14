> **This repository has moved into one monorepo.**
>
> The whole MiniMax set — the credit proxy, the daily check-in, the model-list
> sync, the composer quota chip, the MiniMax Code bridge, and the shared grant
> reader — now lives in
> **[`borncode-app/dsh-minimax-kit`](https://github.com/borncode-app/dsh-minimax-kit)**,
> one repository with one Plugins entry.
>
> | this repo | new home |
> |---|---|
> | `dsh-minimax-auth` | `packages/dsh-minimax-auth` |
> | `dsh-minimax-credit` | `packages/dsh-minimax-credit` |
> | `dsh-minimax-checkin` | `packages/dsh-minimax-checkin` |
> | `dsh-minimax-model-sync` | `packages/dsh-minimax-model-sync` |
> | `dsh-minimax-bridge` | `packages/dsh-minimax-bridge` |
> | `dsh-model-quota` | `packages/dsh-model-quota` |
>
> The code and its history are unchanged; only the repository is shared. Install
> `dsh-minimax-kit` instead of the individual bundles. This repository stays
> open so the old links keep working.
<div align="center">
  <img src="icon.svg" alt="" width="72" height="72">
  <h1>dsh-minimax-credit</h1>
  <p><b>Keeps the MiniMax route alive after the Token Plan quota is spent â€” and shows the credits left.</b></p>
  <p>
    <a href="https://github.com/borncode-app/dsh-minimax-credit/releases"><img alt="release" src="https://img.shields.io/github/v/release/borncode-app/dsh-minimax-credit?style=flat-square&color=6C4BF6"></a>
    <a href="LICENSE"><img alt="license" src="https://img.shields.io/badge/license-MIT-3DA639?style=flat-square"></a>
    <img alt="plugin" src="https://img.shields.io/badge/DeepSeek%20Harness-plugin-6C4BF6?style=flat-square">
    <img alt="wire" src="https://img.shields.io/badge/wire-Anthropic%20Messages-D97757?style=flat-square">
  </p>
</div>

---

## Where the one-entry kit lives

This plugin, `dsh-minimax-checkin`, `dsh-minimax-model-sync`, `dsh-model-quota`
and `dsh-minimax-bridge` are also published as one bundle:
**[`dsh-minimax-kit`](https://github.com/borncode-app/dsh-minimax-kit)**.

Install the kit instead if you want the whole MiniMax set from a single
**Plugins** entry. It ships these same packages as separate rows, so the code
here is unchanged and each part keeps its own switch â€” this repository stays
the home of the proxy, the quota probe, the credit and wallet reads, and the
composer chip.

## Where the sign-in code lives now

The MiniMax Code grant reader â€” reading `~/.minimax/auth`, refreshing it under
the app's own `auth.lock`, and the token/generation bookkeeping â€” moved to
**[`dsh-minimax-auth`](https://github.com/borncode-app/dsh-minimax-auth)**.

This plugin and `dsh-minimax-checkin` both import it, so one refresh path writes
the grant instead of two. The dependency is pinned to a commit; bump
`@local/dsh-minimax-auth` in `package.json` to take a newer one.

Nothing else moved: this repository still owns the proxy, the quota probe, the
credit and wallet reads, and the composer chip. The `TokenSource` export remains
here as a subclass that supplies this plugin's error class and
`MINIMAX_CREDIT_*` codes.

## The problem

A MiniMax Token Plan API key (`sk-cp-â€¦`) works against `https://api.minimax.io/anthropic` until the plan
quota runs out, and then it simply stops:

```json
HTTP 429
{"type":"error","error":{"type":"rate_limit_error",
 "message":"Token Plan usage limit reached: Upgrade your Token Plan or purchase Credits for more usage. (2056)"}}
```

Inside MiniMax Code the same moment is invisible, because that app calls the provider through its own proxy using
the account's OAuth grant (`mcode-public`) â€” a grant that is **billed to credits** once the plan is spent, on the
same Anthropic-Messages wire format. This plugin closes that gap for the Harness.

## What it does

It serves a **local Anthropic-Messages proxy** and points the `minimax-api` route at it:

1. forwards to the Token Plan endpoint while the plan has quota;
2. on a genuine plan-quota 429, retries the request against the MiniMax Code credit endpoint with the account's
   OAuth token â€” automatically, no model switch, no new session;
3. falls back to the plan the moment its quota comes back;
4. reads and shows the **remaining credits** of the account as a chip on the composer.

## Features

- **Transparent failover** â€” requests are relayed byte for byte (SSE, `tool_use`, `thinking`, image `tool_result`
  included); nothing in the payload is rewritten.
- **Plan â†’ credits â†’ plan** â€” no restart, no config flip; the direction is decided per request.
- **Credits chip** â€” the real balance from the account's membership record (`op_credit_summary`), not the Token
  Plan credit pool. Yellow when the plan is spent or credits are 0; red when the grant cannot be used.
- **Reuses MiniMax Code's grant in place** â€” reads `~/.minimax/auth/<buildEnv>/<region>/mcode-public/auth.json`,
  refreshes it with the same OAuth flow app uses, and writes the new grant back in the app's own format
  (bumping `generation`, taking the same lock file) so DSH and MiniMax Code never fight over one token.
- **Status route** â€” `GET /plugins/minimax-credit/status` (and `GET :45677/status`) reports `mode`, `plan.percent`,
  `credits.remaining`, `credits.wallet`, `token.ok`, and request counters.

## Requirements

| | |
|---|---|
| DeepSeek Harness | 0.2.x |
| Runtime | Node 24, supplied by the Harness runtime |
| MiniMax Code | Signed in on the **same machine** â€” the credit fallback has no other token source |
| Env | `MINIMAX_API_KEY` for the plan side |
| Port | `45677` free (configurable) |
| Peer | [`dsh-minimax-auth`](../dsh-minimax-auth) â€” the shared grant reader (a `link:` dependency, not a bundle) |

## Install

```bash
git clone https://github.com/borncode-app/dsh-minimax-credit.git dsh-minimax-credit
git clone https://github.com/borncode-app/dsh-minimax-auth.git dsh-minimax-auth
cd dsh-minimax-credit && pnpm install
```

Then:

```
plugin_manager install_bundle target=C:\path\to\dsh-minimax-credit
```

Finally point the route at the proxy â€” in your profile's `cordis.patch.yml`, the `llm-pi-ai` entry:

```yaml
- id: llm-pi-ai
  name: "@deepseek-ai/dsh-llm-pi-ai"
  config:
    providers:
      minimax-api:
        apiKeyEnv: MINIMAX_API_KEY
        api: anthropic-messages
        baseURL: http://127.0.0.1:45677/anthropic   # was https://api.minimax.io/anthropic
        # models: â€¦ unchanged
```

Restart once. A route's `baseURL` must be a literal origin, which is exactly why the proxy lives on loopback.

## Configuration

| field | default | meaning |
|---|---|---|
| `enabled` | `true` | `false` = plain proxy with no credit fallback |
| `listenHost` / `listenPort` | `127.0.0.1` / `45677` | Where the proxy listens â€” must match `baseURL` |
| `minimaxHome` | `''` | Empty selects `$MINIMAX_HOME`, else `~/.minimax` |
| `buildEnv` / `region` | `prod` / `en` | Locates the OAuth files under `~/.minimax/auth/â€¦` |
| `oauthTokenURL` | `''` | Empty selects `https://account.minimax.io/oauth2/token` (`cn` uses `.cn`) |
| `planBaseURL` | `https://api.minimax.io/anthropic` | Token Plan upstream |
| `creditBaseURL` | `https://agent.minimax.io/mavis/api/v1/llm/v1` | MiniMax Code credit upstream |
| `quotaURL` | `â€¦/coding_plan/remains` | Where plan quota is read |
| `creditURL` | `â€¦/token_plan_credit` | Fallback-switch / Token Plan credit pool |
| `membershipURL` | `â€¦/commerce/get_membership_info` | Where credits remaining are read |
| `walletURL` | `â€¦/account/query_balance` | Wallet balance |
| `apiKeyEnv` | `MINIMAX_API_KEY` | Credential ref used for the plan side |
| `quotaCacheMs` / `creditsCacheMs` | `60000` | Cache lifetimes for the two probes |
| `refreshSkewMs` | `600000` | Refresh the OAuth grant this far ahead of expiry |

## How it works

```
llm-pi-ai route "minimax-api"  (baseURL â†’ 127.0.0.1:45677/anthropic)
                  â”‚
                  â–¼
        dsh-minimax-credit proxy
          â”œâ”€ read plan quota (cached 60 s)
          â”œâ”€ plan has quota â”€â”€â–¶ api.minimax.io/anthropic        (API key)
          â””â”€ plan spent     â”€â”€â–¶ agent.minimax.io/mavis/â€¦/llm/v1 (OAuth grant)
                  â”‚
                  â–¼
        credits chip â”€â”€ membership endpoint â”€â”€ op_credit_summary
```

## Things worth knowing

- **Credits have a price.** When fallback runs, every turn is billed to your credit balance.
- **MiniMax Code must be signed in on the same machine.** No grant, no fallback â€” the chip turns red and says so.
- **The balance can lag.** The membership endpoint caches on their side; do not use the number for real-time billing
  decisions.
- **The proxy has no auth of its own.** It binds loopback only. Do not expose the port.

## Files

| file | role |
|---|---|
| `index.js` | proxy, quota probe, credit/wallet reads, status route; the grant reader comes from `dsh-minimax-auth` |
| `client.js` | the credits chip |
| `cordis.patch.yml` | the bundle patch with the row and its defaults |
| `locale/{en,th,zh}.json` | UI strings |
| `icon.svg` | plugin icon |
| `test-proxy.mjs` | offline suite (14 cases: decision, fallback, 401, stream, status, credits) |
| `test-live.mjs`, `test-refresh.mjs` | live checks against the real endpoints (not shipped to npm) |

## à¸ à¸²à¸©à¸²à¹„à¸—à¸¢

Token Plan à¸«à¸¡à¸”à¹à¸¥à¹‰à¸§ route `minimax-api` à¸‚à¸­à¸‡ DSH à¸ˆà¸°à¸•à¸²à¸¢à¸—à¸±à¸™à¸—à¸µ (429) à¹à¸•à¹ˆà¹ƒà¸™à¹à¸­à¸› MiniMax Code à¹„à¸¡à¹ˆà¸•à¸²à¸¢ à¹€à¸žà¸£à¸²à¸°à¸¡à¸±à¸™à¸¢à¸´à¸‡à¸œà¹ˆà¸²à¸™
proxy à¸‚à¸­à¸‡à¸•à¸±à¸§à¹€à¸­à¸‡à¸”à¹‰à¸§à¸¢ OAuth grant (`mcode-public`) à¸—à¸µà¹ˆ**à¸«à¸±à¸à¹€à¸›à¹‡à¸™à¹€à¸„à¸£à¸”à¸´à¸•**à¹„à¸”à¹‰ Plugin à¸™à¸µà¹‰à¸—à¸³à¹à¸šà¸šà¹€à¸”à¸µà¸¢à¸§à¸à¸±à¸™à¹ƒà¸«à¹‰ DSH:

- à¹€à¸›à¸´à¸” proxy à¹à¸šà¸š Anthropic-Messages à¸šà¸™ `127.0.0.1:45677` à¹à¸¥à¹‰à¸§à¸Šà¸µà¹‰ `baseURL` à¸‚à¸­à¸‡ route à¸¡à¸²à¸—à¸µà¹ˆà¸™à¸µà¹ˆ
- à¹‚à¸„à¸§à¸•à¸²à¸¢à¸±à¸‡à¸¡à¸µ â†’ à¸ªà¹ˆà¸‡à¸•à¹ˆà¸­à¹„à¸› Token Plan; à¹€à¸ˆà¸­ 429 à¹€à¸žà¸£à¸²à¸°à¹‚à¸„à¸§à¸•à¸²à¸«à¸¡à¸” â†’ à¸ªà¹ˆà¸‡à¸•à¹ˆà¸­à¹„à¸› credit endpoint à¸”à¹‰à¸§à¸¢ OAuth token à¸‚à¸­à¸‡à¹à¸­à¸›
- à¹‚à¸„à¸§à¸•à¸²à¸à¸¥à¸±à¸šà¸¡à¸² â†’ à¸à¸¥à¸±à¸šà¹„à¸›à¹ƒà¸Šà¹‰ Token Plan à¹€à¸­à¸‡à¸­à¸±à¸•à¹‚à¸™à¸¡à¸±à¸•à¸´ (à¹„à¸¡à¹ˆà¸•à¹‰à¸­à¸‡à¸ªà¸¥à¸±à¸šà¹‚à¸¡à¹€à¸”à¸¥/à¹€à¸£à¸´à¹ˆà¸¡ session à¹ƒà¸«à¸¡à¹ˆ)
- à¸¡à¸µà¸›à¹‰à¸²à¸¢ **"à¹€à¸„à¸£à¸”à¸´à¸• N"** à¹ƒà¸•à¹‰à¸Šà¹ˆà¸­à¸‡à¸žà¸´à¸¡à¸žà¹Œ (à¸•à¸±à¸§à¹€à¸¥à¸‚à¸ˆà¸£à¸´à¸‡à¸ˆà¸²à¸ `op_credit_summary` à¸•à¸£à¸‡à¸à¸±à¸šà¸«à¸™à¹‰à¸² Usage à¸‚à¸­à¸‡ MiniMax Code)
- à¸­à¹ˆà¸²à¸™/à¸•à¹ˆà¸­à¸­à¸²à¸¢à¸¸ token à¸‚à¸­à¸‡ MiniMax Code à¹ƒà¸™à¹„à¸Ÿà¸¥à¹Œà¹€à¸”à¸´à¸¡ à¸žà¸£à¹‰à¸­à¸¡ lock/generation à¹€à¸«à¸¡à¸·à¸­à¸™à¹à¸­à¸› â†’ à¹„à¸¡à¹ˆà¹à¸¢à¹ˆà¸‡ token à¸à¸±à¸™

âš ï¸ à¹€à¸„à¸£à¸”à¸´à¸•à¸¡à¸µà¸¡à¸¹à¸¥à¸„à¹ˆà¸² â€” à¹€à¸¡à¸·à¹ˆà¸­ fallback à¸—à¸³à¸‡à¸²à¸™ à¸—à¸¸à¸à¹€à¸—à¸´à¸£à¹Œà¸™à¸ˆà¸°à¸–à¸¹à¸à¸«à¸±à¸à¹€à¸‡à¸´à¸™à¸ˆà¸£à¸´à¸‡, à¸•à¹‰à¸­à¸‡à¸¡à¸µ MiniMax Code à¸¥à¹‡à¸­à¸à¸­à¸´à¸™à¸šà¸™à¹€à¸„à¸£à¸·à¹ˆà¸­à¸‡à¹€à¸”à¸µà¸¢à¸§à¸à¸±à¸™,
à¹à¸¥à¸° proxy à¹„à¸¡à¹ˆà¸¡à¸µ auth à¸‚à¸­à¸‡à¸•à¸±à¸§à¹€à¸­à¸‡ (à¸œà¸¹à¸ loopback à¹€à¸—à¹ˆà¸²à¸™à¸±à¹‰à¸™)

## License

[MIT](LICENSE) Â© 2026 BornCode
