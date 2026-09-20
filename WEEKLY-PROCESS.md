# Howegate Capital — Weekly Experiment Process

The standing instruction for the Claude vs S&P 500 weekly update.
Runs Sunday. Claude does every step; Fraser only approves position changes.

---

## Data source

All prices come from Yahoo's public chart endpoint via `curl`. No account, no API key,
no Python, no library. Global coverage: US, LSE, Xetra, and every other major exchange.

```
curl -s -H "User-Agent: Mozilla/5.0" \
  "https://query1.finance.yahoo.com/v8/finance/chart/<TICKER>?interval=1d&range=10d"
```

**Parsing rule (learned the hard way).** Isolate the arrays explicitly:

```
sed 's/.*"timestamp":\[//; s/\].*//'    # timestamps
sed 's/.*"close":\[//;     s/\].*//'    # closes
```

Never grep for loose 10-digit numbers — the response contains other epoch fields
(`firstTradeDate`, `regularMarketTime`) which silently shift every date by one day.

**Never use `regularMarketPrice` as a value (learned the hard way, Week 5).** It is the
*live* quote, not a dated close — on a Sunday run it can be two days stale in the wrong
direction. Always take the close whose timestamp is the target Friday.

The fallback is allowed only under an explicit date assertion. US equity arrays often carry
a trailing `null` for the current session; when the Friday slot itself is null, you may use
`regularMarketPrice` **only after confirming `regularMarketTime` lands on that same Friday**:

```
# assert before use — if this prints anything but the target Friday, stop
date -u -d @$(grep -o '"regularMarketTime":[0-9]*' f.json | head -1 | grep -o '[0-9]*') '+%Y-%m-%d %a'
```

In Week 5 the equity lines passed this check; `EURGBP=X` did not (its live quote was stamped
Sunday, while the array held the correct Friday close) and the unchecked fallback put the
wrong rate into Rheinmetall. FX arrays are complete — always read FX from the array.

**Currency rule.** Read the `currency` field per ticker. Never assume, never apply a
blanket ÷100.

| Value | Meaning | Action |
|---|---|---|
| `GBp` | pence | divide by 100 |
| `GBP` | pounds | use as-is |
| `USD` | dollars | divide by GBPUSD |
| `EUR` | euros | multiply by EURGBP |

FX tickers: `GBPUSD=X`, `EURGBP=X`.

**FX bars are stamped 23:00 UTC on the _previous_ day** (London midnight), so the Friday
rate carries a Thursday-evening timestamp. Equity bars are stamped on the day itself. Line
the two up by position in the array rather than by reading the dates literally, or the FX
will look a day early and tempt an off-by-one correction that is itself the error.

**The FX arrays carry an extra trailing live bar (Week 9).** On a Sunday run both FX arrays
came back with **11** entries against the equities' 10: the last was a live quote stamped
`Sun 00:41` (GBPUSD) and `Fri 21:28` (EURGBP), not a completed daily bar. Taking the last
element would have used a rate up to two days after the target Friday. Two defences, both
cheap — run them every week:
1. **Count the bars.** Ten weekday FX bars should map 1:1 onto the ten equity bars. If FX has
   more, the extras are live and must be dropped from the end.
2. **Anchor-check against last week's published figures.** The bar you believe is the *prior*
   Friday must reproduce the GBP/USD and GBP/EUR printed on the site last week, to 4 dp. If it
   does, the index you are treating as this week's Friday is right by construction.

**The FMP cross-check is no longer available.** `quote` / `batch-quote-short` now return
`ACCESS DENIED` on the free tier (Premium required), so there is no second price source.
Do not spend a call on it. The `calendar` endpoint still works and remains the source for
earnings dates. With no cross-check available, the date gate below carries the weight.

### Holdings

| Book | Tickers |
|---|---|
| A (ETFs) | NVDA, AVGO, TSM, SMGB.L, CEG, NATO.L, SGLN.L, AMD, IB1T.L |
| B (Equities) | NVDA, AVGO, TSM, MU, AMD, CEG, RHM.DE, AEM, MSTR |

`NATO.L` is quoted in **USD** despite the .L suffix. `SGLN.L` is the only **GBp** line.

---

## Order of operations

