# Roll-up and segmentation rules: Rybrevant FASPRO, YTD 2026

Notebook: `HN_FASPRO_DDD_Segmentation`

## Scope
- **Products:** Rybrevant **FASPRO only**. Rybrevant IV and Lazcluze are excluded.
- **Window:** YTD 2026, **2026-01-01 → 2026-10-22** (FASPRO launched end of Dec 2025).
- **Account level:** L2/L3 via OPAL (`L2_L3` = L2 if present, else L3).

## Shared lookups
**NPI → J&J ID (dmn_cust)**
- `npi_id` → `jnj_id`.
- If one NPI has several J&J IDs, keep the one with the **latest `efftv_dt`** (ties: smallest ID).
- Used as delivered: Active, end_dt 2099-12-31.

**J&J ID → L2/L3 account (OPAL_Affiliation_Levels, ONC).** First match wins:

| Priority | ID is… | Account |
|---|---|---|
| 1 | HCP (column `0`) | its `L2_L3` |
| 2 | Outlet / L1 (column `1`) | most frequent `L2_L3` among its HCP rows (ties: smallest ID) |
| 3 | L2 (column `2`) | itself |
| 4 | L3 (column `3`) | itself |
| — | none | unmapped (excluded) |

- **Account level:** L2 if the ID appears in OPAL column `2`, else L3.
- Each ID maps to **at most one** account, so there is no splitting or double counting.

## Rules by dataset
| Dataset | Source level → roll-up | ID used | Filter | Measure | Date | Signal |
|---|---|---|---|---|---|---|
| **DDD** | L1 → L2/L3 | `jnj_id` (buyer) | `prod_nm` contains FASPRO | `equiv_uns` | `caln_dt` | Lung; FASPRO DDD in the prior 13 weeks for the weekly rule |
| **Xponent** | L0 → L2/L3 | `jnj_id` (HCP) | `prod_nm` contains FASPRO; TRx > 0 | `equ_trx` | `caln_dt` | Lung |
| **MX claims** | L0 → L2/L3 | `rendering_npi` → dmn_cust | FASPRO NDCs 57894051001, 57894051401, 57894051501, 57894052201 | infusions (patient + day + NDC) | `service_date` | Lung |
| **RX claims** | L0 → L2/L3 | `prescriber_npi` → dmn_cust | Same 4 FASPRO NDCs; only patients **not** in MX | fills (one per `claim_id`) | `fill_date` | Lung |
| **McKesson EMR** | L0 → L2/L3 | `physician_npi` → dmn_cust | `alert_name` contains **FASPRO** (Ryb IV and "Rybrevant + TKI" alerts excluded) | alerts | `diagnosis_date` | Lung; **H&N** if the diagnosis contains HNSCC / HEAD / NECK |
| **Chargemaster** | L0 (HCP) or organization → L2/L3 | `NPI` → dmn_cust | `Time Frame Descriptive` = **CY-2026** (annual rows; no product column) | `# EncountersNSCLC`, `# EncountersHNSCC` | CY-2026 | Lung if NSCLC > 0; **H&N** if HNSCC > 0 |
| **LE.AI H&N** | L0 → L2/L3 | `J&J ID` (from NPI via dmn_cust if blank or "-") | None (2-year cohort) | H&N HCPs, H&N treating patients | — | **H&N** |

## Segment rules
**Account segment (YTD 2026)**
- **Lung present:** any FASPRO activity placed on the account with an actual date between 2026-01-01 and 2026-10-22, from any source (Chargemaster CY-2026 included).
- **H&N present:** LE.AI HCPs, **or** Chargemaster HNSCC > 0, **or** a McKesson H&N diagnosis.
- **Segment:** both → **H&N + Lung**; H&N only → **H&N only**; Lung only → **Lung only**.

**Weekly DDD split (FASPRO)**
- **Weeks:** those starting on or after 2026-01-01 (first week = week of 5 Jan 2026).
- **Lung present that week:** Lung from the **non-DDD** sources (YTD), **or** a FASPRO DDD purchase in the **13 weeks before** that week. The week's own purchase doesn't count.
- **H&N present:** as above.
- **Segments:** H&N + Lung / H&N only / Lung only / **Unclassified** (no signal) / **Unmapped account** (DDD buyer not in OPAL).

## Results (YTD run)
| Segment | Accounts | FASPRO DDD units (mapped) |
|---|---|---|
| H&N only | 2,373 | 0 |
| H&N + Lung | 722 | 4,708 (73%) |
| Lung only | 313 | 1,757 (27%) |

## Known limitations
- **Unmapped DDD buyers:** ~42% of FASPRO DDD units come from buyer IDs not in `OPAL_Affiliation_Levels`, which holds only outlets linked to ONC HCPs. These are excluded, which pushes accounts toward H&N only.
- **FASPRO-only lung universe:** accounts that treated lung with Rybrevant IV but haven't switched to FASPRO aren't counted as Lung. This is addressed in the IV + FASPRO 52-week version.
- **Chargemaster:** no product column, and annual only (applied to all weeks).
- **McKesson:** data starts May 2026.
- **Mixed levels:** the HCP side and the buyer side of one health system can land on different account IDs (L2 vs L3).
