# GOAT Sorter / GSIM — Agent Notes

Space Engineers Programmable Block (PB) inventory manager script (version 1.3.0, by Khodrin).

## Hard constraint: ~100,000 CHARACTER limit (not bytes)
The in-game PB code editor enforces a limit on **Unicode characters**, not file bytes.
- `wc -c` reports ~106,769 bytes for `GSIM.txt`, but that overcounts because the script uses many multi-byte UTF-8 variable names (Ç, ª, Ť, …). The real character count is ~100k — "within a few characters" of the limit, per the user. So the file is NOT over the limit; just keep edits character-neutral so it doesn't cross it.
- Prefer compact/minified form over adding comments or whitespace.

## File roles
- `GSIM.original.txt` — clean upstream script (no custom "scan" feature).
- `GSIM.txt` — user's modified version that adds a custom **scan** feature (`SCN` flag, `S()` command, scan blocks at ~237 and ~283, `DBG2`, `LN`/`LC` learn surfaces, etc.). This is the file being edited.

## Scan feature bug #1 — overwriting known items (FIXED)
The scan stamped a production block's queued blueprint onto EVERY item in that block's output inventory, overwriting items that already had a correct blueprint (e.g. `Construction`/`Computer` wrongly set to `Electromagnet`).
- Fix: only assign a blueprint when `Ç[ª.Type].Ŧ` is null, i.e. change `if(Ç[ª.Type].Ŧ!=ß[0].BlueprintId)` → `if(!Ç[ª.Type].Ŧ.HasValue)` in both scan blocks. Applied at lines 237 and 283.

## Scan feature bug #2 — compile errors CS0139 / CS0136 (FIXED)
Reported errors: CS0139 "No enclosing loop out of which to break or continue" (~lines 293, 335) and CS0136 "A local or parameter named 'r' cannot be declared" (~lines 379, 438, 439, 450, 455).
- Root cause: the two inserted **scan blocks were each brace-imbalanced** (they only balanced out globally by canceling each other). This mis-nested the code, so a `break`/`continue` fell outside its loop (CS0139) and method `g()` got parsed as nested inside `e()`, making the field `r` (only ever `List<IMyShipConnector>r`, never a local) look like a redeclared local (CS0136).
- Note: `r` is ONLY the field `List<IMyShipConnector>r` (lines 164, 191, 193) plus a `'\r'` char literal. There is no local `r` — so the "'r' redeclared" error is a symptom of the brace mis-nesting, not a literal duplicate declaration.
- Fix: made each scan block internally balanced.
  - Block at 237: trailing `}}}}}}` (6) → `}}}}} (5)`.
  - Block at 283: trailing `}}}}}}` (6) → `}}}}}}}` (7)`.
- After fix, both blocks and the whole file report brace balance 0.

## Verification notes
- No C# compiler is available in this environment (no dotnet/csc/mcs/mono), so fixes were verified structurally (brace-balance script that respects strings/comments) rather than by compiling. Compiler-reported line/col numbers did NOT match `GSIM.txt`'s physical lines (the compiled/pasted version is reformatted differently), so diagnosis was done by code inspection + brace analysis, not by line number.
