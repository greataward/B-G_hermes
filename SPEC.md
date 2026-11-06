# Hermes — Production Data Entry Automation

**Status:** Spec / Phase 0 (investigation)
**Owner:** Krish
**Last updated:** 2026-11-06

## 1. Purpose

Automate manual data entry for the high-speed RFID label line. The
operator scans a lot number and marks a roll complete; Hermes writes the
resulting numbers into the Production Planner, the QC sheet, and Sage.

**Non-goal (for now):** watching the machine or judging label quality.
That is a later phase, after data entry is proven.

### Target end state
No touching the computer while the machine runs. Hermes inputs everything
after the machine stops.

### Resulting data technician job
- Load new lots (scan each lot label for Hermes)
- Change finished rolls
- Print core labels
- Stick labels
- Move rolls to the outside rack

## 2. Architecture principle

Most of Hermes is **deterministic code, not a model.**

Reading CSVs, doing remainder arithmetic, and writing cells are
mechanical. A script does them identically every run and can be audited.
A model re-decides each time.

Use a model for: building this, handling ragged input, and *explaining*
discrepancies. Not for arithmetic, and not for deciding what to post.

**Corollary:** Hermes never guesses. On any mismatch it halts and shows
the conflict to a human as a **pop-up on the production PC**.

## 3. Systems

| System | Role in Hermes | Access |
|---|---|---|
| Encoprint (v4.10.1) | **The count source.** Encode/verify per label, also checks the vision OCR | On-screen counters; per-label CSV on request |
| Vision app (V3.51) | Not a source. Its Record button is a searchable history of NG images, not live counts | — |
| G6 print program | Not a source. Holds the artwork records; never looked at | — |
| Production Planner | Per-roll production record | Excel for the web (SharePoint) → Graph |
| QC File – Horiz UPC | Per-roll QC record | Excel for the web (SharePoint) → Graph |
| Sage 300 + RFID Label Production add-on | Inventory + reporting | **GUI only**, over RDP to `192.168.26.6` |

### Notes per system

**Encoprint** — one work order per UPC (not per roll), organized by MO_NO and SKU. MO_NO is
`SEQ<order>-<timestamp>`, e.g. `SEQ200112021-20261105091307` for order
`200112021`, so the order number can be read from it. Shows Total /
Passed / Total Failures per MO and per SKU, a real-time Total / Pass /
Write Fail / Visual Fail block, and a per-EPC PASS result table. Its log
shows `Visual data=<UPC> … Successful alignment` for every label, so it
already cross-checks the vision camera.

- The work order panel header reads `SKU_<UPC> Target quantity: <n>
  Completed Quantity: <n>`. That running count is what Hermes needs.
- **The CSV is not written automatically.** It is generated on request
  with a button in the Work Order panel (log line `[<MO_NO>]The data file
  has been loaded!`) and lands in `C:\US001\Finished_report\<MO_NO>.csv`.
- The CSV has one row per label for the whole order (299,999 rows, 70 MB
  for a 300,000 order), most `UNPROCESSED` until printed. Processed rows
  have `STATUS` (PASS / FAIL_VISION / …), `EQ_TIME` (ms timestamp),
  `OCRCODE`, `EPC`, `TID`. A retried label keeps only its last result, so
  counts never double. See `fixtures/200112040-750045001271/`.
- **How Hermes reads it:** at each lot scan and roll-complete scan, read
  Completed Quantity from the Encoprint window (UI Automation on the
  production PC, no RDP involved). Request the CSV only when a halt needs
  investigating, or at the end of an order. Fallback if the window text
  can't be read: Hermes presses the export button itself after the
  machine stops.

**Production Planner / QC file** — both are open in Excel for the web on
the production PC, not desktop Excel, so writes through Graph should
appear without a reload. Layouts (from screenshots; confirm from video):
- Planner: one block per UPC, one column per roll, rows Quantity, Date,
  Lot 1, Qty 1, Lot 2, Qty 2, Initials (e.g. `KB P-HS`). Date as `11/6`.
