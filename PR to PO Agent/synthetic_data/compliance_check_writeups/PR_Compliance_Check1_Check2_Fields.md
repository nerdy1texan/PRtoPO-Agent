**PR to PO Compliance Validation**

Field Extraction & Validation Rules

**Check 1: Attachment Existence & Document Classification**

**Check 2: Document Validity & Applicability**

*Version 1.0 \| February 2026*

CHECK 1: Attachment Existence & Document Classification

1.1 Business Purpose

This check acts as a GATEKEEPER. Before any validation can occur, we
must confirm that the PR has the required supporting documentation. The
documentation requirements are POLICY-DRIVEN and vary based on PR
characteristics (value, category, supplier type).

**Key Insight:** Check 1 does NOT extract detailed fields from
documents. It only identifies WHAT TYPE of documents are attached and
whether they satisfy policy requirements.

1.2 Policy-Driven Documentation Requirements

Different PR characteristics trigger different documentation
requirements. These are configured in the Threshold Config / Purchasing
Policy:

  ------------------ -------------------------- --------------------------
  **PR               **Documentation Required** **Policy Rationale**
  Characteristic**                              

  **Value \<         None required (P-Card      Low risk, petty cash /
  \$5,000**          eligible)                  corporate card purchase

  **Value \$5,000 -  Minimum 1 Quote            Basic price validation,
  \$25,000**                                    supplier confirmation

  **Value \$25,000 - Minimum 2 Quotes           Competitive pricing
  \$50,000**                                    validation

  **Value \>         3 Competitive Quotes OR    Full competitive bidding
  \$50,000**         Active Contract            OR pre-negotiated terms

  **Services         SOW or Service Agreement   Scope definition prevents
  Category**         required                   scope creep

  **Capital          Quote + Business Case /    CapEx approval requires
  Equipment**        Justification              ROI justification

  **Single Source    1 Quote + Single Source    Document why competitive
  (\>threshold)**    Justification Form         bid not possible

  **New Supplier**   Quote + Supplier           Ensure supplier setup in
                     Registration / Onboarding  master data
                     Form                       

  **Contract         Contract ID must exist in  Validate contract exists
  Referenced (not    Contract Master            even if not attached
  attached)**                                   
  ------------------ -------------------------- --------------------------

1.3 Document Types & Classification Markers

The LLM classifies each attachment based on document markers. It does
NOT need to extract specific field values - only identify document type.

  ----------------- ------------- ---------------------------- ------------------
  **Document Type** **Formats**   **Classification Markers     **Valid as Primary
                                  (What LLM Looks For)**       Support?**

  **Quotation /     PDF, Excel,   Keywords: \'Quote\',         YES - for Goods
  Quote /           Word, PNG,    \'Quotation\', \'Proposal\', 
  Proposal**        Email         \'Valid Until\'. Has:        
                                  supplier letterhead, line    
                                  items with prices,           
                                  quantities, total amount,    
                                  reference number             

  **Contract / MSA  PDF, Word     Keywords: \'Agreement\',     YES
  / Framework                     \'Contract\', \'Master       
  Agreement**                     Service Agreement\'. Has:    
                                  parties clause,              
                                  effective/expiry dates,      
                                  signatures, terms &          
                                  conditions, recitals         

  **Statement of    PDF, Word     Keywords: \'Statement of     YES - for Services
  Work (SOW)**                    Work\', \'Scope of Work\',   
                                  \'SOW\'. Has: scope section, 
                                  deliverables list,           
                                  milestones, timeline, may    
                                  reference parent MSA         

  **Service         PDF, Word     Keywords: \'Service          YES - for Services
  Agreement / SA**                Agreement\', \'Service Level 
                                  Agreement\'. Has: service    
                                  descriptions, SLAs, coverage 
                                  locations, response times    

  **Invoice**       PDF           Keywords: \'Invoice\',       NO - ATF Detection
                                  \'Bill To\', \'Payment       Only
                                  Due\'. Has: invoice number,  
                                  invoice date, payment terms, 
                                  remit-to address. RED FLAG   
                                  if invoice date \< PR date   
                                  (ATF)                        

  **Competitive Bid Excel, PDF,   Keywords: \'Bid              Supporting -
  Summary**         Word          Comparison\', \'Quote        proves 3-bid
                                  Comparison\', \'Supplier     
                                  Evaluation\'. Has: multiple  
                                  supplier names, comparison   
                                  table, scoring/ranking,      
                                  selection justification      

  **Single Source   PDF, Word,    Keywords: \'Sole Source\',   Supporting - with
  Justification**   Form          \'Single Source\',           1 quote
                                  \'Justification\'. Has:      
                                  reason for single source,    
                                  approver signatures,         
                                  competitive alternatives     
                                  considered                   

  **Business Case / PDF, Word,    Keywords: \'Business Case\', NO - Supporting
  Justification**   PPT           \'ROI\', \'Justification\'.  only
                                  Has: problem statement,      
                                  proposed solution,           
                                  cost-benefit analysis,       
                                  approval request             

  **Specification / PDF, Word     Technical requirements,      NO - Supporting
  Technical Doc**                 specifications, diagrams. NO only
                                  pricing, NO commercial       
                                  terms, NO dates              

  **Email /         PNG, PDF,     Email format with            DEPENDS - Parse
  Screenshot**      MSG, EML      To/From/Date headers. May    for embedded quote
                                  contain embedded quote -     
                                  parse for quote markers      
  ----------------- ------------- ---------------------------- ------------------

