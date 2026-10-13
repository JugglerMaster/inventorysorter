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

## Local compile verification (C# compiler IS now available)
- **.NET SDK 8.0.425 is installed at `~/.dotnet`** (no sudo needed; it was already present). Its `csc` is **Roslyn — the exact compiler engine the SE Programmable Block uses**, so error codes match in-game.
- Install/refresh if ever missing: `curl -sSL https://dot.net/v1/dotnet-install.sh | bash -s -- --channel 8.0 --install-dir /home/dadisc01/.dotnet`.
- **Space Engineers API is NOT available on Linux** (SE has no Linux client; the PB compiler lives inside the game). So we cannot compile against the real `VRage*`/`Sandbox*` assemblies here without a Windows dev machine + game/MDK reference assemblies. This means the harness checks **structure/scope/braces only**, not SE-API correctness.
- Harness: `/tmp/opencode/verify/` (`verify.csproj` + `verify.sh`). It wraps `GSIM.txt` as `class Program { … }` with **no base class**, so every SE API symbol becomes an expected "not found" error (`CS0246`) that we filter out; real structural errors (CS0136/CS0139/brace/syntax) still surface.
  - Run: `bash /tmp/opencode/verify/verify.sh`. It prints a "STRUCTURAL ERROR CHECK" section (targets CS0136/CS0139/CS1513/CS1002/CS1022/CS1519/CS1520/CS1525 …) and a distinct-error-code tally.
- **Result after bug #2 fix: clean.** No CS0136/CS0139/brace/syntax errors. Only `CS0246` (508×, unresolved SE API types — expected stub noise) + one `CS8981` naming warning. Negative self-tests confirm the harness catches CS0136/CS0139 when injected and reports clean when fixed.
- Final full-fidelity confirmation still requires re-pasting `GSIM.txt` into the in-game PB (the only place the real SE API is resolved).

## Scan blocks de-duplicated + stricter learning (DONE)
The two near-identical scan blocks (the `SCN` scan-mode loop at old ~237, and the per-production-block auto-scan in the sort loop at old ~283) were collapsed into a single helper `LearnBlueprintFrom(IMyProductionBlock)` (defined near `S()`). It contains all the original guards (skip null / disassembler assembler / not working / empty queue / non-uniform queue) and the original learn+log+`SCN`-stop behavior. Net: ~300 fewer chars (file now 98,920).

### Stricter "real product" check
`LearnBlueprintFrom` only stamps a blueprint onto an output item when the production block's **output inventory holds a single distinct item type** — a reliable, PB-safe signal that the item is genuinely produced by (not merely sitting in) that block. Any item that differs from the first is skipped, so multi-type outputs are not auto-learned (conservative; the active-learning "is Learning" mode still covers deliberate single-item teaching). This prevents false blueprint associations for leftovers / sorted-in items **without** using the (non-whitelisted) `MyDefinitionManager` API.
- **Avoided API:** an earlier attempt used `MyDefinitionManager.Static.GetBlueprintDefinition(ß[0].BlueprintId).Outputs` to verify outputs authoritatively, but `MyDefinitionManager` / `MyBlueprintDefinition` are **NOT** in the SE Programmable Block whitelist, so it failed to compile in-game (`CS0246`/`could not be found`). The single-distinct-type heuristic is the PB-safe substitute.

### Learning LCD (`LN`) isolation
- Removed the periodic `LN.WriteText` from the generic LCD-update section (was `if(LN!=null){int Ǫ=DBG2.Count;…;LN.WriteText(…);}` at old ~505). Now **only learning code** writes the learn LCD: the new helper `WriteLearnLog()` (called from `LearnBlueprintFrom` and from the active-learning "learn" branch at ~244). `LC` (craftingStatus) and all other LCD code are untouched. `WriteLearnLog()` also owns the `DBG2` 25-line trim, so the generic update no longer manages that buffer.