- QC file: one block per UPC, one row per roll, columns qty, Date, Ship #,
  Notes, Tech (e.g. `KB QC-HS`). Date as `11/6/2026`.
- Example of all three filled in: `fixtures/200112040-750045001271/entered.txt`.

**Sage 300** — database and API access are blocked and will not be
granted. Entry happens in the **Production** window of the
"Production Applications (RFID Label Production)" add-on (dealer: MSI
Technologies), inside the Sage 300 desktop client on `192.168.26.6`,
reached by Remote Desktop from the production PC. Entries are
**practically irreversible** — treat every write as permanent.

- Top grid, one row per order item: Customer ID, Order Date, Item,
  Product Code (= raw material item, e.g. `TGTG030000223`), Max Roll
  Quantity (9,000), Qty Order Allowed, Roll Count, Completed Label %.
- **Roll Count is the planned number of rolls** (`ceil(order qty / 9000)`:
  100,000 → 12, 50,000 → 6). It does not change when a roll is posted.
- **Completed Label %** reads like `36,000 of 50,000 - 72.00%`, where
  36,000 is the sum of Qty Already Printed over all roll rows (scrap
  excluded).
- Bottom grid, **one row per lot used in a roll** (a roll spanning two lots
  has two rows): Roll Number, Operator, Printer, Lot No, Qty to Print,
  Rotate 180, Qty Already Printed, Qty Scrapped, Test Print, Print, Print
  Result, Order Number, Item, Raw Material ItemNo, RePrint, LoftwareCode.
  Roll rows for the order already exist; entry fills Lot No, Qty Already
  Printed and Qty Scrapped. Finished rows have Qty to Print 0. Operators
  sometimes pre-load the next roll (Lot No + Qty to Print) before it runs;
  **Hermes only ever enters rolls that are complete.**
  When a quantity is confirmed (Enter), Sage fills in the **lowest** of:
  the number typed, the number still required for the roll, and the
  number remaining on the lot. Hermes types its own computed quantity, and
  read-back must show exactly that number; if Sage lowered it, Sage and
  Hermes disagree about the lot or the roll, so Hermes halts. How a pre-loaded
  row turns into a finished one is in the pending video.
- The grid is a .NET grid ("Contains:" filter row), so UI Automation run
  *inside* the session should be able to read cells as text.

## 4. Data model

### Lot
Scanned from the raw material roll label (Tageos) when the roll is loaded.
The label is inside the core and is peeled off at load.
- **Lot number** — the only barcode on the label, e.g. `8000456411`
- **Product code** — printed text, e.g. `3000000223`
- **Qty good** — printed text, e.g. `9000`. **Not in the barcode, and not
  needed:** a full lot of a given raw material always holds the same
  number (9,000 for `TGTG030000223`, 20,000 for `TGTG045000048`), so it
  comes from the product table.
- **Partly used lots:** an order can finish with labels left on a lot,
  and that lot is used again on a later order. Its remaining quantity
  comes from Sage, which shows how many are left when the lot is used
  again. Hermes also keeps its own lot ledger (every quantity it posts
  against a lot); once a lot has been through Hermes, the ledger and Sage
  must agree, or Hermes halts.

Starting quantity for the remainder math = full lot size for a lot never
used before, otherwise Sage's remaining quantity.

### Product code → Sage raw material
A lookup table in config: product code → Sage raw material item → full
lot size, e.g. `3000000223` → `TGTG030000223` → 9,000. The product code
for `TGTG045000048` (20,000) is still needed. Each Sage
order already shows its Raw Material ItemNo, so Hermes checks that the
scanned lot matches the order. An unknown product code or a mismatch
halts. No derived mapping rule.

### Roll (output)
- Order number, SKU / UPC, roll number
- Quantity
- Lot 1 + Qty 1, Lot 2 + Qty 2 (a roll may span two lots)
- Date, operator initials
- Scrap: `0` for now