1.4 Fields Required for Check 1

**Minimal fields needed - this is CLASSIFICATION, not full extraction:**

  ----------------- ----------------------------- ------------ --------------------------
  **Source**        **Field**                     **Data       **Purpose**
                                                  Type**       

  **FROM PR HEADER                                             
  (System Data)**                                              

  PR Header         pr_id                         String       Unique identifier for
                                                               logging/tracking

  PR Header         total_value                   Decimal      Determines which threshold
                                                               band applies

  PR Header         category_l1                   String       Services vs Goods
                                                               determination for SOW
                                                               requirement

  PR Header         contract_reference            String       If contract ID cited but
                                                  (nullable)   not attached, lookup in
                                                               Contract Master

  PR Header         supplier_status               String       New vs Existing supplier
                                                               for registration form
                                                               requirement

  **FROM PR                                                    
  ATTACHMENTS                                                  
  (System Data)**                                              

  Attachments       attachment_count              Integer      Basic existence check
                                                               (must be \>= 1)

  Attachments       filename                      String       Display in results, may
                                                               hint at content type

  Attachments       file_extension                String       Route to correct parser
                                                               (PDF vs Excel vs Image)

  Attachments       file_size_bytes               Integer      Skip empty files, set
                                                               processing limits

  **FROM LLM                                                   
  (Classification                                              
  Output)**                                                    

  LLM Output        document_type                 Enum         Quotation \| Contract \|
                                                               SOW \| SA \| Invoice \|
                                                               BidSummary \|
                                                               Justification \| Spec \|
                                                               Other

  LLM Output        classification_confidence     Float 0-1    Confidence \< 0.8 triggers
                                                               Human-in-Loop review

  LLM Output        classification_reason         String       Brief explanation of why
                                                               this classification (audit
                                                               trail)

  **FROM THRESHOLD                                             
  CONFIG (Reference                                            
  Data)**                                                      

  Threshold Config  quote_required_above          Decimal      Value above which at least
                                                               1 quote required (e.g.,
                                                               \$5,000)

  Threshold Config  competitive_quote_threshold   Decimal      Value above which 3 quotes
                                                               required (e.g., \$50,000)

  Threshold Config  contract_satisfies_quotes     Boolean      Whether valid contract can
                                                               substitute for 3 quotes

  Threshold Config  services_require_sow          Boolean      Whether Services category
                                                               requires SOW/SA
  ----------------- ----------------------------- ------------ --------------------------

1.5 Check 1 Output Schema

