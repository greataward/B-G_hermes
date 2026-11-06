# Phase 0 — investigation

No Hermes code until these are answered. Findings go here with the date.

## Checklist

- [x] **G6** — not a data source. It holds the artwork records and nobody
      looks at it. (2026-11-06)
- [x] **Vision app** — not a data source. Its Record button opens a
      searchable history of NG images by date and time, not live counts.
      Encoprint already checks the vision OCR on every label. (2026-11-06)
- [x] **Raw material roll label** — Tageos label inside the core. Only the
      lot number is a barcode; product code and Qty good are printed text.
      (2026-11-06)
- [x] **Product code → Sage item** — no rule; use a lookup table.
      `3000000223` → `TGTG030000223`. (2026-11-06)
- [x] **Sage screen** — Production window of the RFID Label Production
      add-on in the Sage 300 desktop client on `192.168.26.6`. One row per
      lot per roll. Roll Count is planned rolls; read back Completed
      Label % instead. (2026-11-06)
- [x] **Can programs run on the Sage host?** Yes. Shared occasionally,
      never during production. (2026-11-06)
- [x] **Python on the production PC** — not installed. (2026-11-06)
- [ ] **Video: entering one roll** in the Production Planner, QC file and
      Sage, start to finish (recorded, to be added)
- [x] **Encoprint export** — per-label CSV, generated only on request
      (button in the Work Order panel) into `C:\US001\Finished_report`.
      Sample in `fixtures/200112040-750045001271/`. Roll 1 = exactly
      15,000 labels, matching the QC sheet. (2026-11-06)
- [x] **Hand-entered numbers** for order 200112040 →
      `fixtures/200112040-750045001271/entered.txt`. Lots on this product
      hold 20,000, not 9,000. (2026-11-06)
- [ ] **Read Completed Quantity** from the Encoprint window with UI
      Automation (needs Python or AutoHotkey on the production PC)
- [x] **Encoprint live data:** nothing in `C:\US001` is updated while
      printing; reading the window is the plan. (2026-11-06)
- [ ] **Graph:** write one cell to the open log sheet. Does it land
      without a reload and without a file-in-use prompt? (Needs an Entra
      app registration first — ask IT.)
- [ ] **Sage host:** `winver` and Settings → System → About inside the
      remote session; can Python be installed there?
- [ ] **Production PC:** Settings → System → About; can Python be
      installed for the current user?
- [x] **Splice waste:** a couple of labels are lost per splice; ignored,
      rolls are recorded as if none were lost. (2026-11-06)
- [x] **Encoprint granularity:** one work order per UPC, not per roll.
      (2026-11-06)
- [ ] **Count tolerance:** how far Encoprint may differ from the lot
      arithmetic before Hermes halts

## Evidence (2026-11-06, order 200112021, SKU 618842550500)

- Sage: rolls 1–4 entered as two rows each (4957/4043, 4957/4043,
  4957/4043, 4961/4039). Completed `36,000 of 50,000 - 72.00%`. Roll
  Count 6 (= ceil(50,000 / 9,000)).
- One lot appears as 4,043 (roll 3) + 4,961 (roll 4) = 9,004 against a
  9,000 label — an entry error the lot arithmetic would catch.
- Encoprint, SKU 618842550500: Total 50,126, Passed 38,111, failures 6.
  Real-time block for the roll in progress: 2,106 total, 2,106 pass.
- Roll 1 has 7 in Qty Scrapped, although scrap is normally entered as 0.
