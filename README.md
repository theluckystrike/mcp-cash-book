# mcp-cash-book

<!-- mirror-seo:start -->

**MCP server for double entry bookkeeping and a cash book general ledger for a small business.** One double-entry ledger derived from the books you already keep, proved to the minor unit.

Works with Claude Desktop, Claude Code, Cursor and any Model Context Protocol client. Runs on your own machine, or hosted with no install.

## Install

**Hosted, nothing to install.** Get a token from <https://mcp.zovo.one/mcp/connect> (the connect page) or <https://mcp.zovo.one/mcp/token> (the same token as JSON); a free anonymous one is issued on the spot and a Pro key works the same way. Then point an MCP client at `https://mcp.zovo.one/mcp/cash-book` over streamable-http and send the token as `Authorization: Bearer <token>`.

If your client cannot set headers, put the token in the path instead: `https://mcp.zovo.one/mcp/cash-book/t/<token>`. Both forms work. The bare URL with no token answers 401 on `tools/call`, so the token is not optional.

**Claude Desktop, one click.** Download `cash-book.mcpb` from the [latest release](https://github.com/theluckystrike/mcp-servers/releases/latest) and double-click it.

**From source.** The mirror is self-contained: every `@theluckystrike/*` dependency is vendored, so a fresh clone builds with no extra setup.

```sh
git clone https://github.com/theluckystrike/mcp-cash-book.git
cd mcp-cash-book
npm install && npm run build
```

Then point your client at the built entry point:

```json
{
  "mcpServers": {
    "cash-book": {
      "command": "node",
      "args": ["/absolute/path/to/mcp-cash-book/dist/index.js"]
    }
  }
}
```

> `@theluckystrike/mcp-cash-book` is **not published on npm yet**, so an `npx -y @theluckystrike/mcp-cash-book` command will fail. The three paths above are the working ones and each is exercised by CI.

![cash-book demo](https://raw.githubusercontent.com/theluckystrike/mcp-servers/main/assets/demo-cash-book.gif)

Read-only mirror of [mcp-servers/servers/cash-book](https://github.com/theluckystrike/mcp-servers/tree/main/servers/cash-book). See [MIRROR.md](MIRROR.md).

<!-- mirror-seo:end -->

One double-entry ledger over the books you already keep. It reads your invoices, credit notes, purchase orders, deposits, expenses, bank import and fixed asset register, and derives a debit and a credit for every movement in a period: revenue and VAT output from the invoices, receivables and the payments that clear them, deposits held as the liability they are, expenses by category with the VAT taken out of the gross, fixed assets and their monthly depreciation. It proves the trial balance sums to zero to the minor unit, and when it does not it names the document whose own figures do not add up. It writes nothing back into any of those books, and there is no way to type an entry into it: every line carries the server, the document id and the date it came from, so any figure can be walked back to the page it was printed on.

npm publish for `@theluckystrike/mcp-cash-book` is pending, so `npx -y @theluckystrike/mcp-cash-book` returns 404 today. Until then, the `.mcpb` one-click bundle or a clone+build is the working path.

## Install

Claude Desktop, `~/Library/Application Support/Claude/claude_desktop_config.json` (Windows: `%APPDATA%\Claude\claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "cash-book": {
      "command": "npx",
      "args": ["-y", "@theluckystrike/mcp-cash-book"]
    }
  }
}
```

Claude Code:

```sh
claude mcp add cash-book -- npx -y @theluckystrike/mcp-cash-book
```

Cursor: `~/.cursor/mcp.json` (or `.cursor/mcp.json` in a project), same entry as Claude Desktop.

This server is only useful next to the servers that own the books: `mcp-invoice`, `mcp-billing-docs`, `mcp-deposits`, `mcp-expense-tracker`, `mcp-bank-statement` and `mcp-asset-register`. Every one of them is optional. A store that is not installed is simply absent from the ledger; a store that is installed and unreadable is reported as unreadable, and never read as an empty one.

## Tools

| tool | what it does |
| --- | --- |
| `ledger_build` | Derives the ledger for one period in one currency and registers the period |
| `period_delete` | Removes one built period from the register and gives its free-tier slot back |
| `trial_balance` | Totals the debits and the credits and proves they are equal to the minor unit |
| `ledger_lines` | Lists the lines, filtered by account, source server, source document or date |
| `month_close` | Lists what the month leaves unposted or inconsistent, then closes it with a snapshot |
| `ledger_export_csv` | Returns the lines as RFC 4180 CSV, one row per leg |
| `ledger_report` | Movement and balance per account, with the purchase commitments and the exceptions |
| `license_status` | Free or Pro, and where to upgrade |
| `license_activate` | Activates a Pro key, verified offline |

Accounts: `cash`, `receivables`, `revenue`, `vat_output`, `vat_input`, `expenses:<category>`, `deposits_held`, `fixed_assets`, `accumulated_depreciation`, `depreciation_expense`, and `purchase_commitments` as a memo that is never posted.

## Free vs Pro

| | Free | Pro |
| --- | --- | --- |
| `trial_balance` | unlimited | unlimited |
| `ledger_lines` | unlimited | unlimited |
| `ledger_build` | 3 periods a calendar month | unlimited |
| Rebuilding a period already built | free | free |
| Rebuilding one with nothing changed | refused, names the row | refused, names the row |
| `period_delete` | unlimited | unlimited |
| `month_close` | - | yes |
| `ledger_export_csv` | - | yes |
| `ledger_report` | - | yes |

The trial balance is free because it is the only question this server exists to answer. A bookkeeper who cannot check that the books add up has no reason to trust anything else here.

`ledger_lines` already returns every field `ledger_export_csv` does, `bank_ref` included: the export is a formatting convenience, an RFC 4180 file with those same fields as columns, not new data. Round 29 (`data/user_value_r29.json`, prompt 6) measured a model take the Pro refusal on `ledger_export_csv` correctly, then hand-build a substitute CSV from `ledger_lines` and drop `bank_ref` anyway. The gate text now says this plainly, so a client relays `ledger_lines` instead of reassembling one.

Get Pro: https://mcp.zovo.one/buy/cash-book (one-time, lifetime, verified offline).

## The measured insight

On the worked month in `test/_client.mjs`, four of the five bank rows are the same money as a document that was already posted: 1,375,300 of the 1,380,300 minor units of cash movement, 99.6 percent, appear in both books. Posting the bank import as well as the documents, which is the obvious way to build a cash book and the way most spreadsheets do it, moves the cash balance from -10,543.00 to -21,111.00 EUR. It does not look wrong. Every line is individually plausible, the account is off by almost exactly a factor of two, and the trial balance still comes to zero, because each duplicated receipt brought its own contra with it.

So the bank import posts nothing here. It is matched to the posted cash movements as evidence, and the leftovers are the whole point: on that month exactly 7,500 minor units of the bank total, one unexplained withdrawal, is information the documents did not already have. That is the 0.4 percent worth importing a statement for, and it is the part a duplicate-heavy ledger buries.

## Privacy

All data stays on your machine. The ledger is derived on each call from the sibling servers' own data directories under `${XDG_DATA_HOME:-~/.local/share}/mcp-servers/`, and this server writes only its own `cash-book/periods.json` and `cash-book/closes.json`. There is no network call anywhere in `src`, and no telemetry. License keys are verified offline.

Built by theluckystrike. https://github.com/theluckystrike
