**PR to PO Compliance Validation**

Checks 3 through 6

Includes category-specific validation rules for:

- Insurance and certification requirements by spend category
- Rate card vs catalog pricing models

# CHECK 3: Supplier Existence & Status

## 3.1 Business Purpose

This check verifies that the supplier on the PR is known, active, and authorized to do business with the requesting entity. It prevents purchase orders from being issued to unknown vendors, blocked suppliers, or vendors not approved for specific legal entities or categories.

**Business Risk Mitigated:** Payments to fraudulent vendors, compliance violations from using debarred suppliers, audit findings for unapproved vendor usage, insurance and liability gaps.

## 3.2 Core Validation Rules

| **#** | **Rule Name** | **Validation Logic** | **Pass Criteria** | **Fail Action** |
| --- | --- | --- | --- | --- |
| **3.1** | **Supplier Exists** | PR supplier name matches vendor_name OR any entry in vendor_aliases\[\] in Supplier Master. Use LLM based matching | Match found with confidence >= 85% | FAIL: Unknown vendor - requires onboarding |
| **3.2** | **Supplier Status Active** | Matched supplier has status = 'Active'. Statuses that fail: Blocked, Inactive, Pending Approval, Debarred, Suspended. | Status = 'Active' | FAIL: Supplier not active - cannot issue PO |
| **3.3** | **Approved for Entity** | PR company_code exists in supplier's entities_served\[\] list. Some suppliers approved only for specific subsidiaries. | Entity in approved list | FAIL: Supplier not approved for this entity |
| **3.4** | **Insurance Current** | If category requires insurance: supplier.insurance_expiry_date >= today. Buffer recommended (today + PO duration). | Insurance valid OR not required for category | FAIL: Insurance expired - compliance risk |
| **3.5** | **Certifications Current** | Required certifications for category/supplier type have expiry_date >= today. Includes regulatory, quality, diversity certs. | All required certs valid OR none required | WARNING: Certification expired - may block PO |
| **3.6** | **Document-PR Supplier Match** | Supplier name extracted from attachment (quote, contract, SOW) matches PR supplier. Cross-validates that attachment belongs to this transaction. | Fuzzy match >= 85% OR alias match | FAIL: Attachment has different supplier than PR |

## 3.3 Insurance Requirements by Category

Insurance requirements vary significantly by spend category based on risk profile. On-site work and professional advisory carry higher risk than commodity purchases.

| **Category** | **Required Insurance** | **Typical Coverage** | **Rationale** |
| --- | --- | --- | --- |
| **MRO / Facilities** | General Liability, Workers Compensation | \$1M - \$5M | On-site work creates physical risk exposure |
| **Construction / CapEx** | General Liability, Workers Comp, Builders Risk, Umbrella | \$5M - \$10M | High-value projects with significant site liability |
| **Professional Services** | Professional Liability (Errors & Omissions) | \$1M - \$5M | Advisory risk, professional malpractice exposure |
| **IT Services** | Cyber Liability, Professional Liability, Tech E&O | \$2M - \$5M | Data breach exposure, system failure liability |
| **Logistics / Transportation** | Cargo Insurance, Auto Liability, Warehouse Legal | Per shipment value | Goods in transit, vehicle accidents |
| **Staffing / Contingent Labor** | Workers Compensation, Employment Practices Liability | \$1M+ | Co-employment risk, workplace incidents |
| **IT Hardware / Software** | Generally not required | -   | Low physical risk, product shipped |
| **Office Supplies / Indirect** | Generally not required | -   | Low risk commodity purchase |

## 3.4 Certification Requirements by Category

