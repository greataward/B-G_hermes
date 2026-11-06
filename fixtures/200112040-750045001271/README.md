# Order 200112040, UPC 750045001271 (2026-11-06)

`encoprint-processed-only.csv` is the Encoprint export for work order
`SEQ200112040-20261105043453.1`, with the 280,437 `UNPROCESSED` rows
removed (the full file is 70 MB and lists every label in the order).
Nothing else is changed.

What it shows:
- One row per label (`SN` 0–19561), so a reprocessed label overwrites its
  row instead of adding one. `EQ_TIME` is when the label was processed.
- Rolls on this order are 15,000. Labels 0–14999 = 15,000 (14,999 PASS,
  1 FAIL_VISION at SN 13779), matching Roll 1 = 15,000 on the QC sheet.
- The line stopped for 1.9 min right after SN 15000 (15:54:12 → 15:56:04),
  i.e. the roll change.
- Other stops: after SN 13865 (2.6 min) and SN 15342 (6.2 min). Cause
  unknown.
- About 1,008 labels per minute while running.
- Encoprint's screen showed Total Failures 5 while the file has 1 failed
  row: failed labels that were retried and passed keep only their last
  result.
- Roll 2 so far: 4,562 processed rows, while the real-time counter showed
  4,561.

Still needed: what was entered by hand for roll 1 (lots and quantities) in
the Production Planner and Sage.