Check 1 produces a classification result that feeds into Check 2:

  -------------------------- ----------------- ---------------------------------------
  **Output Field**           **Data Type**     **Description**

  check_1_status             Enum              PASS \| FAIL \| NEEDS_REVIEW

  attachments_found          Integer           Count of attachments on PR

  classified_documents\[\]   Array             Array of {filename, document_type,
                                               confidence, reason}

  valid_support_docs_found   Integer           Count of documents classified as
                                               Quote/Contract/SOW/SA

  policy_requirements_met    Boolean           Whether documentation meets threshold
                                               policy for this PR value/category

  missing_requirements\[\]   Array\<String\>   List of what\'s missing (e.g.,
                                               \'Requires 3 quotes, only 1 found\')

  invoice_detected           Boolean           Red flag for potential After-the-Fact
                                               (ATF) - triggers Check 11
  -------------------------- ----------------- ---------------------------------------

CHECK 2: Document Validity & Applicability

2.1 Business Purpose

Once Check 1 confirms WHAT documents are attached, Check 2 verifies each
document is VALID (correct dates, status) and APPLICABLE (correct
supplier, entity, category, geography for this specific PR).

**Key Insight:** This check REQUIRES extracting specific fields from
documents. Different document types require different extraction
schemas.

2.2 Quotation Validity Rules

  --------------- ----------------- --------------- -------------- ----------------------
  **Rule**        **Validation      **Threshold**   **Fail         **Fields Required**
                  Logic**                           Action**       

  **Quote Date    Quote must be     quote_date \<=  FAIL - Quote   quote_date,
  Sequence**      dated on or       pr_date         doesn\'t exist pr_issue_date
                  before PR date. A                 yet            
                  quote dated AFTER                                
                  the PR indicates                                 
                  process                                          
                  violation.                                       

  **Quote Not     Quote             validity_date   FAIL - Expired quote_validity_date,
  Expired**       validity/expiry   \>= pr_date     quote          pr_issue_date
                  date must be on                                  
                  or after PR date.                                
                  Expired quotes                                   
                  cannot support a                                 
                  PR.                                              

  **Quote         Quote must be     30, 60, or 90   WARNING -      quote_date,
  Staleness**     recent. Stale     days (policy)   Request        pr_issue_date,
                  quotes may have                   refresh        staleness_threshold
                  outdated pricing                                 
                  that no longer                                   
                  reflects market.                                 

  **Quote         Quote must be     Exact or fuzzy  FAIL - Wrong   quote_customer_name,
  Addressee       addressed to the  match           entity         pr_company_code
  Match**         correct legal                                    
                  entity placing                                   
                  the PR.                                          
                  Cross-entity                                     
                  quotes may not be                                
                  valid.                                           

  **Quote         Quote currency    Exact match     WARNING -      quote_currency,
  Currency        must match PR                     Needs          pr_currency
  Match**         currency.                         conversion     
                  Mismatch requires                                
                  exchange rate                                    
                  validation.                                      

  **Quote Total   Quote total       +/- 1% or +/-   FAIL - Price   quote_total,
  Match**         should match PR   \$100           mismatch       pr_total_value
                  total within                                     
                  tolerance.                                       
                  Significant                                      
                  variance                                         
                  indicates data                                   
                  entry error.                                     

  **Quote         Supplier on quote Fuzzy match \>= FAIL - Wrong   quote_supplier,
  Supplier        must match        85%             supplier       pr_supplier
  Match**         supplier on PR.                                  
                  Names may not be                                 
                  identical (IBM vs                                
                  IBM Corp).                                       
  --------------- ----------------- --------------- -------------- ----------------------

