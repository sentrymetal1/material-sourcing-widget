# Material Sourcing Widget
### Material Compass — Sentry Metal Services

A hosted Zoho Creator widget that replaces the native `Customer_New_RFQ_Acceptance_Detail_Report` action button with a no-refresh sourcing interface. All Yes/No selections are held client-side and written to Zoho in a single batch on confirm — eliminating the per-click page reload.

---

## What it does

1. Loads all `Customer_RFQ_Acceptance_Detail` records for the given quote
2. Groups them by line item, one row per supplier per line
3. Highlights the lowest price across suppliers with a ★ star
4. MFG clicks Yes/No per supplier row — no refresh on each click
5. On "Confirm sourcing & save":
   - Sets `PREFER = YES` on selected rows, `PREFER = NO` on all others for that line
   - Updates `Quote_Form.Status` → `Sourcing Complete`
   - Updates `Supplier_Verify_Form.Allocation_Amount` and `Allocation_Percentage` per supplier
6. Shows a success summary with sourced items, weight, dollar total by supplier
7. Offers "Review & modify sourcing" or "Proceed to purchase order" buttons

---

## Files

```
material-sourcing-widget/
├── index.html    — the complete self-contained widget
└── README.md     — this file
```

---

## GitHub Pages

- Repo: `https://github.com/sentrymetal1/material-sourcing-widget`
- Live URL: `https://sentrymetal1.github.io/material-sourcing-widget/`
- To update: upload new `index.html` via Add file → Upload files → commit to main

> **Cache fix:** Rename widget in Zoho Settings → Widgets to `material_sourcing_widgetX` and back to `material_sourcing_widget` to force cache clear.

---

## Zoho Widget registration

Already registered as:
- **Widget name:** `material_sourcing_widget`
- **Hosting:** External
- **Index File:** `https://sentrymetal1.github.io/material-sourcing-widget/index.html`

---

## Page Builder setup — IMPORTANT

### Widget must be placed as a direct Widget element, NOT inside an HTML snippet

The portal (`customer.materialcompassportal.com`) blocks external iframes loaded via zc-component inside HTML snippets. The widget must be dragged onto the page as a native Widget element.

### Steps

1. Open `New_Material_Pending_Contract_Page` in Page Builder
2. Left sidebar → **Widgets** → **Custom** tab
3. Find `material_sourcing_widget`
4. Drag it directly onto the page where the sourcing table should appear
5. No params configuration needed — widget reads `quote_id` from the page URL automatically

### HTML Snippet 3

Can be removed or left in place. If left it shows "Loading sourcing widget..." but does not conflict with the direct widget element.

---

## How quote_id is obtained

The widget tries three methods in order:

1. **Zoho SDK params** — `ZOHO.CREATOR.UTIL.getQueryParams()` — works via zc-component embed
2. **Parent window URL** — `window.parent.location.href` — works for direct widget element placement
3. **Self window URL** — `window.location.href` — fallback

The status bar (bottom right corner of widget) shows which method succeeded.

---

## Field reference

### Customer_RFQ_Acceptance_Detail (report: Customer_New_RFQ_Acceptance_Detail_Report)
| Field link name | Purpose |
|---|---|
| `Quote_LU` | Filter — quote ID (number field, always populated) |
| `CustRFQAccept_Line_Item` | Line item number for grouping |
| `CAD_Supplier_LU` | Supplier lookup (object — .ID used internally) |
| `CAD_Supplier_Name` | Supplier display name |
| `CAD_Item_Info` | Item description |
| `CAD_Quantity` | Quantity |
| `CAD_Unit_Weight` | Unit weight |
| `CAD_Calc_Weight` | Total weight |
| `CAD_Price_Per_Lb` | Price per lb |
| `CAD_Unit_Price` | Unit price |
| `CAD_Total_Price` | Total price |
| `CAD_Item_Verification_Status` | Item status (Quote As Is, No Quote, etc.) |
| `PREFER` | YES/NO sourcing flag — written by widget |

### Quote_Form (report: All_Quotes)
| Field link name | Purpose |
|---|---|
| `Status` | Updated to `Sourcing Complete` on confirm |

### Supplier_Verify_Form (report: All_Supplier_Verify_Form_Report)
| Field link name | Purpose |
|---|---|
| `SV_Quote_Form` | Filter — matches quote ID |
| `SV_Supplier_Entry_Form` | Filter — matches supplier ID |
| `Total_Amount_Currency` | Used to calculate allocation percentage |
| `Allocation_Amount` | Written — dollar amount sourced from supplier |
| `Allocation_Percentage` | Written — percentage of supplier quote sourced |

---

## Report requirements

Fields must be added to reports (visible or hidden):

### Customer_New_RFQ_Acceptance_Detail_Report
All fields listed above.

### All_Quotes
- `ID`, `Status`

### All_Supplier_Verify_Form_Report
- `ID`, `SV_Quote_Form`, `SV_Supplier_Entry_Form`, `Total_Amount_Currency`, `Allocation_Amount`, `Allocation_Percentage`

---

## Status flow

| Stage | Quote_Form.Status |
|---|---|
| RFQ sent to suppliers | Open - Quoted |
| MFG closes out quote process | Sourcing In Progress |
| MFG confirms sourcing | **Sourcing Complete** ← widget sets this |
| MFG creates PO | Purchase Order Pending |

---

## Required Deluge change — Step 1 workflow

In `Add_Records_To_Acceptance_Detail` on `MFG_Pending_Purchase_Order`, change:

```javascript
StatusChange.Status = "Purchase Order Pending";
```

To:

```javascript
StatusChange.Status = "Sourcing In Progress";
```

---

## Proceed to purchase order button

Once PO page name is confirmed, update `proceedToPO()` in `index.html`:

```javascript
function proceedToPO(){
  window.parent.location.href = 'https://customer.materialcompassportal.com/#Page:Your_PO_Page_Name?quote_id=' + quoteId;
}
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| "Widget SDK failed to initialize" | SDK not available | Widget falls back to URL reading automatically |
| "No quote ID found" | URL missing quote_id | Ensure page opened from sourcing dashboard with correct params |
| Widget loads but no records | Quote_LU filter empty | Check Customer_RFQ_Acceptance_Detail records exist for this quote |
| PREFER not updating after confirm | Report name mismatch | Verify `Customer_New_RFQ_Acceptance_Detail_Report` is exact report link name |
| Changes not appearing after GitHub push | Zoho cache | Rename widget to `material_sourcing_widgetX` and back |

---

## Related widget

MaterialIntel widget at `sentrymetal1.github.io/materialintel-widget/MaterialIntel.html` follows the same SDK pattern.
