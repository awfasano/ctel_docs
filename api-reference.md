# CTel Ingest - API & Connector Query Reference

Complete reference for every API call, search term, endpoint, and query parameter used by each of the 20 ingestion connectors.

---

## Table of Contents

1. [Federal Register](#1-federal-register)
2. [eCFR](#2-ecfr)
3. [Regulations.gov](#3-regulationsgov)
4. [Congress.gov](#4-congressgov)
5. [Open States](#5-open-states)
6. [LegiScan](#6-legiscan)
7. [CMS Telehealth (Web Scraper)](#7-cms-telehealth)
8. [CMS Telehealth List](#8-cms-telehealth-list)
9. [HCPCS Codes](#9-hcpcs-codes)
10. [Payer Policies](#10-payer-policies)
11. [NPPES NPI Registry](#11-nppes-npi-registry)
12. [Licensure Compacts](#12-licensure-compacts)
13. [OIG LEIE](#13-oig-leie)
14. [SAM.gov](#14-samgov)
15. [CCHP](#15-cchp)
16. [NCSL](#16-ncsl)
17. [PubMed](#17-pubmed)
18. [QPP Measures](#18-qpp-measures)
19. [CDC Telehealth](#19-cdc-telehealth)
20. [Telehealth News](#20-telehealth-news)
21. [Authentication Summary](#authentication-summary)
22. [Rate Limits Summary](#rate-limits-summary)

---

## 1. Federal Register

**Connector:** `federal_register.py` | **Pipeline:** `federal_regulatory` | **Auth:** None

### Endpoint

```
GET https://www.federalregister.gov/api/v1/documents.json
```

### Search Terms (iterated one at a time)

```
telehealth
telemedicine
remote patient monitoring
digital health
virtual care
audio-only telehealth
telecommunications health
```

### Query Parameters

```python
{
    "conditions[term]": "<search_term>",
    "conditions[type][]": ["RULE", "PRORULE", "NOTICE", "PRESDOCU"],
    "fields[]": [
        "document_number", "title", "type", "abstract", "agencies",
        "publication_date", "html_url", "body_html_url", "full_text_xml_url",
        "citation", "action", "dates", "effective_on", "comments_close_on",
        "regulation_id_numbers", "cfr_references", "page_length",
    ],
    "per_page": 1000,
    "order": "newest",
    "page": 1,  # increments
}

# If since date provided:
"conditions[publication_date][gte]": "MM/DD/YYYY"
```

### Pagination

Page-based. Starts at `page=1`, increments by 1. Stops when results are empty or `next_page_url` is null.

### Strategy

For each of the 7 search terms, pages through all matching documents. Deduplicates across terms by `document_number`. For each result, fetches the full HTML body from the `body_html_url` field. If that fails, builds fallback HTML from the abstract and metadata. Targets 4 document types: final rules, proposed rules, notices, and presidential documents.

### Agencies Tracked (for metadata, not used as API filter)

```
health-and-human-services-department
centers-for-medicare-medicaid-services
food-and-drug-administration
drug-enforcement-administration
federal-trade-commission
federal-communications-commission
office-for-civil-rights
office-of-the-national-coordinator-for-health-information-technology
substance-abuse-and-mental-health-services-administration
```

---

## 2. eCFR

**Connector:** `ecfr.py` | **Pipeline:** `federal_regulatory` | **Auth:** None

### Endpoint

```
GET https://www.ecfr.gov/api/renderer/v1/content/enhanced/current/title-{title}?part={part}
```

### No Search Terms

This connector does not search. It fetches a hard-coded list of 19 specific CFR title/part combinations:

### CFR Parts Fetched

**Title 42 (Public Health) - 8 parts:**

| Part | Description |
|------|-------------|
| 410 | Medicare supplementary medical insurance - telehealth coverage |
| 414 | Medicare payment policies - physician fee schedule |
| 425 | Medicare Shared Savings Program - ACO telehealth |
| 438 | Managed care - Medicaid telehealth |
| 440 | Medicaid services - telemedicine |
| 482 | Hospital conditions of participation - telemedicine |
| 485 | Provider conditions - CAHs, RHCs, FQHCs telehealth |
| 486 | Conditions for coverage of specialized services |

**Title 21 (Food and Drugs / DEA) - 5 parts:**

| Part | Description |
|------|-------------|
| 800 | General - medical devices including telehealth devices |
| 820 | Quality System Regulation - medical device manufacturing |
| 1300 | Controlled substances - DEA telemedicine prescribing |
| 1301 | DEA registration - telemedicine special registration |
| 1306 | Prescriptions - electronic prescribing of controlled substances |

**Title 45 (HHS / HIPAA / ONC) - 4 parts:**

| Part | Description |
|------|-------------|
| 160 | HIPAA general administrative requirements |
| 164 | HIPAA security and privacy - telehealth PHI protections |
| 170 | Health IT certification - ONC interoperability |
| 171 | Information blocking - health data access |

**Title 47 (Telecommunications / FCC) - 2 parts:**

| Part | Description |
|------|-------------|
| 54 | Universal Service - Rural Health Care Program |
| 64 | CPNI privacy - telecommunications customer data |

### Pagination

None. Each request returns the entire CFR part as HTML.

### Strategy

Makes 19 individual GET requests, one per CFR part. Always fetches the current version (no date filtering). Wraps each response with metadata headers (title, part, fetch date).

---

## 3. Regulations.gov

**Connector:** `regulations_gov.py` | **Pipeline:** `federal_regulatory` | **Auth:** API key (api.data.gov)

### Endpoint

```
GET https://api.regulations.gov/v4/documents
```

### Search Terms

```
telehealth
telemedicine
remote patient monitoring
digital health
virtual care
```

### Query Parameters

```python
{
    "api_key": "<REGULATIONS_GOV_API_KEY>",
    "filter[searchTerm]": "<search_term>",
    "page[size]": 250,
    "page[number]": 1,  # increments
    "sort": "-postedDate",
}

# If since date provided:
"filter[lastModifiedDate][ge]": "YYYY-MM-DD HH:MM:SS"
```

### Agencies Defined (not used as API filter)

```
HHS, CMS, FDA, DEA, FTC, FCC, ONC
```

These are defined in code but never passed to the API. Filtering is by search term only.

### Pagination

Page-based. Starts at `page[number]=1`. Stops when data is empty or `page >= totalPages`.

### Strategy

For each of the 5 search terms, pages through all matching docket documents sorted by newest posted date. Deduplicates across terms by document ID. Builds HTML content from the document attributes (title, summary, type, agency, dates) -- does not download full document text. Rate limited to 900 req/hr (under the 1,000/hr API limit).

---

## 4. Congress.gov

**Connector:** `congress_gov.py` | **Pipeline:** `state_legislative` | **Auth:** API key (api.data.gov)

### Endpoints

```
GET https://api.congress.gov/v3/bill/{congress}                              # Search
GET https://api.congress.gov/v3/bill/{congress}/{bill_type}/{number}         # Bill detail
GET https://api.congress.gov/v3/bill/{congress}/{bill_type}/{number}/text    # Bill text
```

### Search Terms

```
telehealth
telemedicine
remote patient monitoring
digital health
virtual care
CONNECT for Health
Telehealth Modernization
Ryan Haight
controlled substance telehealth
```

### Congress Sessions

```
119 (current)
118 (previous)
```

### Query Parameters (search)

```python
{
    "api_key": "<CONGRESS_GOV_API_KEY>",
    "query": "<search_term>",
    "offset": 0,  # increments by result count
    "limit": 250,
    "format": "json",
}

# If since date provided:
"fromDateTime": "YYYY-MM-DDT00:00:00Z"
```

### Query Parameters (detail and text)

```python
{"api_key": "<CONGRESS_GOV_API_KEY>", "format": "json"}
```

### Pagination

Offset-based. Starts at `offset=0`, increments by `len(bills)`. Stops when results are empty or `offset >= pagination.count`.

### Strategy

For each of the 2 congress sessions and 9 search terms (18 combinations), searches for bills. For each unique bill found, makes up to 2 additional API calls: one for bill detail (sponsors, committees, subjects, summaries) and one for the latest text version. Prefers "Formatted Text (HTML)" format for bill text, fetches the full HTML from the URL. Deduplicates by `{congress}-{bill_type}-{number}`. Rate limited to 4,500 req/hr.

---

## 5. Open States

**Connector:** `open_states.py` | **Pipeline:** `state_legislative` | **Auth:** API key (X-API-KEY header)

### Endpoints

```
GET https://v3.openstates.org/bills               # Search
GET https://v3.openstates.org/bills/{bill_id}      # Bill detail
GET <version_link_url>                              # Bill text (external URL)
```

### Search Terms

```
telehealth
telemedicine
remote patient monitoring
virtual care
audio-only telehealth
telehealth parity
interstate medical licensure
```

### Query Parameters (search)

```python
{
    "q": "<search_term>",
    "per_page": 20,
    "sort": "updated_desc",
    "include": "sponsorships",
    "page": 1,  # 1 through 5 max
}

# If since date provided:
"updated_since": "YYYY-MM-DD"
```

### Query Parameters (detail)

```python
{
    "include": ["versions", "documents", "actions", "abstracts"]
}
```

### Pagination

Page-based. Pages 1 through 5 maximum (`_MAX_PAGES = 5`). Stops early if `page >= max_page` or results are empty. Max 100 results per search term (20/page x 5 pages).

### Strategy

Searches all 50 states + DC + Puerto Rico simultaneously (no state filter -- the API searches all jurisdictions). For each of the 7 search terms, pages through up to 5 pages. For each bill, fetches detail with versions/documents/actions. Then fetches the actual bill text HTML from the latest version's links (prefers `text/html` media type). Deduplicates by bill ID. Rate limited to 500 req/hr.

---

## 6. LegiScan

**Connector:** `legiscan.py` | **Pipeline:** `state_legislative` | **Auth:** API key

### Endpoint

```
GET https://api.legiscan.com/
```

All operations use the same base URL with different `op` parameter values.

### Search Terms

```
telehealth
telemedicine
remote patient monitoring
virtual care
telehealth parity
interstate medical licensure compact
```

### Query Parameters (search)

```python
{
    "key": "<LEGISCAN_API_KEY>",
    "op": "getSearch",
    "state": "ALL",
    "query": "<search_term>",
    "year": 2,     # current sessions
    "page": 1,     # 1 through 4 max
}
```

### Query Parameters (bill detail)

```python
{
    "key": "<LEGISCAN_API_KEY>",
    "op": "getBill",
    "id": "<bill_id>",
}
```

### Query Parameters (bill text)

```python
{
    "key": "<LEGISCAN_API_KEY>",
    "op": "getBillText",
    "id": "<doc_id>",  # from texts[-1].doc_id
}
```

### Pagination

Page-based. Pages 1 through 4 maximum. Stops when `status != "OK"` or `page >= page_total`.

### Strategy

Searches all states (`state=ALL`) for current sessions (`year=2`). For each of the 6 search terms, pages through up to 4 pages. The search response uses numbered keys (0, 1, 2...) instead of a list -- iterates dict values and filters for entries containing `bill_id`. For each unique bill, fetches detail and then the latest text version. **Bill text is base64-encoded** and must be decoded via `base64.b64decode()`. Deduplicates by `bill_id` (integer). Rate limited to 1,500 req/hr. Free tier: 30,000 queries/month.

---

## 7. CMS Telehealth

**Connector:** `cms_telehealth.py` | **Pipeline:** `billing_coverage` | **Auth:** None

### No API -- Web Scraping Only

### Pages Scraped (13 fixed URLs)

```
https://telehealth.hhs.gov/providers/billing-and-reimbursement
https://telehealth.hhs.gov/providers/billing-and-reimbursement/billing-and-coding-medicare-fee-for-service-claims
https://telehealth.hhs.gov/providers/billing-and-reimbursement/medicare-payment-policies
https://telehealth.hhs.gov/providers/billing-and-reimbursement/medicaid-and-chip-telehealth-coverage
https://telehealth.hhs.gov/providers/billing-and-reimbursement/commercial-insurance
https://telehealth.hhs.gov/providers/telehealth-policy
https://telehealth.hhs.gov/providers/telehealth-policy/telehealth-policy-updates
https://telehealth.hhs.gov/providers/best-practice-guides
https://telehealth.hhs.gov/providers/preparing-patients-for-telehealth
https://www.medicare.gov/coverage/telehealth
https://www.cms.gov/medicare/payment/fee-schedules/physician
https://telehealth.hhs.gov/providers
https://telehealth.hhs.gov/patients
```

### PDF Discovery

After scraping the pages, scans these 2 URLs for PDF links:
```
https://telehealth.hhs.gov/providers/billing-and-reimbursement
https://telehealth.hhs.gov/providers/best-practice-guides
```

Regex: `href="(https?://[^"]+\.pdf)"` -- downloads up to **15 PDFs**.

### Strategy

Simple fetch-and-store. Gets the HTML of each page, extracts the `<body>` content, strips the `<title>` suffix (` | Telehealth.HHS.gov`, ` | CMS`, ` | Medicare.gov`). Then discovers and downloads linked PDFs. Rate limited to 300 req/hr.

---

## 8. CMS Telehealth List

**Connector:** `cms_telehealth_list.py` | **Pipeline:** `billing_coverage` | **Auth:** None

### Endpoints

**Services List Page (for ZIP URL discovery):**
```
https://www.cms.gov/medicare/coverage/telehealth/list-services
```

**ZIP Downloads (fallback if not discovered):**
```
https://www.cms.gov/files/zip/list-telehealth-services-calendar-year-2026.zip
https://www.cms.gov/files/zip/list-telehealth-services-calendar-year-2025.zip
```

**Medicare Telehealth Trends API:**
```
GET https://data.cms.gov/data-api/v1/dataset/939226be-b107-476e-8777-f199a840138a/data
    ?size=5000
```

**Medicaid Toolkit PDFs:**
```
https://www.medicaid.gov/medicaid/benefits/downloads/telehealth-toolkt.pdf
https://www.medicaid.gov/medicaid/benefits/downloads/medicaid-chip-telehealth-toolkit.pdf
```

### Strategy

4-step process:
1. Scrapes the CMS list page and discovers ZIP download URLs via regex: `href="(https?://[^"]*list-telehealth-services[^"]*\.zip)"`
2. Downloads the ZIP and extracts all files inside (XLSX spreadsheets, CSV, PDF)
3. Fetches the trends API data (single request, `size=5000`)
4. Downloads 2 Medicaid toolkit PDFs

Rate limited to 120 req/hr with 60s timeout.

---

## 9. HCPCS Codes

**Connector:** `hcpcs.py` | **Pipeline:** `billing_coverage` | **Auth:** None

### Endpoint

```
GET https://clinicaltables.nlm.nih.gov/api/hcpcs/v3/search
```

### Search Terms

```
telehealth
telemedicine
virtual
remote monitoring
remote patient
audio only
synchronous
telephone evaluation
online digital
e-visit
```

### Known Code Prefixes (also searched individually)

```
G2010  G2012  G2014  G2250  G2251  G2252
G0425  G0426  G0427
98966  98967  98968  98970  98971  98972
99441  99442  99443
98000  98001  98002  98003  98004  98005  98006  98007
98008  98009  98010  98011  98012  98013  98014  98015  98016
```

### Query Parameters

```python
{
    "terms": "<search_term_or_code>",
    "count": 500,
    "sf": "code,short_desc,long_desc",
    "ef": "short_desc,long_desc,effective_date,add_date,term_date",
}
```

### Pagination

None. Single request per term/code with `count=500`.

### Strategy

Makes one request per search term (10) and one per known code prefix (30) = ~40 requests. The API returns an array: `[total_count, codes_list, extra_data_dict, display_strings]`. Deduplicates by code string. Builds both a comprehensive reference document with all codes in a table and individual per-code documents. Rate limited to 600 req/hr.

---

## 10. Payer Policies

**Connector:** `payer_policies.py` | **Pipeline:** `billing_coverage` | **Auth:** None

### No API -- Web Scraping and PDF Downloads

### URLs by Payer

**UnitedHealthcare (3 PDFs):**
```
https://www.uhcprovider.com/content/dam/provider/docs/public/policies/comm-reimbursement/COMM-Telehealth-and-Telemedicine-Policy.pdf
https://www.uhcprovider.com/content/dam/provider/docs/public/policies/comm-reimbursement/COMM-Telehealth-Policy-Facility.pdf
https://www.uhcprovider.com/content/dam/provider/docs/public/policies/medicaid-comm-plan-reimbursement/UHCCP-Telehealth-Virtual-Health-Policy-Professional-and-Facility-R7133.pdf
```

**Aetna (1 PDF + 1 HTML):**
```
https://www.aetna.com/content/dam/aetna/pdfs/aetnacom/pdf/telemedicine.pdf
https://www.aetna.com/medicare/for-members/telehealth.html
```

**Cigna (1 HTML + 1 PDF):**
```
https://static.cigna.com/assets/chcp/resourceLibrary/medicalResourcesList/medicalDoingBusinessWithCigna/medicalDbwCVirtualCare.html
https://static.cigna.com/assets/chcp/pdf/coveragePolicies/medical/mm_0563_coveragepositioncriteria_remote_patient_monitoring_and_remote_therapeutic_monitoring.pdf
```

**Humana (4 HTML pages):**
```
https://www.humana.com/provider/telehealth-billing
https://www.humana.com/provider/telehealth-payment
https://www.humana.com/provider/telehealth-faq
https://www.humana.com/provider/news/medical-news/telehealth-services-policy
```

**Anthem (1 HTML page):**
```
https://www.anthem.com/provider/individual-commercial/policies
```

### PDF Discovery

After fetching HTML pages, scans each for linked PDFs:
- Absolute: `href="(https?://[^"]+\.pdf)"`
- Relative: `href="(/[^"]+\.pdf)"` (resolved against page URL)
- Same-domain only. Up to 10 PDFs per HTML page.

### Strategy

Fetches all 12 fixed URLs. For HTML pages, extracts body content and scans for additional PDF links. Also builds a payer overview document listing all sources. Rate limited to 120 req/hr with 60s timeout.

---

## 11. NPPES NPI Registry

**Connector:** `nppes.py` | **Pipeline:** `provider_licensing` | **Auth:** None

### Endpoint

```
GET https://npiregistry.cms.hhs.gov/api/
```

### Query Parameters

```python
{
    "version": "2.1",
    "enumeration_type": "NPI-1",             # Individual providers only
    "taxonomy_description": "<specialty>",
    "limit": 1,                              # Only need the count
    "skip": 0,
}

# For state-level queries:
"state": "<2-letter state code>"
```

### Specialties Queried (20 total)

| Search Term | Taxonomy Code | Display Name |
|-------------|---------------|--------------|
| Family Medicine | 207Q00000X | Family Medicine |
| Internal Medicine | 207R00000X | Internal Medicine |
| Psychiatry | 2084P0800X | Psychiatry |
| Nurse Practitioner | 363L00000X | Nurse Practitioner |
| Clinical Psychologist | 103T00000X | Clinical Psychologist |
| Licensed Clinical Social Worker | 1041C0700X | Licensed Clinical Social Worker |
| Professional Counselor | 101YM0800X | Licensed Professional Counselor |
| Speech-Language Pathologist | 235Z00000X | Speech-Language Pathologist |
| Occupational Therapist | 225X00000X | Occupational Therapist |
| Physical Therapist | 225100000X | Physical Therapist |
| Registered Dietitian | 133V00000X | Registered Dietitian |
| Physician Assistant | 363A00000X | Physician Assistant |
| Dermatology | 207N00000X | Dermatology |
| Cardiology | 207RC0000X | Cardiology |
| Endocrinology | 207RE0101X | Endocrinology |
| Neurology | 2084N0400X | Neurology |
| Addiction Medicine | 207LA0401X | Addiction Medicine |
| Pulmonary Disease | 207RP1001X | Pulmonary Disease |
| Pediatrics | 208000000X | Pediatrics |
| Geriatric Medicine | 207QG0300X | Geriatric Medicine |

### States Queried (20 total)

```
CA, TX, FL, NY, PA, IL, OH, GA, NC, MI,
NJ, VA, WA, AZ, MA, TN, IN, MO, MD, CO
```

### Strategy

Does **not** download actual provider records -- only counts. Makes one national request per specialty (20 calls) and one per specialty-state combination (20 x 20 = 400 calls) for ~420 total API calls. Extracts only `result_count` from each response. Builds a taxonomy reference document, a national summary, and per-state summary documents. Rate limited to 1,200 req/hr.

---

## 12. Licensure Compacts

**Connector:** `licensure_compacts.py` | **Pipeline:** `provider_licensing` | **Auth:** None

### Primarily Hard-Coded Data (Not API-Driven)

This connector has all compact membership data embedded directly in the code. The web scraping it does is minimal and the scraped content is not used in output documents.

### Compact Websites (scraped but content not used)

```
https://www.imlcc.org/
https://www.nursecompact.com/
https://psypact.org/
https://ptcompact.org/
https://counselingcompact.org/
https://swcompact.org/
```

### Hard-Coded Compact Data

| Compact | Profession | Enacted | Member States |
|---------|-----------|---------|---------------|
| **IMLC** | Physicians | 2017 | 44 states |
| **NLC** | Nurses (RN/LPN) | 2018 | 39 states |
| **PSYPACT** | Psychologists | 2019 | 46 states |
| **PT Compact** | Physical Therapists | 2017 | 39 states |
| **Counseling Compact** | Licensed Counselors | 2022 | 42 states |
| **Social Work Compact** | Social Workers | 2023 | 41 states |

### Strategy

Fetches each compact's homepage (takes first 5,000 characters) but this scraped data is stored in a dict that is never referenced. All output documents are built from the hard-coded `_COMPACTS` data. Produces: overview document, 6 per-compact documents, and a state-by-state guide mapping each state to its compact memberships. Rate limited to 120 req/hr.

---

## 13. OIG LEIE

**Connector:** `oig_leie.py` | **Pipeline:** `compliance` | **Auth:** None

### Endpoint

```
GET https://oig.hhs.gov/exclusions/downloadables/UPDATED.csv
```

### No Query Parameters

Single download of the full active exclusions CSV file.

### CSV Columns

```
LASTNAME, FIRSTNAME, MIDNAME, BUSNAME, GENERAL, SPECIALTY,
UPIN, NPI, DOB, ADDRESS, CITY, STATE, ZIP,
EXCLTYPE, EXCLDATE, REINDATE, WAIVERDATE, WVRSTATE
```

### Exclusion Types Tracked (16 categories)

| Code | Description |
|------|-------------|
| 1128a1 | Conviction of program-related crimes |
| 1128a2 | Conviction relating to patient abuse or neglect |
| 1128a3 | Felony conviction relating to health care fraud |
| 1128a4 | Felony conviction relating to controlled substances |
| 1128b1 | Misdemeanor conviction relating to health care fraud |
| 1128b2 | Misdemeanor conviction relating to controlled substances |
| 1128b4 | License revocation or suspension |
| 1128b5 | Exclusion or suspension under federal or state health care program |
| 1128b6 | Claims for excessive charges or unnecessary services |
| 1128b7 | Fraud, kickbacks, and other prohibited activities |
| 1128b8 | Entities controlled by a sanctioned individual |
| 1128b9 | Failure to disclose required information |
| 1128b10 | Failure to return overpayments |
| 1128b11 | Failure to grant immediate access |
| 1128b12 | Failure to take corrective action |
| 1128b13-16 | Various other fraud/compliance violations |
| 1156 | Failure to meet professionally recognized standards |

### Strategy

Downloads the entire CSV file (single request). Parses with `csv.DictReader`. Builds 3 types of documents:
1. **National summary** -- counts by exclusion type and by state
2. **Per-state summaries** -- one document per state found in the data (only recent exclusions with `EXCLDATE >= 20240101`)
3. **Per-provider documents** -- one document per record with a real NPI (not `0000000000`)

Rate limited to 60 req/hr with 120s timeout.

---

## 14. SAM.gov

**Connector:** `sam_gov.py` | **Pipeline:** `compliance` | **Auth:** API key (sam.gov)

### Endpoint

```
GET https://api.sam.gov/entity-information/v4/exclusions
```

### Query Parameters (by agency)

```python
{
    "api_key": "<SAM_GOV_API_KEY>",
    "size": 100,
    "offset": 0,  # increments
    "excludingAgencyCode": "<agency>",
    "updateDate": "[MM/DD/YYYY,MM/DD/YYYY]",
}
```

### Query Parameters (by keyword)

```python
{
    "api_key": "<SAM_GOV_API_KEY>",
    "size": 100,
    "offset": 0,
    "q": "<keyword>",
    "updateDate": "[MM/DD/YYYY,MM/DD/YYYY]",
}
```

### Agency Codes Queried

```
HHS, CMS, FDA, NIH, CDC, HRSA, SAMHSA, IHS, AHRQ, OIG-HHS
```

### Keywords Queried

```
healthcare, medical, telehealth, medicare
```

### Pagination

Offset-based. Page size 100. Stops when `len(results) >= min(total, 500)` or batch is less than 100. Cap of 500 records per query.

### Date Range

Default: last 90 days if `since` is None. Format: `updateDate=[MM/DD/YYYY,MM/DD/YYYY]`.

### Strategy

Makes 10 agency-based queries + 4 keyword-based queries = 14 base queries, each of which may paginate. Deduplicates by `samNumber` or composite key `{name}_{exclusionDate}`. Builds a summary document (counts by agency and type) and per-record documents. Rate limited to 500 req/hr with 60s timeout.

---

## 15. CCHP

**Connector:** `cchp.py` | **Pipeline:** `telehealth_policy` | **Auth:** None

### No API -- Web Scraping Only

### State Pages (51 URLs -- all 50 states + DC)

```
https://www.cchpca.org/alabama/
https://www.cchpca.org/alaska/
https://www.cchpca.org/arizona/
https://www.cchpca.org/arkansas/
https://www.cchpca.org/california/
https://www.cchpca.org/colorado/
https://www.cchpca.org/connecticut/
https://www.cchpca.org/delaware/
https://www.cchpca.org/district-of-columbia/
https://www.cchpca.org/florida/
https://www.cchpca.org/georgia/
https://www.cchpca.org/hawaii/
https://www.cchpca.org/idaho/
https://www.cchpca.org/illinois/
https://www.cchpca.org/indiana/
https://www.cchpca.org/iowa/
https://www.cchpca.org/kansas/
https://www.cchpca.org/kentucky/
https://www.cchpca.org/louisiana/
https://www.cchpca.org/maine/
https://www.cchpca.org/maryland/
https://www.cchpca.org/massachusetts/
https://www.cchpca.org/michigan/
https://www.cchpca.org/minnesota/
https://www.cchpca.org/mississippi/
https://www.cchpca.org/missouri/
https://www.cchpca.org/montana/
https://www.cchpca.org/nebraska/
https://www.cchpca.org/nevada/
https://www.cchpca.org/new-hampshire/
https://www.cchpca.org/new-jersey/
https://www.cchpca.org/new-mexico/
https://www.cchpca.org/new-york/
https://www.cchpca.org/north-carolina/
https://www.cchpca.org/north-dakota/
https://www.cchpca.org/ohio/
https://www.cchpca.org/oklahoma/
https://www.cchpca.org/oregon/
https://www.cchpca.org/pennsylvania/
https://www.cchpca.org/rhode-island/
https://www.cchpca.org/south-carolina/
https://www.cchpca.org/south-dakota/
https://www.cchpca.org/tennessee/
https://www.cchpca.org/texas/
https://www.cchpca.org/utah/
https://www.cchpca.org/vermont/
https://www.cchpca.org/virginia/
https://www.cchpca.org/washington/
https://www.cchpca.org/west-virginia/
https://www.cchpca.org/wisconsin/
https://www.cchpca.org/wyoming/
```

### Resource Pages (3 URLs)

```
https://www.cchpca.org/resources/
https://www.cchpca.org/policy-trends/
https://www.cchpca.org/all-telehealth-policies/
```

### PDF Discovery

Scans `https://www.cchpca.org/resources/` for PDFs matching:
```
href="(https?://(?:www\.)?cchpca\.org/[^"]+\.pdf)"
```
Downloads up to **20 PDFs**.

### Strategy

Fetches all 51 state pages + 3 resource pages + discovers and downloads up to 20 PDFs. ~75 total requests. Rate limited to 300 req/hr.

---

## 16. NCSL

**Connector:** `ncsl.py` | **Pipeline:** `telehealth_policy` | **Auth:** None

### No API -- Web Scraping Only

### Primary Policy Pages (10 URLs)

```
https://www.ncsl.org/health/state-telehealth-policies
https://www.ncsl.org/health/the-telehealth-explainer-series
https://www.ncsl.org/health/the-telehealth-explainer-series/telehealth-definitions-and-key-concepts
https://www.ncsl.org/health/the-telehealth-explainer-series/medicaid-reimbursement-for-telehealth
https://www.ncsl.org/health/the-telehealth-explainer-series/telehealth-private-insurance-laws
https://www.ncsl.org/health/the-telehealth-explainer-series/licensure-and-interstate-compacts
https://www.ncsl.org/health/the-telehealth-explainer-series/ensuring-patient-safety
https://www.ncsl.org/health/the-telehealth-explainer-series/behavioral-health-and-telehealth
https://www.ncsl.org/health/health-workforce-legislation-database
https://www.ncsl.org/health/state-public-health-legislation-database
```

### Additional Health Pages (4 URLs)

```
https://www.ncsl.org/health/health-information-technology
https://www.ncsl.org/health/health-workforce
https://www.ncsl.org/health/scope-of-practice
https://www.ncsl.org/health/prescription-drug-resource-center
```

### Link Discovery

After scraping primary pages, scans the state telehealth policies page for internal links:
```
href="(https?://(?:www\.)?ncsl\.org/health/[^"]+)"
```
Follows up to **15 additional discovered links** (only `/health/` paths, no anchors or query strings).

### PDF Discovery

Known PDF:
```
https://documents.ncsl.org/wwwncsl/Health/Telehealth-Private-Insurance-Laws_36242.pdf
```

Also scans the first 2 policy pages for PDFs:
```
href="(https?://documents\.ncsl\.org/[^"]+\.pdf)"
```
Downloads up to **10 PDFs**.

### Strategy

Scrapes ~29 HTML pages + downloads up to 10 PDFs. Rate limited to 300 req/hr.

---

## 17. PubMed

**Connector:** `pubmed_telehealth.py` | **Pipeline:** `clinical_evidence` | **Auth:** Optional API key

### Endpoints

```
GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi    # Search for PMIDs
GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi     # Fetch article XML
```

### Search Queries (exact PubMed syntax)

| Key | Title | PubMed Query |
|-----|-------|-------------|
| telehealth_systematic_reviews | Telehealth Systematic Reviews | `(telehealth[MeSH] OR telemedicine[MeSH]) AND systematic review[pt]` |
| telehealth_meta_analyses | Telehealth Meta-Analyses | `(telehealth[MeSH] OR telemedicine[MeSH]) AND meta-analysis[pt]` |
| remote_monitoring_reviews | Remote Patient Monitoring Reviews | `("remote patient monitoring" OR "remote physiologic monitoring") AND (review[pt] OR systematic review[pt])` |
| telepsychiatry_evidence | Telepsychiatry Evidence | `(telepsychiatry OR "telebehavioral health") AND (review[pt] OR clinical trial[pt])` |
| telehealth_cost_effectiveness | Telehealth Cost-Effectiveness | `(telehealth[MeSH] OR telemedicine[MeSH]) AND (cost-effectiveness[tiab] OR "cost analysis"[MeSH])` |

### ESearch Parameters

```python
{
    "db": "pubmed",
    "term": "<pubmed_query>",
    "retmax": 50,
    "retmode": "json",
    "sort": "relevance",
    "mindate": "YYYY/MM/DD",   # default: 2 years ago
    "maxdate": "YYYY/MM/DD",   # default: today
    "datetype": "pdat",        # publication date
}
```

### EFetch Parameters

```python
{
    "db": "pubmed",
    "id": "<comma_separated_pmids>",
    "retmode": "xml",
    "rettype": "abstract",
}
```

### Strategy

Two-step process for each of the 5 queries:
1. **ESearch** returns up to 50 PMIDs per query (max 250 total)
2. **EFetch** retrieves full XML for all PMIDs from each query in one batch call

Parses XML with `xml.etree.ElementTree`. Extracts: PMID, title, abstract (with labeled sections like BACKGROUND, METHODS, RESULTS), authors, journal, date, DOI, MeSH terms, publication types. Builds an overview document + per-article documents. Default date window is 2 years (730 days). Rate limited to 1,000 req/hr. Optional API key increases rate from 3/sec to 10/sec.

---

## 18. QPP Measures

**Connector:** `qpp_measures.py` | **Pipeline:** `clinical_evidence` | **Auth:** None

### Endpoints

```
GET https://raw.githubusercontent.com/CMSgov/qpp-measures-data/master/measures/2025/measures-data.json
GET https://raw.githubusercontent.com/CMSgov/qpp-measures-data/master/measures/2024/measures-data.json
```

### No Query Parameters

Direct download of the full JSON files from GitHub.

### Client-Side Keyword Filtering

After downloading, filters measures by checking if any of these keywords appear in title, description, measureSpecification, clinicalGuidelineChanged, category, subcategoryId, or objective:

```
telehealth
telemedicine
virtual
remote monitoring
remote patient
telephone
audio
video
e-visit
digital
online
```

### Strategy

Makes exactly 2 HTTP requests (one per year: 2025, 2024). Downloads the full measures JSON, filters for telehealth-relevant measures client-side. Builds: overview document, per-year summaries, per-measure documents. Rate limited to 60 req/hr.

---

## 19. CDC Telehealth

**Connector:** `cdc_telehealth.py` | **Pipeline:** `utilization_data` | **Auth:** None

### SODA API Endpoints

| Key | Dataset | URL |
|-----|---------|-----|
| telemedicine_use | Telemedicine Use in the Last 4 Weeks | `https://data.cdc.gov/resource/h7xa-837u.json` |
| mental_health_telehealth | Mental Health Telehealth Use | `https://data.cdc.gov/resource/yni7-er2q.json` |
| rands_telemedicine | RANDS Telemedicine During COVID-19 | `https://data.cdc.gov/resource/8xy9-ubqz.json` |

### Query Parameters

```python
{"$limit": 5000, "$order": ":id"}
```

Falls back to no parameters if the first request fails.

### Web Scraping

```
https://www.cdc.gov/nchs/covid19/pulse/telemedicine-use.htm
```

### Strategy

Makes 3 SODA API requests (one per dataset, `$limit=5000` each) + 1 CDC page scrape. Dynamically detects state columns by checking for column names: `state`, `State`, `geography`, `Geography`, `subgroup`. Builds: overview document, 3 per-dataset summaries, per-state documents (for states with 2+ data points), CDC page document. Rate limited to 500 req/hr.

---

## 20. Telehealth News

**Connector:** `telehealth_news.py` | **Pipeline:** N/A | **Auth:** None

### RSS Feed URLs

| Source | URL |
|--------|-----|
| Google News (telehealth) | `https://news.google.com/rss/search?q=telehealth+when:7d&hl=en-US&gl=US&ceid=US:en` |
| Google News (telemedicine) | `https://news.google.com/rss/search?q=telemedicine+policy+when:7d&hl=en-US&gl=US&ceid=US:en` |
| Google News (virtual care) | `https://news.google.com/rss/search?q=%22virtual+care%22+healthcare+when:7d&hl=en-US&gl=US&ceid=US:en` |
| mHealth Intelligence | `https://mhealthintelligence.com/rss/feed` |
| Healthcare IT News | `https://www.healthcareitnews.com/feed` |
| Fierce Healthcare | `https://www.fiercehealthcare.com/rss/xml` |

### Google News Embedded Search Queries

```
telehealth when:7d
telemedicine policy when:7d
"virtual care" healthcare when:7d
```

### Strategy

Fetches 6 RSS feeds. Parses XML to extract `<item>` elements (title, link, pubDate, description). Filters to last 7 days by default (or since `since` date). For each RSS item, **fetches the full article** at the link URL -- falls back to RSS description if the fetch fails. Deduplicates by URL (using SHA-256 hash of URL as unique key). Rate limited to 600 req/hr.

---

## Authentication Summary

| Connector | Auth | Key Variable | Delivery Method |
|-----------|------|-------------|-----------------|
| federal_register | None | -- | -- |
| ecfr | None | -- | -- |
| regulations_gov | API key | `REGULATIONS_GOV_API_KEY` | Query param `api_key` |
| congress_gov | API key | `CONGRESS_GOV_API_KEY` | Query param `api_key` |
| open_states | API key | `OPEN_STATES_API_KEY` | Header `X-API-KEY` |
| legiscan | API key | `LEGISCAN_API_KEY` | Query param `key` |
| cms_telehealth | None | -- | -- |
| cms_telehealth_list | None | -- | -- |
| hcpcs | None | -- | -- |
| payer_policies | None | -- | -- |
| nppes | None | -- | -- |
| licensure_compacts | None | -- | -- |
| oig_leie | None | -- | -- |
| sam_gov | API key | `SAM_GOV_API_KEY` | Query param `api_key` |
| cchp | None | -- | -- |
| ncsl | None | -- | -- |
| pubmed_telehealth | Optional | Constructor param | Query param `api_key` |
| qpp_measures | None | -- | -- |
| cdc_telehealth | None | -- | -- |
| telehealth_news | None | -- | -- |

**5 required keys + 1 optional = 6 total API keys across all connectors.**

---

## Rate Limits Summary

| Connector | Rate Limit | Timeout | Approx. Requests Per Run |
|-----------|-----------|---------|-------------------------|
| federal_register | Unrestricted | 30s | ~7-70 (7 terms, paginated) |
| ecfr | Unrestricted | 30s | 19 (fixed) |
| regulations_gov | 900/hr | 30s | ~5-50 (5 terms, paginated) |
| congress_gov | 4,500/hr | 30s | ~18-500+ (18 search combos + detail + text) |
| open_states | 500/hr | 30s | ~7-100+ (7 terms x 5 pages + detail + text) |
| legiscan | 1,500/hr | 30s | ~6-100+ (6 terms x 4 pages + detail + text) |
| cms_telehealth | 300/hr | 30s | ~28 (13 pages + up to 15 PDFs) |
| cms_telehealth_list | 120/hr | 60s | ~5-10 (page + ZIP + API + PDFs) |
| hcpcs | 600/hr | 30s | ~40 (10 terms + 30 codes) |
| payer_policies | 120/hr | 60s | ~12-50 (12 URLs + discovered PDFs) |
| nppes | 1,200/hr | 30s | ~420 (20 specialties x 21 queries) |
| licensure_compacts | 120/hr | 30s | 6 (compact homepages) |
| oig_leie | 60/hr | 120s | 1 (single CSV download) |
| sam_gov | 500/hr | 60s | ~14-70 (14 queries, paginated) |
| cchp | 300/hr | 30s | ~75 (51 states + 3 resource + ~20 PDFs) |
| ncsl | 300/hr | 30s | ~40 (14 pages + 15 discovered + 10 PDFs) |
| pubmed_telehealth | 1,000/hr | 30s | 10 (5 ESearch + 5 EFetch) |
| qpp_measures | 60/hr | 30s | 2 (fixed) |
| cdc_telehealth | 500/hr | 30s | 4 (3 SODA + 1 scrape) |
| telehealth_news | 600/hr | 30s | ~6-100+ (6 feeds + article fetches) |