2.3 Contract Validity Rules

  ---------------- ---------------- ----------------- -------------- ---------------------
  **Rule**         **Validation     **Threshold**     **Fail         **Fields Required**
                   Logic**                            Action**       

  **Contract       Contract must    effective_date    FAIL - Not yet effective_date,
  Effective**      have started on  \<= pr_date       effective      pr_issue_date
                   or before PR                                      
                   date. Cannot use                                  
                   a contract that                                   
                   hasn\'t begun.                                    

  **Contract Not   Contract         expiration_date   FAIL -         expiration_date,
  Expired**        expiration must  \>= pr_date       Contract       pr_issue_date
                   be on or after                     expired        
                   PR date. Expired                                  
                   contracts are                                     
                   invalid.                                          

  **Covers         Contract must    expiration \>=    WARNING - May  expiration_date,
  Delivery         still be active  need_by           expire         pr_need_by_date
  Period**         when                                              
                   goods/services                                    
                   are delivered.                                    
                   Expiring before                                   
                   delivery is                                       
                   risky.                                            

  **Contract       Contract must be status =          FAIL -         contract_status
  Status Active**  in Active        \'Active\'        Inactive       
                   status.                            contract       
                   Suspended,                                        
                   Terminated, or                                    
                   Draft contracts                                   
                   are not valid.                                    

  **Ceiling Not    Contract         remaining \>=     FAIL - Ceiling contract_remaining,
  Exceeded**       remaining value  pr_total          exceeded       pr_total_value
                   must cover PR                                     
                   amount.                                           
                   Exceeding                                         
                   ceiling requires                                  
                   amendment.                                        

  **Near-Expiry    Flag if contract 30, 60, 90 days   WARNING -      expiration_date,
  Warning**        expires soon     (policy)          Expiring soon  pr_issue_date
                   after PR. Risk                                    
                   of expiring                                       
                   before                                            
                   PO/delivery.                                      

  **Near-Ceiling   Flag if contract 10% or 20%        WARNING -      contract_remaining,
  Warning**        ceiling is       remaining         Ceiling low    contract_value
                   nearly                                            
                   exhausted. May                                    
                   need new                                          
                   contract soon.                                    

  **Minimum Order  Some contracts   pr_total \>=      WARNING -      contract_minimum,
  Value**          have minimum     minimum           Below minimum  pr_total_value
                   order                                             
                   requirements. PR                                  
                   below minimum                                     
                   may be rejected                                   
                   by supplier.                                      

  **Maximum PO     Some contracts   pr_total \<=      FAIL - Exceeds contract_max_po,
  Value**          cap individual   max_po            max PO         pr_total_value
                   PO value. PR                                      
                   exceeding cap                                     
                   needs split or                                    
                   special                                           
                   approval.                                         
  ---------------- ---------------- ----------------- -------------- ---------------------

2.4 Coverage / Applicability Rules

  --------------- ---------------------- ------------------------ ----------------------------
  **Rule**        **Validation Logic**   **Match Type**           **Fields Required**

  **Entity        Contract/SOW must      IN covered_entities\[\]  pr_company_code,
  Coverage**      cover the legal entity                          doc.covered_entities\[\]
                  (company code) placing                          
                  the PR. Cross-entity                            
                  usage may violate                               
                  contract terms.                                 

  **Category      Contract must cover    IN                       pr_category_l1,
  Coverage**      the spend category of  covered_categories\[\]   doc.covered_categories\[\]
                  the PR. Using contract                          
                  for out-of-scope                                
                  category is breach.                             

  **Geographic    Contract must cover    IN covered_regions\[\]   pr_ship_to_region,
  Coverage**      the ship-to region.                             doc.covered_regions\[\]
                  Geographic                                      
                  restrictions may apply                          
                  to pricing/delivery.                            

  **SKU / Item    If contract has        IN covered_skus\[\]      pr_material_code,
  Coverage**      specific approved item                          doc.covered_skus\[\]
                  list (catalog), PR                              
                  items must be on that                           
                  list.                                           

  **Scope         For SOW, the PR line   Semantic similarity      pr_line_description,
  Semantic Match  description must fall                           sow.scope_summary
  (SOW)**         within defined scope.                           
                  LLM performs semantic                           
                  comparison.                                     
  --------------- ---------------------- ------------------------ ----------------------------