| **Category** | **Typical Required Certifications** | **Compliance Rationale** |
| --- | --- | --- |
| **MRO / Industrial** | ISO 9001, OSHA compliance, trade licenses (electrical, plumbing, HVAC) | Quality systems, workplace safety, legal authorization for trade work |
| **Facilities / Janitorial** | OSHA compliance, EPA compliance, state/local business licenses | Environmental regulations, hazardous materials handling |
| **Construction** | State contractor license, bonding certificate, OSHA 30-hour, prevailing wage certification | Legal requirement to perform construction work, Davis-Bacon compliance |
| **IT Services / Cloud** | SOC 2 Type II (issued within 12 months), ISO 27001, GDPR compliance attestation | Data security, privacy compliance, customer data protection |
| **Medical / Life Sciences** | FDA registration, ISO 13485, GxP compliance, DEA license (if controlled substances) | Regulatory compliance for medical devices, pharmaceuticals |
| **Food Services** | Health department permits, ServSafe certification, food handler licenses | Food safety regulations, public health |
| **Transportation / Logistics** | DOT registration, FMCSA authority, C-TPAT certification, hazmat endorsements | Federal transportation regulations, customs compliance |
| **Financial Services Vendors** | SOC 1 Type II, PCI-DSS (if card data), state financial licenses | Financial controls, data security for payment processing |
| **Diversity Suppliers** | MWBE, HUBZone, 8(a), SDVOSB, WBENC, NMSDC certifications | Diversity spend tracking, government contracting requirements |
| **Sustainability Programs** | ISO 14001, EcoVadis rating, CDP disclosure, Science-Based Targets | Environmental compliance, ESG reporting requirements |

## 3.5 Fields Required

| **Source** | **Field** | **Data Type** | **Used In Rule** |
| --- | --- | --- | --- |
| **FROM PR (System Data)** |     |     |     |
| PR Header | pr_suggested_supplier | String | 3.1 Supplier lookup, 3.6 Document cross-validation |
| PR Header | pr_company_code | String | 3.3 Entity authorization check |
| PR Header | pr_category_l1 | String | 3.4, 3.5 Determine insurance/cert requirements by category |
| **FROM SUPPLIER MASTER (Reference Data)** |     |     |     |
| Supplier Master | vendor_id | String | Unique identifier for matched supplier record |
| Supplier Master | vendor_name | String | 3.1 Primary name for fuzzy matching |
| Supplier Master | vendor_aliases\[\] | Array&lt;String&gt; | 3.1 Alternate names (IBM, IBM Corp, IBM Corporation) |
| Supplier Master | status | Enum | 3.2 Active \| Inactive \| Blocked \| Pending \| Debarred \| Suspended |
| Supplier Master | entities_served\[\] | Array&lt;String&gt; | 3.3 List of company codes authorized for this supplier |
| Supplier Master | insurance_policies\[\] | Array&lt;Object&gt; | 3.4 {policy_type, coverage_amount, expiry_date, carrier} |
| Supplier Master | certifications\[\] | Array&lt;Object&gt; | 3.5 {cert_type, cert_id, issue_date, expiry_date, issuing_body} |
| **FROM ATTACHMENT (Extracted)** |     |     |     |
| Quote/Contract/SOW | doc.supplier_name | String | 3.6 Cross-validation: attachment supplier must match PR supplier |
| **FROM CATEGORY CONFIG (Reference Data)** |     |     |     |
| Category Config | insurance_requirements\[category\] | Array&lt;Object&gt; | 3.4 {insurance_type, min_coverage, required: true/false} |
| Category Config | certification_requirements\[category\] | Array&lt;Object&gt; | 3.5 {cert_type, required: true/false, max_age_months} |

## 3.6 Check Output Schema

| **Output Field** | **Data Type** | **Description** |
| --- | --- | --- |
| check_3_status | Enum | PASS \| FAIL \| WARNING - overall check result |
| supplier_found | Boolean | Whether supplier was matched in Supplier Master |
| matched_vendor_id | String | Vendor ID of matched supplier (null if not found) |
| match_confidence | Float | Fuzzy match confidence score (0.0 - 1.0) |
| supplier_status | String | Status of matched supplier (Active, Blocked, etc.) |
| entity_authorized | Boolean | Whether supplier is approved for PR entity |
| insurance_valid | Boolean | Whether required insurance is current |
| insurance_gaps\[\] | Array&lt;String&gt; | List of missing or expired insurance types |
| certifications_valid | Boolean | Whether required certifications are current |
| certification_gaps\[\] | Array&lt;String&gt; | List of missing or expired certifications |
| document_supplier_match | Boolean | Whether attachment supplier matches PR supplier |
| failure_reasons\[\] | Array&lt;String&gt; | List of specific failures for remediation |

# CHECK 4: Item Description Match

## 4.1 Business Purpose

