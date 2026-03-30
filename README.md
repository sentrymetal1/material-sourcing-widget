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
└── index.html       — the complete self-contained widget
```

---

## GitHub Pages setup

1. Create a new GitHub repo: `material-sourcing-widget`
   - Under account: `sentrymetal1`
2. Upload `index.html` to the root of the repo
3. Go to **Settings → Pages**
4. Source: **Deploy from branch** → `main` → `/ (root)`
5. Save — GitHub will publish at:
   ```
   https://sentrymetal1.github.io/material-sourcing-widget/
   ```

---

## Zoho Creator Widget registration

1. Go to **Zoho Creator → Settings → Widgets**
2. Click **New Widget**
3. Fill in:
   - **Widget name:** `material_sourcing_widget`
   - **Widget URL:** `https://sentrymetal1.github.io/material-sourcing-widget/`
   - **Widget type:** URL
4. Save

> **Cache fix (use when updating the widget):**
> If changes don't appear after pushing to GitHub, rename the widget to `material_sourcing_widgetX` and back to `material_sourcing_widget` in Settings → Widgets to force Zoho to clear its cache.

---

## Zoho Creator Page Builder setup

On `New_Material_Pending_Contract_Page` (or whichever page hosts the sourcing view):

### Replace the existing report embed

Remove the existing `Customer_New_RFQ_Acceptance_Detail_Report` zc-component and replace with a **Widget** element.

### Widget element configuration

In the Page Builder widget embed, pass these params:

```
<%
widgetParams = "quote_id=" + input.quote_id
  + "&quote_reference_number=" + input.quote_reference_number
  + "&quote_description=" + input.quote_description;
%>
<div clName='zc_component'
  widgetLinkName='material_sourcing_widget'
  Page_Name='New_Material_Pending_Contract_Page'
  params='<%=widgetParams%>'
  style='height:800px; width:100%; scrolling:no'>
  Loading...
</div>
```

> Adjust `height` as needed. The widget scrolls internally so 800px is usually sufficient.

---

## Field reference

### Customer_RFQ_Acceptance_Detail
| Field link name | Purpose |
|---|---|
| `Quote_LU` | Filter — quote ID (number field, always populated) |
| `CustRFQAccept_Line_Item` | Line item number for grouping |
| `CAD_Supplier_LU` | Supplier lookup (returns object — `.ID` used internally) |
| `CAD_Supplier_Name` | Supplier display name |
| `CAD_Item_Info` | Item description |
| `CAD_Quantity` | Quantity |
| `CAD_Unit_Weight` | Unit weight |
| `CAD_Calc_Weight` | Total weight |
| `CAD_Price_Per_Lb` | Price per lb |
| `CAD_Unit_Price` | Unit price |
| `CAD_Total_Price` | Total price |
| `CAD_Item_Verification_Status` | Item status (Quote As Is, No Quote, etc.) |
| `PREFER` | YES/NO sourcing flag — this is what the widget writes |

### Quote_Form (via All_Quotes report)
| Field link name | Purpose |
|---|---|
| `Status` | Updated to `Sourcing Complete` on confirm |

### Supplier_Verify_Form (via All_Supplier_Verify_Form_Report)
| Field link name | Purpose |
|---|---|
| `SV_Quote_Form` | Filter — matches quote ID |
| `SV_Supplier_Entry_Form` | Filter — matches supplier ID |
| `Total_Amount_Currency` | Used to calculate allocation percentage |
| `Allocation_Amount` | Written — dollar amount sourced from this supplier |
| `Allocation_Percentage` | Written — percentage of supplier quote being sourced |

---

## Report requirements

These reports must exist and have the listed fields added (visible or hidden):

### Customer_New_RFQ_Acceptance_Detail_Report
All fields listed above under Customer_RFQ_Acceptance_Detail must be present.

### All_Quotes
- `ID`
- `Status`

### All_Supplier_Verify_Form_Report
- `ID`
- `SV_Quote_Form` (shown as Quote ID)
- `SV_Supplier_Entry_Form` (shown as Supplier ID)
- `Total_Amount_Currency`
- `Allocation_Amount`
- `Allocation_Percentage`

---

## Status flow

| Stage | Quote_Form.Status |
|---|---|
| RFQ sent to suppliers | Open - Quoted |
| MFG closes out quote process | Sourcing In Progress |
| MFG confirms sourcing selections | **Sourcing Complete** ← widget sets this |
| MFG creates purchase order | Purchase Order Pending |

---

## Step 1 Deluge change required

In the `Add_Records_To_Acceptance_Detail` workflow on `MFG_Pending_Purchase_Order` form, change the Quote Form status update from:

```javascript
StatusChange.Status = "Purchase Order Pending";
```

To:

```javascript
StatusChange.Status = "Sourcing In Progress";
```

---

## Proceed to purchase order

The "Proceed to purchase order" button in the success panel sends a `postMessage` to the parent window:

```javascript
window.parent.postMessage({type: 'sourcing_complete', quote_id: quoteId}, '*');
```

The parent page can listen for this and navigate accordingly. If the PO page has a direct URL, replace the `proceedToPO()` function body in `index.html` with:

```javascript
function proceedToPO(){
  window.parent.location.href = '#Page:Your_PO_Page_Name?quote_id=' + quoteId;
}
```

---

## Parallel widget

The MaterialIntel widget follows the same pattern and lives at:
```
sentrymetal1.github.io/materialintel-widget/MaterialIntel.html
```
Both widgets use the same SDK initialization approach and Zoho API call patterns.