2.5 Supplier Match Rules

  ------------------ ---------------------- --------------------- ------------------------------
  **Rule**           **Validation Logic**   **Match Type**        **Fields Required**

  **Document-PR      Supplier name on       Fuzzy \>= 85% OR      doc.supplier_name, pr_supplier
  Match**            document must match    alias                 
                     supplier on PR. Names                        
                     may differ (IBM vs IBM                       
                     Corporation).                                

  **Supplier in      PR supplier must exist Exact or alias match  pr_supplier, vendor_name,
  Master**           in Supplier Master.                          vendor_aliases\[\]
                     Missing supplier =                           
                     onboarding required.                         

  **Supplier Status  Supplier must be       status = \'Active\'   supplier.status
  Active**           Active. Blocked,                             
                     Inactive, or Pending                         
                     suppliers cannot                             
                     receive POs.                                 

  **Approved for     Supplier must be       IN                    pr_company_code,
  Entity**           approved to transact   entities_served\[\]   supplier.entities_served\[\]
                     with the PR legal                            
                     entity.                                      

  **Insurance        Supplier\'s insurance  expiry \>= today      supplier.insurance_expiry
  Current**          coverage must be                             
                     current. Expired                             
                     insurance may block                          
                     PO.                                          

  **Certifications   Supplier\'s required   expiry \>= today      supplier.cert_expiry
  Current**          certifications must be                       
                     current (ISO, SOC2,                          
                     diversity, etc.).                            
  ------------------ ---------------------- --------------------- ------------------------------