### Lot→roll attribution
Lot identity comes from the scan — no estimation. **Lot arithmetic is the
source of truth** for the split: the lot carried X, earlier rolls used Y,
the remainder X − Y goes to this roll, the next lot covers the rest.

Labels lost at a lot splice (a couple per splice) are **ignored**: the
roll is recorded as if none were lost.

Encoprint counts are a **check, not a source.** Encoprint keeps one work
order per UPC, not per roll, so Hermes takes the roll's count as the
difference in that work order's count between the previous roll-complete
scan and this one. If that disagrees with the arithmetic by more than the
tolerance, something happened — a missed changeover scan, a reprint, a
jam. Halt and show it.

Example of what this catches: in order 200112021 a lot was entered as
4,043 + 4,961 = 9,004 against a 9,000 label.

## 5. Per-roll sequence

Ordered so that cheap-to-fix surfaces are written before the irreversible
one.

1. **Lot scan** at load — lot number, product code, qty good, timestamp
2. **Roll complete** — operator marks it (scan a `ROLL COMPLETE` card)
3. **Read** the Encoprint data for the window
4. **Reconcile** — Encoprint count agrees with lot arithmetic, within the
   tolerance in §9
5. **Write Production Planner** (reversible)
6. **Write QC sheet** (reversible)
7. **Stage Sage entry** — show exact values, human approves (PoC phase)
8. **Type into Sage** — the helper on `192.168.26.6` fills the roll's rows
9. **Read back** — the roll's rows hold the right lots and quantities, and
   Completed Label % went up by exactly the roll quantity. If not, halt.

QC sheet: the machine already reads, encodes, locks and re-reads, so the
QC file is a formality. Fill it out per completed roll.

## 6. Audit log

Append-only local log. Hermes records every value it computed and every
action it intends to take **before taking it**, plus the result after.
This is how the system is audited without watching it.

**Crash rule:** an intended Sage write with no recorded result means the
write may or may not have happened. Hermes halts on start-up and a human
checks Sage before anything else is posted. Never retry a Sage write
automatically.

## 7. Deployment

**Production PC** (Windows 11) runs Hermes: scanner input, Encoprint
reading, reconciliation, Excel writes, pop-ups, audit log.

**Sage host** (`192.168.26.6`) runs a small helper inside the RDP session
that Hermes uses for Sage: it takes staged entries, types them into the
Production window, reads the grid back, and returns the result. Programs
can be run there. Other people use that machine occasionally, never at
the same times as production. The RDP session must stay connected (not
minimized-and-suppressed, not logged off) while the helper works.

**Language:** Python is not installed on the production PC yet (checked
2026-11-06). Plan is Python on both machines; the python.org installer can
install for the current user without admin rights. AutoHotkey is the
fallback for the Sage helper.

## 8. Rollout

**Phase 0 — investigation (no code)** — see `phase0/README.md`

**Phase 1 — read-only shadow.** Hermes computes every number and shows
what it *would* enter. Runs alongside manual entry for a week. Compare.
Risks nothing. This is the proof of concept.

**Phase 2 — Excel writes.** Reversible. Sage still manual.

**Phase 3 — staged Sage.** Hermes types after human approval. Read-back
verification mandatory.

**Phase 4 — unattended**, with halt-on-discrepancy.

**Later:** teach Hermes good vs. bad labels; take over watching the
machine. Hand finished rolls to easyQC's Cores tab for core labels.

## 9. Open questions

- Can UI Automation read Completed Quantity from the Encoprint window?
- Count tolerance: how far Encoprint may differ from the arithmetic
  before Hermes halts. Splice losses make Encoprint run a few labels
  short; one snapshot also showed 5 more passes than expected
- Graph access: an app registration in Microsoft Entra usually needs IT
  admin consent. Fallbacks: Office Scripts via Power Automate, or driving
  Excel for the web
- Can Python be installed on the production PC and the Sage host?
- How Hermes reads a partly used lot's remaining quantity before typing
  (Sage clamps to it on Enter; see video)
- Exact steps and pop-ups when entering a roll in the Sage Production
  window (video pending)