### 0. Fetch and validate (before writing anything)
- Pull Friday's dated close for every holding + both FX pairs.
- Confirm the date really is Friday — never accept an undated "last price".
- **Validation gates — if any fail, stop and report instead of publishing:**
  - **Date check: print the dated timestamp of every price actually used — all 14 tickers
    and both FX pairs — and confirm each one reads the target Friday.** This is the gate
    that would have caught the Week 5 EURGBP error. Run it on the values going into the
    calculation, not on the raw response.
  - Unit check: a price ~100× off last week means the currency convention flipped.
  - Move check: anything beyond ±25% week-on-week gets flagged, not auto-published.
  - Arithmetic: holdings + cash must equal NAV to the pound, on the rounded figures.
  - Share counts unchanged unless a trade was approved.
  - Before publishing, re-read the finished copy and confirm every percentage in it appears
    in the computed output, and that superlatives ("best performer", "the standout") are
    checked against the full ranking rather than the names that came to mind.

### 1. Benchmark first — VUAG
Establish the yardstick before looking at our own book, so the write-up can't be
framed to flatter us. Record VUAG's close, index it against the £107.76 inception
anchor, and note the weekly and since-inception move.

### 2. Weekly market overview
**Macro only — no mention of our own tickers.** What actually happened in markets, and why:
rates/Fed, the AI/semis complex, power, defence, gold, crypto. It's fine to name index-moving
mega-caps (Nvidia, Broadcom, AMD) when they're genuinely the market story, but frame it as
market news, not "our position" — no weights, no trim thresholds, no cost basis. Lead the
paragraph with a short bold label ("This week.").

Two paragraphs only, since Week 8's restructure:
1. **This week.** — the macro paragraph described above. Keep it tight; it front-loads the
   page and everything portfolio-specific now lives in "Inside the book" (below), not here.
2. **For a GBP investor.** — cut to one line: the FX move and its read-through to VUAG's own
   price in pounds. This used to be a full paragraph including our own books' weekly % — that
   content moved to "Inside the book" too, to stop the two sections repeating each other.

### 3. Inside the book (the decision narrative)
This is where the portfolio-specific story lives — what happened to our own holdings and
why, then the decision. It replaced the old "This week" pine section (renamed "Inside the
book" at Week 8) and now also carries what used to be the market overview's "Inside our own
book" paragraph, so that paragraph exists in exactly one place, not two.

Three paragraphs:
1. **Performance + drivers.** Both books' weekly moves, and which specific holdings drove the
   gap between them (this is where Rheinmetall, Micron, Constellation, Agnico Eagle etc. get
   named — never in the market overview).
2. **The decision.** See section 5 below — the actual call and reasoning.
3. **Housekeeping.** Largest line per book, trim-threshold status, dealing friction, cash
   levels.

**Check per-book facts before writing them together.** A fact true of one book can be false
of the other because the two books don't hold identical tickers (e.g. Agnico Eagle is
Portfolio B only — Portfolio A's gold line is SGLN.L). Caught at Week 8 before publishing:
a draft claimed Agnico Eagle "added a leg in both books," which was wrong. Same applies to
superlatives like "the book's largest line" or "best AI-complex performer" — check the claim
against each book's own holdings and weights, not just one book's.

**Check superlatives against the sorted list, not against the name you just wrote about.**
Week 9 produced two of these in one pass: Agnico Eagle was called Portfolio B's "best position
since entry" while Strategy sat higher (+40.1% vs +32.0%), and Broadcom was placed fourth by
weight in Portfolio A when it was fifth. Both were written while focused on that one holding.
The fix is mechanical — before publishing, print the full ranking for whichever dimension the
claim uses (weekly move, since-entry P/L, weight), per book, and read the claim off it:

```
sorted(holdings, key=lambda h: -h.weight)      # for "largest line", "second-largest", "Nth"
sorted(holdings, key=lambda h: -h.plPct)       # for "best/worst since entry"
sorted(holdings, key=lambda h: -h.weeklyMove)  # for "best/worst this week"
```

Scope the wording to match what was actually checked: **"in this book"** when ranked within one
book, **"in either book"** only when ranked across both.

### 4. Positions
Both books: current price, £ value, weight, £ P/L (the table has **one** performance column,
P/L — not a separate "Chg" local-currency column; P/L is the number that matters for a GBP
investor and is the number quoted in prose, so keeping only that one avoids the table
implying a different answer than the text).
Percentages are always **computed from the underlying numbers**, never typed by hand.
Recompute NAV, weights, and accrue one week of interest on cash.

Cash interest accrues at **3.75% annualised — the BoE Bank Rate — applied as `× (1 + 0.0375/52)`
each week.** Started Week 5; prior weeks held cash flat at round numbers. Revisit the rate if
Bank Rate moves (next decision 30 Jul 2026).

**Compute the weekly move as its own column** — `(Friday close ÷ prior Friday close − 1)`,
alongside the since-entry column. Two different percentages exist for every holding and they
are easy to confuse: in Week 5 Rheinmetall's **+9.6% since entry** was written up as its
weekly move (actually **+5.4%**), which also produced a false "best performer of the week"
claim — the real answer was Constellation at +9.0%.

The rule that follows: **no percentage appears in the prose that is not in the computed
output.** If the write-up wants to quote a number, that number gets calculated first. This
already governs the tables; it governs the narrative too.

Reconciliation is on the **rounded** figures — the holdings column as displayed must sum to
the displayed NAV. Rounding each line independently can leave a £1 gap against the true
total; adjust the NAV to what the column actually sums to.

### 5. Decision and rule checks
- **Make a real call.** Hold is the right answer when nothing warrants a change — it is not a
  way of avoiding a decision. If the analysis supports a trade, say so with conviction and
  reasoning. Do not default to hold out of caution.
- A three-to-five-year horizon does not trade weekly noise, but it *does* act on a genuine
  change of thesis, a valuation dislocation, or a better use of the cash.
- Forced trim: any position above **12%** is trimmed to cash (mechanical, automatic).
  **Trim back to exactly 12.0% of post-trade NAV, and execute it without asking** — this is the
  one trade that does not wait for Fraser. First fired in Week 9 on Agnico Eagle (12.7% after a
  15.9% week). Solve for the gross proceeds `S` rather than eyeballing it, because the friction
  itself moves NAV:
  `(value − S) = 0.12 × (NAV − friction_rate × S)`
  Then: reduce the share count by `S ÷ price`, scale the cost basis by the **same fraction** of
  shares sold (so the since-entry P/L% is unchanged by the trim — a trim banks cash, it does not
  change how good the investment has been), add `S − friction` to cash, and re-derive every
  weight off the new NAV. Friction convention: **0.10% commission + 0.25% FX spread** on a
  non-UK line, no UK stamp duty.
- The 10% cap applies **at purchase**, not to drift.
- Any *discretionary* buy/sell/rebalance is **proposed to Fraser and held until he replies**.
  Propose it as a recommendation, not a menu — state what to do and why.
- **The site never carries proposal language (added Week 13, hard rule).** When a discretionary
  trade is identified, publish index.html as if the call were simply "hold" — no "we're
  proposing", no "pending", no description of the idea anywhere in the site copy (weekNote,
  portfolio summaries, holding notes, watchlist). The proposal is raised **only in the chat
  reply to Fraser**, closed with an obvious, clearly-separated one-line yes/no question (see the
  trade-proposals-yes-no memory). Do not push a commit that narrates a live proposal to the
  public site. Once Fraser replies yes: execute the trade (recompute shares, cost basis, cash,
  NAV, weights and friction) and republish describing it in the past tense as done — "we
  initiated...", "we trimmed..." — matching the Week 8 Rheinmetall-trim execution style, still
  under the same week number, not a new week. If he replies no, it doesn't appear on the site;
  it can go back on the watchlist.