2.6 Hierarchical Validity Rules

  ----------------- --------------------------------- ---------------------------
  **Rule**          **Validation Logic**              **Fields Required**

  **Quote Contract  If quote cites a contract ID      quote.contract_reference,
  Reference**       (e.g., \'Per MSA-2024-001\'),     Contract Master lookup
                    that contract must exist in       
                    Contract Master and pass all      
                    validity rules.                   

  **SOW Parent      If SOW references a parent MSA,   sow.parent_contract_id,
  MSA**             the MSA must be valid (effective, Contract Master lookup
                    not expired, active). Invalid MSA 
                    invalidates the SOW.              

  **Amendment       If document is a contract         doc.parent_contract_id,
  Parent**          amendment, the parent contract    Contract Master lookup
                    must exist and be Active. Cannot  
                    amend non-existent contract.      

  **Blanket PO      For release orders against        blanket_po_remaining,
  Release**         Blanket PO, sum of all releases   pr_total_value
                    (including this PR) must not      
                    exceed Blanket ceiling.           
  ----------------- --------------------------------- ---------------------------

Field Extraction Schemas by Document Type

3.1 Quotation - Fields to Extract for Check 2

  ---------------------------- ------------ --------------------- ---------------------
  **Field**                    **Data       **Used In Rule**      **Example Value**
                               Type**                             

  **Required for Validity                                         
  Checks**                                                        

  supplier_name                String       Quote Supplier Match, CDW Corporation
                                            Document-PR Match     

  quote_date                   Date         Quote Date Sequence,  2026-01-15
                                            Quote Staleness       

  quote_validity_date          Date         Quote Not Expired     2026-03-15

  quote_validity_days          Integer      Calculate expiry if   60
                                            no explicit date      
                                            (quote_date + days)   

  customer_name                String       Quote Addressee Match Acme Corporation

  currency                     String       Quote Currency Match  USD

  total_amount                 Decimal      Quote Total Match     77,101.06

  contract_reference           String       Quote Contract        MSA-2024-CDW-001
                               (nullable)   Reference             
                                            (hierarchical)        

  **Optional / Supporting**                                       

  quote_reference_number       String       Audit trail,          Q-2026-00142
                                            deduplication         

  supplier_address             String       Legal entity          200 N Milwaukee Ave,
                                            verification (if      Vernon Hills IL 60061
                                            needed)               

  **Needed for Later Checks                                       
  (Extract Now, Validate                                          
  Later)**                                                        

  line_items\[\].description   String       Check 4: Item         Lenovo ThinkPad P16
                                            Description Match     Gen 2

  line_items\[\].part_number   String       Check 4: Item         21FA000QUS
                                            Description Match     

  line_items\[\].quantity      Decimal      Check 5: Quantity/UOM 25

  line_items\[\].unit_price    Decimal      Check 6: Unit Price   2,849.00

  payment_terms                String       Check 8: Payment      Net 30
                                            Terms                 

  delivery_terms               String       Check 12: Delivery    2-3 weeks ARO
                                            Date                  
  ---------------------------- ------------ --------------------- ---------------------

3.2 Contract - Fields to Extract for Check 2

  -------------------------- ----------------- --------------------- ---------------------
  **Field**                  **Data Type**     **Used In Rule**      **Example Value**

  **Required for Validity                                            
  Checks**                                                           

  contract_id                String            Identification,       CNT-2024-0892
                                               Cross-reference       

  contract_status            Enum              Contract Status       Active \| Draft \|
                                               Active                Suspended \|
                                                                     Terminated \| Expired

  supplier_name              String            Document-PR Match     CDW Corporation

  effective_date             Date              Contract Effective    2024-01-01

  expiration_date            Date              Contract Not Expired, 2026-12-31
                                               Covers Delivery,      
                                               Near-Expiry           

  **Required for Coverage                                            
  Checks**                                                           

  covered_entities\[\]       Array\<String\>   Entity Coverage       \[US01, US02, CA01\]

  covered_categories\[\]     Array\<String\>   Category Coverage     \[IT-HW, IT-SW,
                                                                     IT-Services\]

  covered_regions\[\]        Array\<String\>   Geographic Coverage   \[North America,
                                                                     EMEA\]

  covered_skus\[\]           Array\<String\>   SKU/Item Coverage (if \[21FA000QUS,
                                               applicable)           40AY0090US\]

  **Required for Financial                                           
  Checks**                                                           

  contract_value             Decimal           Near-Ceiling          5,000,000.00
                                               calculation           

  contract_value_utilized    Decimal           Remaining calculation 1,750,000.00
                                               (prefer Contract      
                                               Master)               

  contract_value_remaining   Decimal           Ceiling Not Exceeded, 3,250,000.00
                                               Near-Ceiling          

  minimum_order_value        Decimal           Minimum Order Value   500.00

  maximum_po_value           Decimal           Maximum PO Value      250,000.00

  **Required for                                                     
  Hierarchical Checks**                                              

  parent_contract_id         String (nullable) Amendment Parent      CNT-2024-0001
                                               Valid                 

  contract_type              String            Processing logic      MSA \| Blanket PO \|
                                               (MSA, Blanket,        Purchase Agreement \|
                                               Amendment)            Amendment

  **Needed for Later                                                 
  Checks**                                                           

  payment_terms              String            Check 8: Payment      Net 30
                                               Terms                 

  price_table\[\]            Array             Check 6: Unit Price   \[{sku, description,
                                               validation against    unit_price}\]
                                               contract pricing      
  -------------------------- ----------------- --------------------- ---------------------

3.3 SOW / Service Agreement - Fields to Extract

  --------------------- ------------ --------------------- ---------------------
  **Field**             **Data       **Used In Rule**      **Example Value**
                        Type**                             

  sow_id / agreement_id String       Identification        SOW-2026-MKT-001

  supplier_name         String       Document-PR Match     Acme Consulting LLC

  client_legal_entity   String       Entity Coverage       MegaCorp US
                                                           Operations

  effective_date        Date         SOW Effective         2026-01-15

  end_date              Date         SOW Not Expired       2026-04-30

  scope_summary         String       Scope Semantic Match  Full-service digital
                                                           marketing campaign
                                                           including SEO, PPC,
                                                           social media\...

  total_value           Decimal      Value comparison,     125,000.00
                                     ceiling check         

  not_to_exceed         Decimal      NTE ceiling (for T&M) 150,000.00

  currency              String       Currency Match        USD

  parent_contract_id    String       SOW Parent MSA Valid  MSA-2024-ACME-001
                        (nullable)                         

  payment_terms         String       Check 8: Payment      Net 30 upon milestone
                                     Terms                 completion

  rate_card\[\]         Array        Check 6: Rate         \[{role: \'Senior
                                     validation (T&M)      Consultant\', rate:
                                                           225, unit:
                                                           \'Hour\'}\]
  --------------------- ------------ --------------------- ---------------------

3.4 Reference Data: Supplier Master

  --------------------------- ----------------- ------------------------------------------
  **Field**                   **Data Type**     **Used In Rule**

  vendor_id                   String            Unique identifier for lookup and
                                                cross-reference

  vendor_name                 String            Primary name for matching against PR
                                                supplier and document supplier

  vendor_aliases\[\]          Array\<String\>   Alternate names for fuzzy matching
                                                (\'IBM\', \'IBM Corporation\',
                                                \'International Business Machines\')

  status                      Enum              Active \| Inactive \| Blocked \| Pending
                                                --- Supplier Status Active rule

  entities_served\[\]         Array\<String\>   Which company codes this supplier can
                                                transact with --- Approved for Entity rule

  insurance_expiry_date       Date              Insurance Current rule --- must be \>=
                                                today

  certification_expiry_date   Date              Certifications Current rule --- must be
                                                \>= today
  --------------------------- ----------------- ------------------------------------------

3.5 Reference Data: Contract Master (System of Record)

**CRITICAL:** Contract Master in the system may have more current data
than attached documents (e.g., updated expiry date from extension,
current utilized amount from POs/invoices). Always validate
ceiling/utilization against Contract Master, not just the attached
document.

  -------------------------- --------- ------------------------------------------
  **Field**                  **Data    **Notes**
                             Type**    

  contract_id                String    Primary key for lookup --- use when
                                       quote/SOW references a contract ID

  effective_date             Date      System of record --- may differ from
                                       attached doc if amended

  expiration_date            Date      System of record --- may have been
                                       extended since doc was created

  status                     Enum      Current status (Active/Expired/Terminated)
                                       --- always use system status over doc

  contract_value_utilized    Decimal   Real-time spend against contract ---
                                       updated from PO/Invoice systems

  contract_value_remaining   Decimal   Calculated: value - utilized. USE THIS for
                                       ceiling checks, not doc value.
  -------------------------- --------- ------------------------------------------

3.6 PR Header & Line Items (System Data)

These fields come from the source system (Coupa, SAP Ariba, Oracle) ---
NOT extracted from documents.

  --------------------------------- ------------ ------------------------------------------
  **Field**                         **Data       **Used In**
                                    Type**       

  pr_id                             String       Identification, logging, tracking

  pr_issue_date                     Date         ALL date comparisons --- quote date,
                                                 contract effective/expiry, staleness

  pr_need_by_date                   Date         Covers Delivery Period check

  pr_total_value                    Decimal      Quote Total Match, Ceiling checks,
                                                 Threshold determination, Min/Max PO

  pr_currency                       String       Quote Currency Match

  pr_suggested_supplier             String       ALL Supplier Match rules

  pr_company_code                   String       Entity Coverage, Supplier Approved for
                                                 Entity, Quote Addressee

  pr_category_l1                    String       Category Coverage, Services vs Goods
                                                 determination

  pr_ship_to_region                 String       Geographic Coverage

  pr_justification                  String       Single Source Justification check

  pr_contract_reference             String       Contract lookup when referenced but not
                                    (nullable)   attached

  pr_line_items\[\].description     String       Scope Semantic Match, Item Description
                                                 Match (Check 4)

  pr_line_items\[\].material_code   String       SKU/Item Coverage check
  --------------------------------- ------------ ------------------------------------------

Summary: Check 1 vs Check 2

  --------------- --------------------------- ---------------------------
  **Aspect**      **Check 1: Existence &      **Check 2: Validity &
                  Classification**            Applicability**

  **Purpose**     Does the PR have the RIGHT  Are those documents VALID
                  TYPE of documents attached  (dates, status) and
                  per policy?                 APPLICABLE (supplier,
                                              entity, category)?

  **Extraction    MINIMAL --- only            FULL --- specific field
  Depth**         classification markers      values for validation rules
                  (keywords, structure)       

  **LLM Usage**   Document classification     Field extraction + semantic
                  only                        matching + fuzzy matching

  **Reference     Threshold Config only       Supplier Master, Contract
  Data Needed**                               Master, Threshold Config

  **Pass/Fail     Policy requirements met     All validity rules pass
  Criteria**      (quote count, doc types)    (dates, coverage, supplier
                                              match)

  **Failure       PR CANNOT PROCEED ---       PR FLAGGED --- document
  Impact**        missing required            issues need resolution
                  documentation               before PO

  **Depends On**  Nothing --- this is first   Check 1 --- needs
                  check                       classified documents as
                                              input
  --------------- --------------------------- ---------------------------
