# Ibuprofen suspension 200 mg/5 mL — f₂ dissolution analysis — 2026-08-10

**Repo:** none (local data analysis, not a Revelator project)
**Source data:** `C:\Users\vasil\Downloads\Ибупрофен сус..xlsx`, sheet `200 mg` (read-only, never modified)
**Deliverables:** `C:\Users\vasil\Downloads\Ibuprofen_analysis\`

## What we did

- Executed the prompt in `C:\Users\vasil\Downloads\prompt.md`: full formulation-development analysis of an ibuprofen oral suspension 200 mg/5 mL vs Nurofen Junior reference, targeting dissolution similarity f₂ ≥ 50.
- Programmatically reconstructed the workbook (openpyxl): 2 reference batches (AGB791 col B, AGA143 col C), 45 experimental formulations (D–AV), compositions g/100 mL (water = 116 − Σ mass balance), 4 dissolution media, similarity factors hardcoded in rows 42–43.
- **Decoded the f₂ convention**: rows 42/43 are both f₂ (not f₁/f₂) — vs batch B and vs batch C respectively, computed on the first 3 time points (10/15/30 min) only. Early columns reproduce exactly; verified against 9 computation variants.
- **Found material f₂ discrepancies**: recorded values for E3П54, Е3П51, E7П1, E7П3, Е5П7 are 1.3–12 points LOWER than their own profiles compute to. Biggest: E3П54 S-250 recorded 47.8/45.2, recalculated **49.9 vs AGA143 / 57.3 vs AGB791** — i.e. possibly already passing vs one reference batch.
- Factor analysis: API grade dominates 30-min completeness (S-250 95.0 → std 69.5 at matched composition); xanthan dominates 10-min burst (slope ≈ −8.9%/0.1%, r = −0.86, n = 29); P80 ≥ 0.4 causes burst; glycerol ≥ 15% chokes fine-API release; reference viscosity (3255–3609) exceeds all experiments.
- Mechanism: reference = sigmoid (thick vehicle suppresses 0–15 min + fast API completes by 30 min); every experiment breaks one half of that pair.
- Recommended **Formulation A** = E3П54 base with single change xanthan 0.50 → 0.60 (S-250, P80 0.10, gly 12); alternatives B (E3П54 replicate n=12) and C (Е4П4 rework: xan 0.65, P80 0.15); 5-run xanthan-ladder DoE with per-run confirm/reject criteria.
- EU regulatory review with verified sources: EMA BE guideline App. I f₂ conditions, ICH M9 (no BCS biowaiver — ibuprofen Class II), EMA ibuprofen PSG 200–800 mg, physical-stability/CQA trade-off flagged.
- Built deliverables: 14-section Word report (9 tables, 4 embedded charts), 4 dataviz-validated PNG overlay charts, reconstructed dataset (xlsx 4 sheets + 2 UTF-8-BOM CSVs, 47 formulations / 380 tidy dissolution points).

## Key decisions

- Used 3-point f₂ (EMA-compliant here: refs > 85% at 30 min, max one point > 85% allowed) as the primary recomputation; reported all variants (vs B/C/mean, 3/4/5 pts) in the dataset.
- Treated the AK-profile-vs-recorded-f₂ conflict as unresolvable from the workbook → made "recompute/replicate E3П54 first" the top recommendation instead of picking a side.
- Charts follow the dataviz skill: palette validated with `validate_palette.js` (chart 2 initially FAILED normal-vision floor with magenta-beside-orange → re-slotted to default order); legends moved to empty lower-right after visual render check.
- Advised against the Sheet4 embedded plan (P80 0.5–0.6) — contradicted by the burst evidence.

## Files changed

| File | Change |
|---|---|
| Downloads\Ibuprofen_analysis\Ibuprofen_200mg_Formulation_Analysis.docx | New — full 14-section report, charts embedded |
| Downloads\Ibuprofen_analysis\Ibuprofen_200mg_Reconstructed_Dataset.xlsx | New — ReadMe / Formulations / Dissolution_long / Reference_profiles |
| Downloads\Ibuprofen_analysis\formulations.csv, dissolution_long.csv | New — tidy exports for DoE/stats software |
| Downloads\Ibuprofen_analysis\charts\01–04*.png | New — top candidates, failure modes, Formulation A prediction band, factor evidence |

Original workbook untouched. No code committed anywhere (analysis scripts live in session scratchpad only).

## Still to do / follow-up

- User to recompute rows 42/43 from raw dissolution data (or rerun E3П54) to resolve the DISCREPANCY-flagged runs before the next formulation round.
- 5-run DoE (xanthan 0.50/0.55/0.60/0.65 + P80 0.05 arm) on S-250 base; reference batch in-session each run; n=12 + CV% for the winner.
- Measure viscosity/redispersibility of retained AF–AV samples; get PSD certificates for std/S-250/S-500 API lots.
