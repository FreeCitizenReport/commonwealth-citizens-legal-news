# Commonwealth Citizens Legal News — SORT RULE (canonical, do not change without Van's sign-off)

The main landing list and all filtered views MUST sort in this exact order:

1. **In-custody first**, released last. A record is released iff `releaseDate` is a non-empty string.
2. **With mugshot first**, without-mugshot last.
3. **Most recent `bookDate` first** (MM/DD/YYYY parsed via `parseDate`). This is the PRIMARY ordering key.
4. Tiebreaker when bookDates are identical: **more recent `mugshotDate` first** (ISO 8601 string compare).
5. Final tiebreaker for stability: **name alphabetically (A–Z)**.

## DO NOT use `firstSeenAt` as a primary or secondary sort key.

`firstSeenAt` records the moment the Commonwealth aggregator first ingested a record.
When a new source is onboarded (e.g. Peninsula was added 2026-04-18), every record it
ingests — regardless of how old the actual booking is — gets `firstSeenAt = today`. This
pushes stale, old-mugshot records above fresh same-day bookings.

The canonical symptom of this regression: **FARMER, CLIFTON (bookDate 04/13/2026) appearing
above PRESSLEY, ALEX (bookDate 04/18/2026)** on the landing page on 2026-04-18, because
FARMER was first-seen 13 minutes after PRESSLEY when the Peninsula source came online.

## Canonical comparator

```js
RECORDS.sort(function(a, b) {
  var ar = a.releaseDate ? 1 : 0, br = b.releaseDate ? 1 : 0;
  if (ar !== br) return ar - br;
  var am = a.mugshot ? 0 : 1, bm = b.mugshot ? 0 : 1;
  if (am !== bm) return am - bm;
  var bd = parseDate(b.bookDate) - parseDate(a.bookDate);
  if (bd) return bd;
  var amd = a.mugshotDate || "", bmd = b.mugshotDate || "";
  if (amd !== bmd) return bmd < amd ? -1 : 1;
  var an = (a.name || "").toUpperCase(), bn = (b.name || "").toUpperCase();
  return an < bn ? -1 : an > bn ? 1 : 0;
});
```

## History
- `1229ac1d` initial scaffold (bookDate sort).
- `8345feae` regressed to `firstSeenAt` primary with `bookDate` tiebreaker — this is the bug.
- Fix (this commit): restored `bookDate` primary; added `mugshotDate` + name tiebreakers; added this rule file.