This check verifies that the items or services described on the PR accurately reflect what was quoted or contracted. It prevents scope drift, unauthorized substitutions, and data entry errors where the PR describes something materially different from the supporting documentation.

**Business Risk Mitigated:** Receiving wrong items, payment disputes, contract compliance violations, production disruptions from incorrect materials, audit findings for scope creep.

## 4.2 Core Validation Rules

| **#** | **Rule Name** | **Validation Logic** | **Pass Criteria** | **Fail Action** |
| --- | --- | --- | --- | --- |
| **4.1** | **Description Semantic Match** | PR line description semantically matches corresponding quote/contract line. LLM compares meaning, not exact text. | Semantic similarity >= 80% | WARNING: Description mismatch - verify items are equivalent |
| **4.3** | **Manufacturer Match** | If manufacturer specified on both PR and quote, must match or be verified equivalent (OEM vs distributor name). | Manufacturer matches OR not specified | WARNING: Different manufacturer - verify equivalence |
| **4.4** | **Line Count Alignment** | Number of line items on PR should align with quote. PR may be subset (partial order) but not superset. | PR lines <= Quote lines, all PR lines have match | FAIL: PR has items not on quote |
| **4.6** | **Scope Coverage (Services)** | For services, PR description must fall within SOW scope definition. Semantic comparison. | PR scope within SOW scope | FAIL: Services outside contracted scope |

## 4.4 Catalog/SKU Applicability by Category

| **Category** | **Catalog Type** | **Coverage Check Logic** |
| --- | --- | --- |
| **IT Hardware** | Approved product list, standard configs | PR item must be on IT-approved hardware list. Non-standard requires exception approval. |
| **MRO** | Storeroom catalog, VMI items | Check against storeroom master. VMI items should auto-replenish, not PR. |
| **Office Supplies** | Punch-out catalog | Should match catalog item from punch-out. Off-catalog triggers review. |
| **Lab Supplies** | Approved vendor catalog, controlled items list | Verify against approved vendor catalog. Controlled items require additional documentation. |
| **Services** | Rate card roles (not SKUs) | Check role/resource type against approved rate card, not product catalog. |

## 4.5 Fields Required

| **Source** | **Field** | **Data Type** | **Used In Rule** |
| --- | --- | --- | --- |
| **FROM PR LINE ITEMS (System Data)** |     |     |     |
| PR Line | pr_line_items\[\].line_number | Integer | 4.4 Line matching and ordering |
| PR Line | pr_line_items\[\].description | String | 4.1 Semantic match target, 4.6 Scope coverage |
| PR Line | pr_line_items\[\].manufacturer | String | 4.3 Manufacturer validation |
| PR Header | pr_category_l1 | String | Determine part number criticality level |
| **FROM QUOTE (Extracted)** |     |     |     |
| Quote | quote.line_items\[\].line_number | Integer | 4.4 Line matching |
| Quote | quote.line_items\[\].description | String | 4.1 Semantic comparison source |
| Quote | quote.line_items\[\].manufacturer | String | 4.3 Manufacturer comparison |
| **FROM SOW (Extracted - Services Only)** |     |     |     |
| SOW | sow.scope_summary | String | 4.6 Scope coverage semantic match |
| SOW | sow.in_scope\[\] | Array&lt;String&gt; | 4.6 Explicit in-scope items for matching |
| SOW | sow.out_of_scope\[\] | Array&lt;String&gt; | 4.6 Explicit exclusions to check against |
| **FROM CONTRACT (Reference Data)** |     |     |     |
| Contract | contract.catalog_items\[\] | Array&lt;Object&gt; | 4.5 {sku, description, manufacturer} for catalog lookup |

# CHECK 5: Quantity & Unit of Measure Match

## 5.1 Business Purpose

This check verifies that quantities and units of measure on the PR match the quote and comply with contract ordering constraints. It catches data entry errors and prevents ordering incorrect quantities (e.g., 100 vs 10) or in wrong units (e.g., EA vs BOX).

**Business Risk Mitigated:** Over-ordering (excess inventory, cash flow), under-ordering (stockouts, production delays), supplier rejection (below MOQ), incorrect pricing (per-each vs per-case confusion).

## 5.2 Core Validation Rules

