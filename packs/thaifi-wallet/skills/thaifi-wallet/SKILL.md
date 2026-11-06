---
name: thaifi-wallet
description: Pair a non-custodial ThaiFi wallet with the thaifi CLI, check balances and spending limits, top up via PromptPay, transfer, pay x402/MPP services, and issue TIP-20 / ERC-1155 tokens. Use when the user mentions ThaiFi, pathTHB, THCFI, THCOC, x402 payments, or setting up an agent wallet.
---

# ThaiFi Wallet CLI — Skill for AI Agents

## Overview

ThaiFi Wallet CLI (`thaifi`) is a command-line wallet for the **ThaiFi** blockchain that lets an AI agent hold, manage, and spend funds non-custodially. The agent pairs its own key with the web wallet (**https://wallet.thaifi.com**) — the user approves once, and the key becomes an **on-chain access key** with a spending limit enforced by the chain itself.

- Network: ThaiFi, chain ID `17` (Tempo-compatible). Gas is paid in **pathTHB** `0x20c0000000000000000000000000000000000000` (6 decimals) by default; THCFI/THCOC can also pay gas (fee-token transactions, swapped via the protocol FeeAMM).
- Tokens (all TIP-20, 6 decimals) — supported by `balance`/`transfer` via `--token <symbol|address>`:

| Symbol | Address | Notes |
|--------|---------|-------|
| pathTHB | `0x20c0000000000000000000000000000000000000` | fee/gas token (default) |
| THCFI | `0x20c000000000000000000000c82102FFe7064362` | ThaiCoin ThaiFi (also usable as fee token via `setUserToken`) |
| THCOC | `0x20C0000000000000000000007c24a0c628e8A940` | ThaiCoin OpenCraft (also usable as fee token via `setUserToken`) |

- RPC: `https://rpc.thaifi.com` · Explorer: `https://exp.thaifi.com`
- Custody: the agent's P256 key lives only in `~/.thaifi/store.json` (chmod 600). The server never sees it. Spending is capped on-chain per key, **per token** (defaults at approval: 100 pathTHB / 5,000 THCFI / 5,000 THCOC per 30 days, adjustable; revocable any time from the web wallet under "Authorized Apps").

## Setup

1. Install: `npm install -g thaifi-wallet-cli` (0.3.6+; or run one-off with `npx thaifi-wallet-cli@latest <command>`)
2. Login (pairs with the web wallet) — pick ONE pattern for your environment:

```bash
# Desktop with a browser — the browser opens the approval page itself:
thaifi login

# Headless machine (SSH/server) — foreground; copy the PAIR_URL line it prints:
thaifi login --no-browser 2>&1 | tee /tmp/pair.log

# Background launcher (bot/supervisor captures stdout) — write to a log file:
nohup thaifi login --no-browser > /tmp/pair.log 2>&1 &
grep PAIR_URL /tmp/pair.log   # post this link to the user immediately
```

**URL reliability (CLI 0.3.6+):** the approval link is ALWAYS also written to
`~/.thaifi/pair.log` (chmod 600). If stdout was lost, recover it with
`cat ~/.thaifi/pair.log`. Re-running `thaifi login` while a pairing is still
waiting is safe — it reuses the SAME pairing/URL (never creates a duplicate)
until it is approved or expires server-side.

**Empty wallets approve fine:** brand-new wallets hold no tokens, so the
approval page automatically adds 0.01 THCFI of network gas the moment it opens
(no user action). The user does NOT need to top up before approving — funding
comes after, via `thaifi deposit`.

3. Verify pairing:

```bash
thaifi whoami
```

The user approves the pairing at `https://wallet.thaifi.com/pair?id=…&code=…`. The page walks them through up to three steps, with a "Connect an AI agent" panel naming your machine:

1. **Sign in** (LINE or email code).
2. **Wallet setup**: create or restore a wallet, only if this browser has none yet. The page says "Wallet setup required" and explains why.
3. **Approve**: set the spending limits, compare the **Matching code** shown right above the Approve button, and approve with passkey or PIN. A brand-new wallet first needs a small balance for the approval's network fee: the page offers a cloud backup and a PromptPay top-up (from 6 THB) right there, then continues by itself.

The code is a **visual check only**: there is no field to type it into, and it appears only at the Approve step. Default spend limits at approval: **100 pathTHB**, **5,000 THCFI**, **5,000 THCOC** per 30 days — the user can adjust each one.

What to tell the user when you send the link (adapt the language, keep the meaning):

> Open this link to connect me to your ThaiFi wallet: ‹PAIR_URL›
> Before you approve, check that the **matching code** on the page is **‹the 6-digit code `thaifi login` printed›**. There's nothing to type. If it's your first time on this browser, you'll be asked to sign in and set up a wallet (passkey or PIN) first; the approve step comes after that. Tell me when you're done.

## Commands

```bash
thaifi login [--no-browser]          # pair with the web wallet (one-time)
thaifi whoami                        # account, agent key, balances, remaining limits
thaifi tokens                        # list supported TIP-20 tokens (symbol / decimals / address)
thaifi balance [--token <symbol|address>]   # balances (default: all supported tokens)
thaifi transfer <to> <amount> [--token <symbol|address>]   # TIP-20 transfer (default: pathTHB), limit enforced on-chain
thaifi deposit --amount <THB> [--token THCFI|THCOC] [--ref <id>]   # PromptPay QR top-up (1 THB = 1 token) and wait until delivered
thaifi request <url> [options]       # HTTP request with automatic x402/MPP payment
thaifi services [--search <query>]   # discover services from the MPP registry
thaifi logout                        # remove the local key (revoke on-chain via the web wallet)
```

## Rules for agents

- ALWAYS run `whoami` after login to confirm pairing, balance, and the remaining limit.
- **Never ask the user to type or "enter" the pairing code** — tell them to CHECK it matches the matching code shown on the approval page. If it doesn't match, they must not approve.
- **Don't tell the user to "click approve" right away.** A first-time user must sign in and set up a wallet before the Approve button exists. Send the link and the code, then poll `thaifi whoami` — don't promise to check "immediately".
- **Login discipline:** run `thaifi login` ONCE per pairing. Never re-run it in a loop while waiting for approval — if it's already waiting, a re-run simply returns the same link (no duplicate), so the right move is: post `PAIR_URL` (from stdout or `~/.thaifi/pair.log`) to the user, then poll `thaifi whoami` until it succeeds.
- Check the remaining limit before large transfers; transfers beyond the limit fail on-chain with `SpendingLimitExceeded` (report this clearly and stop).
- Use `request --dry-run` before expensive paid calls; respect `--max-spend`.
- NEVER print or log the private key or the contents of `~/.thaifi/store.json`.
- If a call returns 401/403, the pairing may have been revoked by the user — ask them to re-run `thaifi login`.
- Expiry: keys expire (default 90 days). Re-run `thaifi login` to re-pair.

## Funding — PromptPay QR top-up (`thaifi deposit`, CLI 0.4.0+)

The agent requests a **PromptPay QR** for its own wallet and a human scans it
with any banking app. No API key: the backend verifies on-chain (AccountKeychain
precompile) that the agent's access key belongs to a wallet and credits THAT
wallet — the CLI never sends a recipient. 1 THB = 1 token, minted automatically
after payment:

```bash
thaifi deposit --amount 50                  # 6–10,000 THB → THCFI (default)
thaifi deposit --amount 50 --token THCOC    # different payout token
thaifi deposit --ref <referenceNo>          # resume watching an existing order
```

The QR is saved to `~/.thaifi/qr-<referenceNo>.png` (chmod 600, also rendered
in the terminal and opened on desktop) and machine-parsable lines are printed:

```
REFERENCE=<id>
QR_FILE=~/.thaifi/qr-<id>.png
AMOUNT=50.00 THB -> THCFI
STATUS=delivered
TX=0x…          (mint tx — verify on https://exp.thaifi.com)
```

Show the QR (terminal or file) to the user; the CLI polls until delivered.
Orders expire after 15 minutes — paying late still delivers. Limits: 6–10,000
THB per order, 50,000 THB per wallet per 24h, max 5 pending orders.

Alternative funding: transfer pathTHB/THCFI/THCOC directly to the account
address (`thaifi whoami` / `thaifi fund`, or the web wallet's Deposit tab at
https://wallet.thaifi.com/deposit). Transferring THCFI/THCOC still needs
**pathTHB for gas** — keep a small pathTHB balance (a transfer costs well
under 0.01 pathTHB).

## Agent auth — free APIs (ThaiFiAgent)

Free data APIs can require agent **identity** instead of payment (humans log
in with OTP; agents prove their on-chain access key). `thaifi request`
handles it automatically: on `401/403` with
`WWW-Authenticate: ThaiFiAgent realm="…", challenge="…"` it signs the
challenge with your access key and retries — no flags, no payment. The
server verifies the signature against the AccountKeychain precompile, so
revocation/expiry in the web wallet applies immediately, and the API learns
your pseudonymous `{account, keyId}`.

Try it (reference endpoint):

```bash
thaifi request https://mpp.thaifi.com/whoami
```

Server developers: the reference implementation (challenge issuance +
on-chain verification via the Signature Verifier precompile) is in the
ThaiFi mpp repo, `src/agentAuth.ts` — copy that pattern into your own API.

## x402/MPP payments

`thaifi request` behaves like curl with automatic machine payments (MPP / x402):

```bash
thaifi request --dry-run -X POST -H 'content-type: application/json' -d '{"prompt":"..."}' <SERVICE_URL>
thaifi request -X POST -H 'content-type: application/json' -d '{"prompt":"..."}' <SERVICE_URL>
thaifi request --max-spend 1 <URL>   # cap automatic payment at $1
```

On `402 Payment Required` the CLI parses the MPP challenge
(`WWW-Authenticate: Payment …`), pays the quoted pathTHB amount on-chain from
the spend limit (memo-bound to the challenge), and retries once with the
payment credential.

## Use as an MCP server (`thaifi mcp`, CLI 0.5.0+)

Instead of curl-ing endpoints, run the CLI as a **stdio MCP server** that
exposes every paid service in the live catalog as an MCP tool and pays each
call automatically with the paired access key (the on-chain spending limit
still applies). Works with Claude Desktop, ZCode, Cursor, or any MCP client:

```json
{ "mcpServers": { "thaifi": { "command": "thaifi", "args": ["mcp"] } } }
```

- Tools appear from the live registry automatically (e.g. `thaifi_legal_search`,
  `thaifi_legal_ask`, `thaifi_iapp`, `thaifi_qwen`, `thaifi_qwen3_8`,
  `thaifi_course`, `thaifi_token_create`, `thaifi_voucher_create`) — no config
  update needed when the server adds services.
- Every call is capped by `THAIFI_MCP_MAX_SPEND` (USD per call, default 5)
  **on top of** the on-chain access-key limit.
- Requires pairing first: run `thaifi login` once (see Setup), then restart
  the MCP client. If not paired, tools fail with a clear message instead of
  paying.

## Services

First-party paid services live on the ThaiFi MPP server (`mpp.thaifi.com`).
**Read the live catalog from the server itself — do not rely on a list copied
here** (prices and services change):

```bash
thaifi services                                          # list services from the registry
curl https://mpp.thaifi.com/services                     # JSON catalog (canonical)
curl https://mpp.thaifi.com/llms.txt                     # agent-readable docs with schemas
curl https://mpp.thaifi.com/openapi.json                 # OpenAPI + x-payment-info offers
```

Payment is automatic: on `402 Payment Required` the CLI pays the quoted THCFI
amount from the agent's access key (on-chain, memo-bound to the challenge) and
retries once. Live usage/sales stats: `GET https://mpp.thaifi.com/stats`.

### Example: Generate an image (iapp — Thai text rendering)

When a user asks you to create an image (e.g., "สร้างรูปแมวส้มสวมหมวกเชฟ"), run:

```bash
thaifi request https://mpp.thaifi.com/iapp/generate \
  -X POST -H 'content-type: application/json' \
  -d '{"prompt": "แมวส้มสวมหมวกเชฟ กำลังปรุงอาหารในครัวญี่ปุ่น", "text": "เชฟแมว", "size": "1024x1024"}'
```

**What happens:** the CLI calls the endpoint → `402` with the price → automatic
on-chain payment → retry with proof → the service responds (images take ~25-60s).
The response's `image_base64` is decoded and saved as a PNG file in the current
directory automatically; the CLI prints file name, dimensions, mime type and
generation time.

**Rule of thumb:** Thai text inside the image → `iapp` (exact Thai rendering,
fonts via `thaifi request https://mpp.thaifi.com/iapp/fonts`); everything else
→ `qwen` (photorealistic, fast). For each service's exact body schema see
`https://mpp.thaifi.com/llms.txt`.

### Legal Q&A — กฎหมายสินทรัพย์ดิจิทัล / การชำระเงินไทย

When the user asks about Thai digital-asset or payment law (พ.ร.ก. สินทรัพย์ดิจิทัล
2561 including the 2568 offshore-platform amendment, พ.ร.บ. ระบบการชำระเงิน
2560, related MOF notifications), use the legal knowledge-base services:

```bash
# free: see which documents/sections are covered
curl https://mpp.thaifi.com/legal/outline

# search: full text of matching มาตรา/ข้อ + citations (0.1 THCFI)
thaifi request https://mpp.thaifi.com/legal/search -X POST \
  -H 'content-type: application/json' \
  -d '{"query": "ธุรกิจรับฝากสินทรัพย์ดิจิทัล ต้องขออนุญาตไหม"}'

# ask: retrieval + LLM answer with citations (0.5 THCFI) — best for plain questions
thaifi request https://mpp.thaifi.com/legal/ask -X POST \
  -H 'content-type: application/json' \
  -d '{"question": "ถ้าทำ bridge รับฝาก USDT จากลูกค้าแล้ว mint token ให้ ผิดกฎหมายอะไร"}'
```

**What happens:** `search` returns the actual gazette text per มาตรา/ข้อ (exact
`มาตรา N` queries resolve deterministically; Thai trigram full-text otherwise) —
read the returned `body`/`analysis` yourself and answer with citations. `ask`
adds a Qwen3.8 synthesis that cites its sources; answers always end with the
not-legal-advice disclaimer. Prefer `search` when you want the raw ตัวบท to
reason over yourself; use `ask` for one-shot plain questions.

### Launchpad: create a token or voucher/ticket NFTs (one prompt is enough)

When the user asks to **create their own token** ("ทำเหรียญของร้านฉันชื่อ …"),
collect `name`, `symbol`, `mint_amount` (integer — decimals are applied for
you) and optionally a square logo (PNG/WebP ≤1024×1024), then:

```bash
thaifi request https://mpp.thaifi.com/tokens/create \
  -X POST -H 'content-type: application/json' \
  -d '{"name":"Cafe Moon","symbol":"MOON","mint_amount":10000,"logo_base64":"…"}'
```

1,000 THCFI. Your paying wallet becomes the token's full admin (non-custodial
handover); the created TIP-20 appears on <https://tokens.thaifi.com>.

When the user asks for **vouchers or event tickets as NFTs** ("ทำตั๋วงานวิ่ง /
คูปองแลกของรางวัล"), ask them the required questions first (see
`llms.txt` §voucher-create: kind ticket/voucher, tier, quantity, event
date/venue or redemption value/instructions), quote with the free
`POST https://mpp.thaifi.com/vouchers/quote` (price = max(1,000, qty × 0.20)
THCFI), then create with `POST https://mpp.thaifi.com/vouchers/collections`.
Each order deploys a dedicated ERC-1155 contract, mints the quantity to your
wallet and hands you contract ownership. Gallery: <https://tokens.thaifi.com>.

## Troubleshooting

- `Not paired` → run `thaifi login`.
- `Cannot see the approval URL` (background/lost stdout) → `cat ~/.thaifi/pair.log` — the `PAIR_URL` line is always written there (CLI 0.3.6+).
- `SpendingLimitExceeded` → the user must raise the limit from the web wallet (Authorized Apps) or fund more frequently.
- `KeyExpired` / revoked → re-run `thaifi login`.
- RPC errors → check `https://rpc.thaifi.com` is reachable; override with `THAIFI_RPC_URL`.