- Write one honest paragraph on what the week cost or gained us, and the reasoning behind
  holding or trading.

### 6. Watchlist
Update the names under consideration. **2–3 entries** (trimmed from 2–4 at Week 8, now the
section is higher up the page and more compact), each with its theme and a one-sentence
monitoring thesis. Reasoning only — never targets, never recommendations. Add names that fit
existing themes; retire ones whose case has broken, or is redundant with another entry
already on the list (e.g. two "AI platforms" names making near-identical points).

### 7. Key events / upcoming catalysts
The week ahead: earnings dates, macro releases, central bank decisions, sector events.
Keep it to what could actually move our holdings.

### 8. Publish
- Update `index.html` only — preserve all existing layout, classes and structure.
- Advance the week number and date; update the chart series.
- Verify in the browser: no console errors, numbers reconcile.
- `git add . && git commit && git push` → Cloudflare deploys to howegatecapital.com.

### Page order (set at Week 8 restructure)
Hero → Market overview (macro only) → Scorecard → **Inside the book** (the decision
narrative, section 3 above) → Week ahead (catalysts, now its own section) → Performance
chart → Holdings table + Position & reasoning → Theme allocation → Watchlist → Methodology
(moved to the very end — least urgent, evergreen reference material).

Section backgrounds alternate white/cream/pine to match; Scorecard and Inside the book sit
back-to-back on pine as one visual "results + story" block, which is intentional.

---

## Compliance guardrails

- No price targets, no upside/downside percentages, no buy/sell language.
- Watchlist framed as a research log.
- Keep the student-research-project label and the not-investment-advice lines.
- Valuation updates publish automatically. **Position changes require Fraser's yes.**

---

## What Fraser has to do

Nothing weekly, except reply yes/no if a trade is proposed.

Everything else — fetching, converting, validating, writing, publishing — is automatic.
