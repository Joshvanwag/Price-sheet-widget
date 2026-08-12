# Zoho Creator: Install Price Sheet CRM Sync

## What this project is

A Zoho Creator app has a Price Sheet form. On submit, a widget opens that:
1. Fetches the record's `Total_Install_Hours` and existing `CRM_Install_Price_Sheet_ID`
2. Shows an SLA tier picklist (Bronze/Silver/Gold/Platinum) with live-calculated dollar amounts
3. On Edit only (i.e. `CRM_Install_Price_Sheet_ID` already had a value before this submit), shows a second prompt asking whether to update cost & sell price, or just cost
4. Calls a Zoho Deluge standalone function (`Sync_Install_Price_Sheet_And_PDF`) with the record ID, SLA selection, and update mode
5. That function syncs the record to a CRM "Install Price Sheets" module, updates the linked Deal/Quote, adds an SLA product line to the Quote if one was selected, attaches a versioned PDF, and returns success/failure to the widget

## Files

- `index.html` — the widget, hosted on GitHub Pages at https://joshvanwag.github.io/Price-sheet-widget/
- `Sync_Install_Price_Sheet_And_PDF.dg` — the Deluge function. This is NOT deployable from a file; Zoho Creator only runs Deluge that's pasted into its own function editor (Settings → Workflows/Functions). Treat this `.dg` file as the source of truth / draft copy, and copy its final contents into Creator manually after each change.

## Key business logic already confirmed

- **SLA formula**: `Total_Install_Hours * 125 (hourly rate) * SLA%`
  - Bronze = 0.08, Silver = 0.10, Gold = 0.12, Platinum = 0.14
  - `Total_Install_Hours` already includes programming hours (confirmed against the pricing calculation script) — do NOT add any other hours field to it, that would double-count.
- **SLA CRM Product IDs** (already filled into the function):
  - Bronze: 5439147000096147796
  - Silver: 5439147000006705652
  - Gold: 5439147000006666808
  - Platinum: 5439147000096145714
- **Create vs Edit detection**: done by checking whether `CRM_Install_Price_Sheet_ID` already had a value on the record *before* this submission — not by trusting a URL param from the form action.
- **Cost vs cost+price update mode**: on Edit, if the user picks "just cost," the function skips pushing `Total_MSRP`/`State_Contract_MSRP` to CRM and skips overwriting `list_price`/`List_Price` on the Install line in the linked Quote, but still updates cost fields (`Unit_Cost_1`, etc.) and internal cost fields on the Install Price Sheet CRM record. On Create, this is always full (both cost and price).

## Open TODOs (blocking full functionality)

1. **PDF private-link URL** — `privateLinkPDFURL` in the function still has a placeholder (`YOUR_PRIVATE_LINK_KEY`). Need the real published private-link URL for the PDF report from Zoho Creator, then swap it in. Until this is fixed, the PDF attach step will fail every run.
2. **Widget's `appName`/`reportName`** — in `index.html`, the `ZOHO.CREATOR.DATA.getRecordById` call uses `appName: "application-by-chris"` and `reportName: "All_Price_Sheets"`, carried over from other URLs used elsewhere in this app. These need to be confirmed against Zoho Creator's actual API names for this app before this call will reliably work.

## Reference: original pricing calculation script

The Price Sheet form has a separate calculation script (not shown in these two files) that computes hours and costs from a `Line_Item_Lookup` subform. Relevant facts already extracted from it:
- `Total_Install_Hours` = sum of every line item's hours × quantity, across ALL task types (video, audio, cabling, component, AND programming) — despite the name, it's a grand total, not install-only.
- `Programming_Hours` = only the programming-only portion (Tier 2 + Tier 3 programming hours). Do not use this in the SLA formula alongside `Total_Install_Hours` — that would double-count.
- `Sale_Price` and `Admin_Expenses` are already blended (install + programming) values, consistent with how the sync function uses them today. No changes needed there.

## Next steps for whoever/whatever picks this up

- Resolve the two TODOs above
- Test the widget end-to-end: does `ZOHO.CREATOR.init()` reliably connect when the widget is loaded cross-origin from GitHub Pages inside Creator's iframe? (Externally hosted widgets sometimes need the domain whitelisted in Creator's widget settings for the SDK handshake to succeed.)
- Confirm the exact shape of `ZOHO.CREATOR.UTIL.invokeCustomFunction`'s response (`response.output` vs `response.result` vs top-level) — currently handled defensively in the widget with a fallback chain, but should be locked down once observed once via `console.log`.
- Build out the real UI in `showPrompts()` in the widget (currently just shows "Synced successfully. CRM ID: ...") — no confirmed design for this yet.