| **#** | **Rule Name** | **Validation Logic** | **Pass Criteria** | **Fail Action** |
| --- | --- | --- | --- | --- |
| **5.1** | **Quantity Match** | PR quantity equals quote quantity for each matched line item. | Quantities match exactly | WARNING: Quantity differs from quote |
| **5.2** | **Partial Order Validation** | If policy allows partial orders: PR qty <= Quote qty. PR cannot exceed quoted quantity. | PR qty <= Quote qty | FAIL: PR quantity exceeds quote |
| **5.3** | **UOM Match or Equivalent** | Unit of measure matches or is convertible equivalent. EA=EACH=PC, CS=CASE, BX=BOX. | UOM matches or equivalent found | FAIL: Incompatible units of measure |
| **5.7** | **UOM-Price Alignment** | Ensure PR UOM matches quote pricing UOM to prevent per-each vs per-case pricing errors. | UOM consistent with pricing | FAIL: UOM mismatch may cause pricing error |

## 5.5 Category-Specific Validation Logic

| **Category** | **Special Validation Rules** |
| --- | --- |
| **MRO / Office Supplies** | If quote UOM is case (CS) but PR UOM is each (EA), WARNING and calculate: Does pr_qty = quote_qty × units_per_case? Common error: ordering 100 EA when quote was for 100 CS. |
| **Services (T&M)** | UOM should be time-based (HR, DAY, WK, MO). If UOM is not time-based, WARNING: T&M services should use time units. Check that total hours are reasonable for duration. |
| **Chemicals / Hazmat** | Verify quantity against hazmat storage limits and permits. Large quantities may require safety review. Check container size compatibility. |
| **Print / Marketing** | Quantity must be valid print run increment (often 500 or 1000). Order 1,750 = invalid. Check warehouse capacity for storage. |

## 5.6 Fields Required

| **Source** | **Field** | **Data Type** | **Used In Rule** |
| --- | --- | --- | --- |
| **FROM PR LINE ITEMS (System Data)** |     |     |     |
| PR Line | pr_line_items\[\].quantity | Decimal | 5.1-5.6 All quantity validation rules |
| PR Line | pr_line_items\[\].uom | String | 5.3, 5.7 UOM equivalence and alignment |
| PR Header | pr_category_l1 | String | Determine category-specific MOQ/increment rules |
| **FROM QUOTE (Extracted)** |     |     |     |
| Quote | quote.line_items\[\].quantity | Decimal | 5.1, 5.2 Quantity comparison baseline |
| Quote | quote.line_items\[\].uom | String | 5.3, 5.7 UOM comparison |
| **FROM CONTRACT (Reference Data)** |     |     |     |
| Contract | contract.minimum_order_qty | Decimal | 5.4 MOQ check (contract level) |
| Contract | contract.maximum_order_qty | Decimal | 5.5 Max qty check (contract level) |
| Contract | contract.order_increment | Decimal | 5.6 Valid increment check |
| Contract | contract.units_per_case | Decimal | 5.3 UOM conversion calculation |
| **FROM ITEM MASTER (Reference Data)** |     |     |     |
| Item Master | item.default_uom | String | 5.3 Standard UOM for item |

# CHECK 6: Unit Price Match

## 6.1 Business Purpose

This check verifies that unit prices on the PR match quoted prices, contracted prices, or approved rate cards. It is critical for cost control - ensuring negotiated pricing is being captured and preventing markup, keying errors, or circumvention of contract terms.

**Business Risk Mitigated:** Overpayment to suppliers, leakage of negotiated savings, contract non-compliance, budget overruns, margin erosion, audit findings for pricing discrepancies.

## 6.2 Core Validation Rules

| **#** | **Rule Name** | **Validation Logic** | **Pass Criteria** | **Fail Action** |
| --- | --- | --- | --- | --- |
| **6.1** | **PR-Quote Price Match** | PR unit price equals quote unit price for matched line items. Allow tolerance for rounding. | Within +/- 1% or +/- \$1 | FAIL: Price mismatch vs quote |
| **6.2** | **Contract Price Ceiling** | If contract exists with pricing, PR price <= contract price. May not exceed contracted rates. | PR price <= contract price | FAIL: Price exceeds contract |
| **6.3** | **Rate Card Compliance** | For T&M services, hourly/daily rate matches approved rate card for the role/level. | Rate <= approved rate card | FAIL: Rate exceeds rate card |
| **6.4** | **Discount Applied** | If contract specifies discount % off list, verify discount is correctly applied. | Discount properly calculated | WARNING: Expected discount not applied |
| **6.5** | **No Markup vs Quote** | PR price should not exceed quote price. Internal markup not allowed. | PR price <= Quote price | FAIL: PR price exceeds quote |
| **6.6** | **Currency Alignment** | If PR and quote currencies differ, convert using approved exchange rate and compare. | Converted prices match | WARNING: Currency mismatch requires review |

