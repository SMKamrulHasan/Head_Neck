# Roll-up and segmentation rules: Rybrevant IV + FASPRO, past 52 weeks

Notebook: `HN_RYB_52W_Segmentation`

## Scope
- **Products:** Rybrevant IV + Rybrevant FASPRO. Lazcluze is excluded.
- **Window:** past 52 weeks, **2025-10-24 → 2026-10-22**.
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
| **DDD** | L1 → L2/L3 | `jnj_id` (buyer) | `prod_nm` contains RYBREVANT (IV + FASPRO) | `equiv_uns` | `caln_dt` | Lung; DDD in the prior 13 weeks for the weekly rule |
| **Xponent** | L0 → L2/L3 | `jnj_id` (HCP) | `prod_nm` contains RYBREVANT; TRx > 0 | `equ_trx` | `caln_dt` | Lung |
| **MX claims** | L0 → L2/L3 | `rendering_npi` → dmn_cust | NDCs 57894050100, 57894050101 (IV); 57894051001, 57894051401, 57894051501, 57894052201 (FASPRO) | infusions (patient + day + NDC) | `service_date` | Lung |
| **RX claims** | L0 → L2/L3 | `prescriber_npi` → dmn_cust | Same 6 NDCs; only patients **not** in MX | fills (one per `claim_id`) | `fill_date` | Lung |
| **McKesson EMR** | L0 → L2/L3 | `physician_npi` → dmn_cust | `alert_name` contains RYB (IV, FASPRO, Rybrevant + TKI) | alerts | `diagnosis_date` | Lung; **H&N** if the diagnosis contains HNSCC / HEAD / NECK |
| **Chargemaster** | L0 (HCP) or organization → L2/L3 | `NPI` → dmn_cust | **Monthly** rows overlapping the 52 weeks, summed per NPI (no product column) | `# EncountersNSCLC`, `# EncountersHNSCC` | month in `Time Frame Descriptive` | Lung if NSCLC > 0; **H&N** if HNSCC > 0 |
| **LE.AI H&N** | L0 → L2/L3 | `J&J ID` (from NPI via dmn_cust if blank or "-") | None (2-year cohort) | H&N HCPs, H&N treating patients | — | **H&N** |

## Segment rules
**Account segment (past 52 weeks)**
- **Lung present:** any Rybrevant IV or FASPRO activity placed on the account in the window, from any source.
- **H&N present:** LE.AI HCPs, **or** Chargemaster HNSCC > 0, **or** a McKesson H&N diagnosis.
- **Segment:** both → **H&N + Lung**; H&N only → **H&N only**; Lung only → **Lung only**.

**Weekly DDD split**
- **Products split:** `SPLIT_PRODUCTS` (default **FASPRO**).
- **Weeks:** those starting inside the window.
- **Lung present that week:** Lung from the **non-DDD** sources (52 weeks), **or** any Rybrevant (IV or FASPRO) DDD purchase in the **13 weeks before** that week.
- **H&N present:** as above.
- **Segments:** H&N + Lung / H&N only / Lung only / **Unclassified** (no signal) / **Unmapped account** (DDD buyer not in OPAL).

## Known limitations
- **Unmapped DDD buyers:** buyer IDs not in `OPAL_Affiliation_Levels` are dropped. That table holds only outlets linked to ONC HCPs. In the FASPRO YTD run this was ~42% of FASPRO DDD units.
- **Chargemaster:** has no product column, so it can't be limited to Rybrevant.
- **McKesson:** data starts May 2026, so it doesn't cover the full 52 weeks.
- **Mixed levels:** the HCP side and the buyer side of one health system can land on different account IDs (L2 vs L3).
