# 0xDNX DHIP Richlist — WordPress plugin

![0xDNX DHIP Richlist — live dashboard](assets/live-dashboard.png)

**Live holder rankings** for Wrapped Dynex (0xDNX) in the DHIP v2 pool on Ethereum — who holds how much, what share of the pool they represent, and what moved recently. Holders, researchers, and community members use [logicencoder.com/0xdnx-dhip-v2-richlist/](https://logicencoder.com/0xdnx-dhip-v2-richlist/) to track concentration and whale activity without running an indexer or parsing explorer pages line by line.

Pool share shifts when large wallets deposit or exit — a static export is wrong within hours. A Python indexer watches the DHIP contract, recalculates ranks and percentages, and pushes updates into WordPress; the public page polls on a timer so stats, recent flows, and the holder table stay aligned during active days without manual reload or opening holder tabs on a block explorer.

## Tech stack

| Layer | Technologies |
|-------|--------------|
| WordPress plugin | PHP (single-file ~4.9k LOC), shortcodes, wp-admin UI, WordPress AJAX, WordPress REST API |
| Public UI | HTML, CSS (Mulish, dark theme), JavaScript (live refresh, load-more, nonce handling) |
| WordPress database | MySQL — custom tables for holders, stats, transactions, daily snapshots, history |
| Chain indexer | Python 3, web3.py, requests, sqlite3, orjson, threading/queue workers |
| Ingest | Authenticated REST push (X-API-Key), balance/percentage math in Python before persist |
| SEO | Static snapshot HTML, JSON-LD structured data, bot vs human user-agent routing |
| Caching | LiteSpeed-friendly no-cache on live poll routes; snapshot files for crawlers |
| Networking | Cloudflare tunnel, Ethereum JSON-RPC, Etherscan links |
| Hosting | WordPress on shared hosting; Python indexer on self-hosted Linux server |

## Summary statistics

The **stats band** above the table is the denominator for every percentage in the rankings. Without it, “3.42% of pool” is hard to interpret; with it, you see the whole picture at a glance. The band refreshes on the same interval as the table so stats and rows never disagree after a large move.

Each stat box is computed from the latest holder rows and transaction history in WordPress — not re-derived on every page view from chain RPC:

| Stat | What it tells you |
|------|-------------------|
| **Total holders** | Unique addresses with a non-zero DHIP balance (deduplicated count). |
| **Total 0xDNX in DHIP** | Aggregate wrapped supply sitting in the v2 pool contract. |
| **Last updated** | Timestamp of the last successful indexer push — confirms data freshness. |
| **Top 10 holders** | Combined pool share of the ten largest ranks (capped at 100%). |
| **Top 50 holders** | Combined pool share of the fifty largest ranks — broader concentration signal. |
| **Transactions count** | Total indexed deposit/withdraw events stored in the plugin database. |
| **Average balance** | Total pool supply divided by unique holder count. |
| **Whale holders (>1%)** | Count of addresses holding **≥1%** of pool supply, plus their combined share. |
| **24h activity** | Number of deposit/withdraw transactions in the last 24 hours. |
| **Small holders (<0.1%)** | Count of addresses below 0.1% share, plus their combined percentage. |

Whale and small-holder thresholds are fixed in the plugin logic (≥1% and <0.1%) so community readers get consistent labels across sessions.

<img src="assets/summary-stats.png" alt="Summary statistics — holders, pool totals, whales, and 24h activity" width="860" />

## Recent transactions

Rankings show *who holds now*; the **recent activity panel** shows *what changed*. Each row is a deposit into or withdrawal from the DHIP pool:

- **Time** — when the transfer landed on-chain.
- **Direction** — deposit (green) vs withdraw (red) badge.
- **Addresses** — truncated wallet with **Etherscan** link for full hex.
- **TxID** — transaction hash link when the indexer captured it.
- **Amount** — formatted 0xDNX moved with sign (+ deposit, − withdraw).
- **Block** — block number link for on-chain verification.

The panel row count follows wp-admin **Number of transactions to show** (default 5, max 20 in settings UI). During live refresh, new rows can appear without reloading the page when the poll detects a newer transaction timestamp than the one the browser last saw.

Use this panel to spot a single large deposit that will appear in the table on the next refresh, or to confirm a known wallet exited the pool. Researchers often keep it open beside the rankings during governance or listing events.

<img src="assets/recent-transactions.png" alt="Recent transactions — deposits and withdrawals with Etherscan links" width="860" />

## Ranked holder table

The **main table** is the product core — ranks of every tracked holder sorted by balance:

| Column | Meaning |
|--------|---------|
| **Rank** | Position by balance (1 = largest). |
| **Address** | Truncated `0x…` with Etherscan link. |
| **Balance** | Formatted 0xDNX in the DHIP pool (decimal places from settings). |
| **% of pool** | Exact share of total wrapped supply in the contract. |

**Initial page size** comes from **Default display limit** (default 30, overridable per shortcode). **Load more** appends the next batch via AJAX using **Load more amount** (default 20) — no full page reload. The button hides when no ranks remain.

Optional **minimum balance** and **minimum percentage** filters (wp-admin or shortcode) hide small holders from the public table while stats still reflect the full indexed set.

Percentages are computed in the **Python indexer** before push so WordPress serves consistent math; the plugin stores formatted values and ranks, not raw wei recalculated in PHP on every row.

<img src="assets/holder-rankings.png" alt="Ranked holder table — balance and pool percentage by address" width="860" />

## Live refresh without reload

When **Auto refresh** is enabled (default), the front-end JavaScript polls WordPress on a timer — **Refresh interval** in seconds (default 60, min 10, max 3600). Each poll calls `dxdhip_check_updates` with the last seen stats timestamp and current holder count.

If new data arrived since the last poll, the plugin returns refreshed stats HTML, holder rows, and recent transactions; the script replaces those sections in place and briefly shows a status line (“Stats updated!” or “New transactions detected!”).

**LiteSpeed and page cache:** anonymous visitors skip nonce verification on read-only poll endpoints so a cached page with an expired WordPress nonce does not permanently break live updates. Logged-in users still require a valid nonce.

**Nonce recovery:** on HTTP 403 (expired nonce) or after repeated poll failures, the client calls `dxdhip_refresh_nonce` to fetch a fresh nonce and resume polling. If nonce refresh fails, auto-refresh stops with a “please reload the page” message rather than spamming errors.

Multiple shortcode instances on one page each get a unique instance ID so polls and DOM updates do not collide.

## Shortcode embed

Place the richlist on any WordPress page or post:

```
[0xdnxdhip_richlist]
```

Every display knob is also available as a shortcode attribute (defaults come from wp-admin **Settings** when omitted):

| Attribute | Purpose | Default |
|-----------|---------|---------|
| `limit` | Initial holder rows | 30 |
| `load_more_amount` | Rows per Load more click | 20 |
| `show_all` | Show all holders at once (`yes`/`no`) | `no` |
| `balance_decimals` | Decimal places for balances | 2 |
| `percentage_decimals` | Decimal places for percentages | 4 |
| `min_balance` | Hide holders below this balance | 0 |
| `min_percentage` | Hide holders below this % | 0 |
| `auto_refresh` | Enable timed polling (`yes`/`no`) | `yes` |
| `refresh_interval` | Poll interval in seconds | 60 |
| `show_stats` | Show statistics band (`yes`/`no`) | `yes` |
| `show_title` | Show heading above the widget (`yes`/`no`) | `no` |
| `title` | Custom heading text | `0xDNXDHIP Richlist` |
| `show_transactions` | Show recent activity panel (`yes`/`no`) | `yes` |
| `transactions_count` | Recent transaction rows | 5 |

Example for a community “top holders” page with a visible title and wider first page:

```
[0xdnxdhip_richlist limit="50" title="Top 0xDNXDHIP Holders" show_title="yes" balance_decimals="2" percentage_decimals="4"]
```

## Search engines and snapshots

Human visitors get the **interactive UI** described above. Crawlers, AI answer engines, and other bots can receive **pre-rendered snapshot HTML** plus **JSON-LD** so headline stats and top ranks appear in search without executing the live poll loop.

**User-agent routing** inspects the request agent string (including common search and AI crawlers). Bots get the snapshot payload; browsers get the full app — same URL, appropriate response.

The Python indexer pushes snapshot HTML to WordPress on a schedule (approximately every 15 minutes). wp-admin **Snapshot settings** control:

- **Snapshot file path** — where the static HTML file lives on disk (default under WordPress `cache/`).
- **Reverse DNS verification** — optional Googlebot/Bingbot IP verification before serving snapshots.
- **Live status panel** — file exists, size, and last update time (auto-refreshes every 30 seconds in settings).

Authenticated REST endpoints accept holder pushes and snapshot uploads from the indexer using the **API key** configured in settings — the public site never exposes that key.

## WordPress admin

Top-level **0xDNXDHIP** menu in wp-admin — operators manage display settings, inspect data health, and export history without editing PHP.

### Admin dashboard

The **dashboard** home shows headline holder stats at a glance, a **Refresh Data** button to trigger a manual refresh event after deploys, and a **preview of recent transactions** with a link to the full history browser. Use it after an indexer restart to confirm row counts moved and the last ingest timestamp is fresh.

![wp-admin dashboard — holder stats and recent transaction preview](assets/admin-dashboard.png)

### Settings

**Settings** is where operators tune the public page without touching code:

- **API key** — shared secret for indexer REST pushes; **Generate New Key** rotates it instantly.
- **Default display limit** / **Load more amount** — pagination defaults for front-end and admin previews (5–1000 / 5–500).
- **Show all holders** — disable pagination and render the full ranked list at once.
- **Show title in frontend** — optional “0xDNXDHIP Richlist” heading above the widget.
- **Balance decimals** / **Percentage decimals** — formatting precision (0–18 / 0–10).
- **Minimum balance filter** / **Minimum percentage filter** — hide small ranks from the public table.
- **Auto refresh** / **Refresh interval** — enable timed polling and set seconds between checks.
- **Show statistics panel** / **Show recent transactions** / **Number of transactions to show** — toggle and size the two public bands above the table.
- **Refresh Data Now** — same manual refresh as the dashboard button.
- **Snapshot settings** — path, DNS verification toggle, and live snapshot file status (documented above).

Changes apply on the next page load; live visitors pick up new intervals and panel toggles after their next poll cycle.

### Database tools

**Database tools** lists every custom table — holders, transactions, stats, daily snapshots, balance/rank/percentage history, significant moves, and more — with **row counts** and **storage size**. When the public table feels slow or stats look stale, check here first: a zero row count on `holders` means ingest never landed, not a front-end bug.

The same screen includes **Database maintenance** actions:

- **Duplicate address check** — lists any holder rows that share an address (would skew percentages); **Remove duplicates** keeps the highest-balance row per address.
- **Clean old transactions** — deletes transaction records older than 30 days to cap database growth.
- **Optimize database tables** — runs `OPTIMIZE TABLE` across all plugin tables.
- **Backup database** — writes a timestamped SQL dump to `wp-content/backups/` with a download link.

Run duplicate check after a bad import or indexer bug before announcing rankings publicly.

![Database tools — table row counts and sizes](assets/database-tools.png)

![Database maintenance — transaction statistics and duplicate address check](assets/database-maintenance.png)

### Transaction browser

The **transaction browser** is the full paginated history — every deposit and withdrawal the indexer captured:

- **Export CSV** for spreadsheets or sharing with analysts.
- **Etherscan-linked** addresses and transaction IDs.
- **Deposit / withdraw badges** per row.
- **Pagination** through the full dataset when the dashboard preview is not enough.

![Transaction browser — full history with CSV export](assets/transaction-browser.png)

Private code: [dynex-0xdnx-dhip-richlist-plugin](https://github.com/logicencoder/dynex-0xdnx-dhip-richlist-plugin) · indexer [dynex-0xdnx-dhip-richlist-monitor](https://github.com/logicencoder/dynex-0xdnx-dhip-richlist-monitor)

Guide: [Wrapped Dynex richlist explained](https://logicencoder.com/wrapped-dynex-richlist-0xdnx-dvhip-v2/)

See [REPOS.md](REPOS.md).

---

**Made by [Logic Encoder](https://logicencoder.com)** · [GitHub](https://github.com/logicencoder) · [Contact](https://logicencoder.com/contact/)