## 6.6 Fields Required

| **Source** | **Field** | **Data Type** | **Used In Rule** |
| --- | --- | --- | --- |
| **FROM PR LINE ITEMS (System Data)** |     |     |     |
| PR Line | pr_line_items\[\].unit_price | Decimal | 6.1-6.7 All price comparison rules |
| PR Header | pr_currency | String | 6.6 Currency alignment check |
| PR Header | pr_category_l1 | String | Determine pricing model (rate card vs catalog) |
| **FROM QUOTE (Extracted)** |     |     |     |
| Quote | quote.line_items\[\].unit_price | Decimal | 6.1, 6.5 Quote price comparison |
| Quote | quote.currency | String | 6.6 Currency for conversion |
| **FROM CONTRACT (Reference Data)** |     |     |     |
| Contract | contract.price_table\[\].contract_price | Decimal | 6.2 Contract price ceiling |
| Contract | contract.price_table\[\].list_price | Decimal | 6.4 List price for discount calc |
| Contract | contract.discount_percentage | Decimal | 6.4 Expected discount % |
| **FROM APPROVED RATE CARD (Reference Data)** |     |     |     |
| Rate Card | rate_card\[\].role | String | 6.3 Role name for matching |
| Rate Card | rate_card\[\].approved_rate | Decimal | 6.3 Maximum approved rate for role |
| Rate Card | rate_card\[\].effective_date | Date | 6.3 Ensure using current rate card version |
| Rate Card | rate_card\[\].expiry_date | Date | 6.3 Check rate card hasn't expired |

# Summary: Checks 3-6 Category Applicability

| **Category** | **Check 3: Supplier** | **Check 4: Description** | **Check 5: Quantity** | **Check 6: Price** |
| --- | --- | --- | --- | --- |
| **IT Hardware** | Standard validation. Insurance not required. | CRITICAL: Part number exact match required. | Simple: EA UOM, typically no MOQ. | Catalog: Discount off list price. |
| **MRO / Facilities** | Insurance required (GL, WC). Trade licenses for on-site. | CRITICAL: Part + MPN exact match. | Complex: Case/pack UOM, MOQ common. | Contract unit price. |
| **Professional Services** | E&O insurance. SOC2 if data access. | Scope semantic match (4.6). | T&M: Time UOM, validate hours. | RATE CARD: Role-based rates. |
| **IT Services** | Cyber + E&O insurance. SOC2/ISO27001. | Scope semantic match. | T&M: Time UOM. | RATE CARD: Technical roles. |
| **Construction** | Full insurance. License, bonding, OSHA. | Scope match to contract. | Per project/milestone. | Trade rates + prevailing wage. |
| **Direct Materials** | Standard. Quality certs (ISO). | CRITICAL: BOM exact match. | Lot size, pallet, check vs forecast. | Negotiated/index-based. |
| **Office Supplies** | Standard. No special requirements. | LOW: Fuzzy match OK. | Simple: Watch case vs each. | Catalog price. |
| **Staffing** | WC, EPL insurance. | Job description match. | Time UOM. | Bill rate = pay + markup %. |

## Check Dependencies

| **Check** | **Depends On** | **Reason** |
| --- | --- | --- |
| **3** | Check 1, Check 2 | Need document classification (Check 1) and validity (Check 2) before supplier extraction from docs. |
| **4** | Check 1, Check 2, Check 3 | Need valid documents and confirmed supplier before comparing item descriptions. |
| **5** | Check 4 | Need matched line items (Check 4) before comparing quantities between PR and quote. |
| **6** | Check 4, Check 5 | Need matched items (Check 4) and verified quantities (Check 5) for meaningful price comparison. |