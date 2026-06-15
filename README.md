# 0xDNX DHIP Richlist — WordPress plugin

![0xDNX DHIP Richlist — live dashboard](assets/live-dashboard.png)

**Live holder rankings** for Wrapped Dynex (0xDNX) in the DHIP v2 pool on Ethereum — who holds how much, what share of the pool they represent, and what moved recently. Holders, researchers, and community members use [logicencoder.com/0xdnx-dhip-v2-richlist/](https://logicencoder.com/0xdnx-dhip-v2-richlist/) to track concentration and whale activity without running an indexer or parsing explorer pages line by line.

The hero screenshot shows the full public page in one view: **summary statistics** across the top, **recent deposits and withdrawals** in the middle band, and the **ranked holder table** below with load-more pagination. Dark Mulish typography keeps long addresses readable; live refresh runs on a timer so rankings update during active trading days without manual reload.

## Tech stack

| Layer | Technologies |
|-------|--------------|
| WordPress plugin | PHP (single-file ~4.9k LOC), shortcodes, wp-admin UI, WordPress AJAX, WordPress REST API |
| Public UI | HTML, CSS (Mulish, dark theme), JavaScript (live refresh, load-more, nonce handling) |
| WordPress database | MySQL — custom tables for holders, stats, transactions, daily snapshots |
| Chain indexer | Python 3, web3.py, requests, sqlite3, orjson, threading/queue workers |
| Ingest | Authenticated REST push (X-API-Key), balance/percentage math in Python before persist |
| SEO | Static snapshot HTML, JSON-LD structured data, bot vs human user-agent routing |
| Caching | LiteSpeed-friendly no-cache on live poll routes; snapshot files for crawlers |
| Networking | Cloudflare tunnel, Ethereum JSON-RPC, Etherscan links |
| Hosting | WordPress on shared hosting; Python indexer on self-hosted Linux server |

## Summary statistics

The **stats band** above the table is the denominator for every percentage in the rankings. Without it, “3.42% of pool” is hard to interpret; with it, you see the whole picture at a glance.

Typical headline metrics:

- **Total holders** — unique addresses with a non-zero DHIP balance.
- **Total 0xDNX in pool** — aggregate wrapped supply sitting in the v2 pool contract.
- **Top holder share** — concentration signal for whale dominance.
- **Whale count** — addresses above a configured threshold (community-defined “large holder”).
- **24h deposits / withdrawals** — flow in and out of the pool in the last day; spikes often precede rank shuffles in the table below.

The band refreshes on the same interval as the table so stats and rows never disagree after a large move.

<img src="assets/summary-stats.png" alt="Summary statistics — holders, pool totals, whales, and 24h activity" width="860" />

## Recent transactions

Rankings show *who holds now*; the **recent activity panel** shows *what changed*. Each row is a deposit into or withdrawal from the DHIP pool:

- **Time** — when the transfer landed on-chain.
- **Direction** — deposit (green) vs withdraw (red) badge.
- **Addresses** — truncated wallet with **Etherscan** link for full hex.
- **Amount** — formatted 0xDNX moved.
- **Transaction** — tx hash link for block explorer verification.

Use this panel to spot a single large deposit that will appear in the table on the next refresh, or to confirm a known wallet exited the pool. Researchers often keep it open beside the rankings during governance or listing events.

<img src="assets/recent-transactions.png" alt="Recent transactions — deposits and withdrawals with Etherscan links" width="860" />

## Ranked holder table

The **main table** is the product core — sortable ranks of every tracked holder:

| Column | Meaning |
|--------|---------|
| **Rank** | Position by balance (1 = largest). |
| **Address** | Truncated `0x…` with Etherscan link. |
| **Balance** | Formatted 0xDNX in the DHIP pool. |
| **% of pool** | Exact share of total wrapped supply in the contract. |

**Load more** appends the next page of ranks without a full page reload — default page size is configurable in wp-admin (typically 30 rows). **Auto-refresh** re-polls on an interval so a wallet that just deposited climbs the table during your session.

Percentages are computed in the **Python indexer** before push so WordPress serves consistent math; the plugin does not re-derive balances in PHP on every row.

<img src="assets/holder-rankings.png" alt="Ranked holder table — balance and pool percentage by address" width="860" />

## Live page and search engines

Human visitors get the **interactive UI**: AJAX stats refresh, pagination, timed polling, and nonce-protected load-more requests. Crawlers and AI answer engines can receive **pre-rendered snapshot HTML** plus **JSON-LD** so headline stats and top ranks appear in search without executing the live poll loop.

**User-agent routing** serves snapshots to bots and the full app to browsers — same URL, appropriate payload.

Shortcode **`[0xdnxdhip_richlist]`** embeds stats, recent transactions, table, and refresh behaviour on any WordPress page or post.

## WordPress admin

Top-level **0xDNXDHIP** menu in wp-admin — operators manage display settings, inspect data health, and export history without editing PHP.

### Admin dashboard

The **dashboard** home shows headline holder stats at a glance, a **Refresh Data** button to pull the latest push from the indexer, and a **preview of recent transactions** with a link to the full history browser. Use it after a deploy to confirm row counts moved and the last ingest timestamp is fresh.

![wp-admin dashboard — holder stats and recent transaction preview](assets/admin-dashboard.png)

### Database tools

**Database tools** lists every custom table — holders, transactions, stats, daily snapshots, history — with **row counts** and **storage size**. When the public table feels slow or stats look stale, check here first: a zero row count on `holders` means ingest never landed, not a front-end bug.

![Database tools — table row counts and sizes](assets/database-tools.png)

### Database maintenance

**Maintenance** goes deeper than row counts:

- **Deposit vs withdrawal breakdown** — aggregate flow totals for sanity checks.
- **Transaction date range** — earliest and latest indexed event.
- **Duplicate address check** — confirms holder rows are unique; duplicates would skew percentages in the public table.

Run duplicate check after a bad import or indexer bug before announcing rankings publicly.

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
