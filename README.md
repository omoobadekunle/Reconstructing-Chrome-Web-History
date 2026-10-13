# Reconstructing Chrome Web History

**Digital Forensics Investigation Report**  
**Course:** SBT-DF204 — Computer Forensics Case Studies  
**Case Study:** 2 — Reconstructing Chrome Web History  
**Platform:** Kali Linux virtual machine  
**Assessment:** Individual browser history investigation  
**Evidence database:** `Chrome_History_Cryptocurrency_Lab.db`  
**Report date:** 24 September 2026  
**Author:** Adekunle Ogunyemi

> **Scope note:** This README documents findings reported from the supplied Chrome History SQLite database and its accompanying investigation report. It does not claim that missing screenshots, the downloaded PNG, email contents, or independent transaction records have been examined.

## Table of Contents

- [Project Overview](#project-overview)
- [Objectives](#objectives)
- [Scope of Examination](#scope-of-examination)
- [Tools and Technologies](#tools-and-technologies)
- [Lab Environment and Evidence Handling](#lab-environment-and-evidence-handling)
  - [Evidence files and recorded integrity information](#evidence-files-and-recorded-integrity-information)
  - [Integrity verification still required](#integrity-verification-still-required)
- [Methodology](#methodology)
  - [Database discovery and schema inspection](#database-discovery-and-schema-inspection)
  - [Timeline reconstruction](#timeline-reconstruction)
  - [Download analysis](#download-analysis)
  - [Navigation relationships](#navigation-relationships)
- [Practical Commands and Queries](#practical-commands-and-queries)
  - [List the evidence and working copy](#list-the-evidence-and-working-copy)
  - [Verify SHA-256 hashes](#verify-sha-256-hashes)
  - [Inspect the SQLite database](#inspect-the-sqlite-database)
  - [Reconstruct the visit timeline](#reconstruct-the-visit-timeline)
  - [Decode basic transition types](#decode-basic-transition-types)
  - [Inspect download records and URL chains](#inspect-download-records-and-url-chains)
- [Findings and Results](#findings-and-results)
  - [Evidence acquisition](#evidence-acquisition)
  - [Database tables and fields](#database-tables-and-fields)
  - [Timeline of relevant activity](#timeline-of-relevant-activity)
  - [Craigslist posting workflow](#craigslist-posting-workflow)
  - [Email and cryptocurrency-related browsing](#email-and-cryptocurrency-related-browsing)
  - [Recorded PNG download](#recorded-png-download)
- [Analysis and Discussion](#analysis-and-discussion)
- [Attribution Assessment and Limitations](#attribution-assessment-and-limitations)
- [Evidence Log](#evidence-log)
- [Conclusion](#conclusion)
- [Recommendations and Outstanding Work](#recommendations-and-outstanding-work)
- [Screenshot Checklist](#screenshot-checklist)
- [Suggested Repository Structure](#suggested-repository-structure)
- [References](#references)
- [Author Information](#author-information)

## Project Overview

This investigation reviewed a supplied Google Chrome History SQLite database to reconstruct a time-ordered account of relevant browser activity. The recorded activity on **19 April 2022** includes a Craigslist posting workflow, Gmail pages with an order-related subject, an Imgur image page, blockchain-explorer and Bitcoin-address pages, Kraken pages, and a download record for a PNG file named `proof_of_payment.png`.

The findings are based on browser-history records and the SQL output described in the investigation report. The sequence is consistent with posting an offer and later reviewing transaction-related material, but browser history alone does not establish who operated the browser, what an email or image contained, whether a payment was completed, or whether a crime occurred.

**Overall assessment recorded in the source report:**
- **Moderate confidence** that the browser-history sequence is consistent with the case hypothesis.
- **Low confidence** in attribution to a specific person or proof of an offence from this evidence alone.

## Objectives

- Examine the supplied Chrome History SQLite database.
- Identify relevant tables and fields for URLs, visits, downloads, and download URL chains.
- Convert Chrome WebKit timestamps into UTC date and time.
- Reconstruct a chronological sequence of relevant visits.
- Review navigation relationships using `from_visit` and raw `transition` values.
- Correlate the recorded download with its URL chain.
- Document findings, limitations, and further verification needed.

## Scope of Examination

The examination covered:

- The `urls` and `visits` tables, joined through `visits.url = urls.id`.
- The `downloads` and `downloads_url_chains` tables.
- Chrome WebKit timestamp conversion.
- Page titles, URLs, visit relationships, raw navigation-transition values, and download metadata.

No live websites, email accounts, cryptocurrency wallets, exchanges, or transaction services were visited as part of the reported analysis. The actual email body, listing content beyond the recorded title/URL information, and downloaded PNG image were not supplied as examined artefacts in the findings.

## Tools and Technologies

| Tool or technology | Use recorded in the report |
|---|---|
| Kali Linux virtual machine | Investigation environment |
| `sqlite3` 3.53.4 | Open and query the Chrome History database; inspect tables and schema |
| `sha256sum` | Calculate SHA-256 hashes for evidence integrity checks |
| `ls -lh` | Display evidence and working-copy file sizes |
| SQL | Join URL and visit records, convert timestamps, and inspect download records |

## Lab Environment and Evidence Handling

The report shows an evidence file and a separate working copy in the case-study folder on Kali Linux.

| Item | Recorded value |
|---|---|
| Original evidence filename | `Chrome_History_Cryptocurrency_Lab.db` |
| Evidence path shown | `evidence/Chrome_History_Cryptocurrency_Lab.db` |
| Working-copy path shown | `working/working_Chrome_History_Cryptocurrency_Lab.db` |
| Original file size | 192 KB as displayed by `ls -lh`; exact byte count not recorded |
| Working-copy size | 192 KB as displayed by `ls -lh` |
| Original SHA-256 shown | `b991b0fa69662c44caf599e468c8088576ebd9cebbf46d02c89cf51bcde3fd19` |
| SQLite version | 3.53.4 |

### Evidence files and recorded integrity information

The source report identifies the original database under `evidence/` and the working copy under `working/`. The reported workflow examined the database with `sqlite3`, rather than opening it in a browser.

The original SHA-256 value is visible in the supplied report. However, the report states that the attempted hash-comparison command contained a typing error. **Hash equality between the original and working copy has therefore not been verified by the supplied findings.**

### Integrity verification still required

From the case-study directory, rerun the hash command against both files:

```bash
sha256sum evidence/Chrome_History_Cryptocurrency_Lab.db working/working_Chrome_History_Cryptocurrency_Lab.db
```

Compare the complete hash strings. Matching values support that the two files are byte-for-byte identical at the time of hashing. Save the terminal output as evidence and record the date/time and operator. Do not state that hash equality was verified until the corrected command has been run and its output checked.

**Screenshot placeholder — hash verification:**  
Save a screenshot of the corrected command and both hash outputs as `screenshots/hash-comparison-verified.png`. This screenshot was not supplied as a separate image in the source material.

## Methodology

### Database discovery and schema inspection

The database was opened using `sqlite3`. The table list and table structures were inspected before examining the relevant records. The source report records tables including `urls`, `visits`, `downloads`, `downloads_url_chains`, `keyword_search_terms`, `visit_source`, and other Chrome metadata tables.

The main visit analysis joined URL metadata to individual visit records using:

```sql
JOIN urls AS u ON v.url = u.id
```

This allowed URL IDs and page titles to be correlated with individual visit IDs and timestamps.

### Timeline reconstruction

Chrome stores `visit_time` as microseconds since **1 January 1601 UTC**. The report's SQL converts that value to Unix time and formats it as a UTC date/time. The timeline reported below uses the times shown in the supplied SQL output. No separate conversion to local time was documented, so the times remain in UTC.

### Download analysis

Download metadata was reviewed separately in `downloads`, then compared with `downloads_url_chains`. The source report records a 33,844-byte PNG download whose URL chain contains `https://i.imgur.com/dTgrkP7.png`.

The recorded filename is `proof_of_payment.png`. The name suggests a payment-related purpose, but the image itself was not provided for examination. The filename must not be treated as proof of the image's contents.

### Navigation relationships

The `from_visit` field was used to examine links between visit records. The `transition` values in the output include large values and values displayed as negative numbers. These are raw stored values that can include qualifier bits; they should not be interpreted from their decimal representation alone.

The basic transition type can be extracted using `transition & 255`. The supplied report does not show a separate output column containing that calculation, so precise navigation types are not assigned to every visit in this README.

## Practical Commands and Queries

The commands below document the procedures described in the report and provide reproducible ways to complete checks explicitly identified as outstanding. Run them only against evidence you are authorised to examine. Where the source report did not preserve an exact command or output, the query is identified as a reproduction of the reported method rather than a claim that the command was previously executed exactly as written.

### List the evidence and working copy

From the case-study directory:

```bash
ls -lh evidence/Chrome_History_Cryptocurrency_Lab.db
ls -lh working/working_Chrome_History_Cryptocurrency_Lab.db
```

**Purpose:** Display the files and human-readable sizes. The source report records both as 192 KB. This display does not provide the exact byte count.

### Verify SHA-256 hashes

```bash
sha256sum evidence/Chrome_History_Cryptocurrency_Lab.db working/working_Chrome_History_Cryptocurrency_Lab.db
```

**Purpose:** Compare the hash of the original evidence and the working copy. The supplied report says the earlier comparison attempt had a typing error, so a successful comparison remains outstanding.

### Inspect the SQLite database

Open the working copy:

```bash
sqlite3 working/working_Chrome_History_Cryptocurrency_Lab.db
```

Within the SQLite prompt, list tables:

```sql
.tables
```

Inspect the relevant schemas:

```sql
PRAGMA table_info(urls);
PRAGMA table_info(visits);
PRAGMA table_info(downloads);
PRAGMA table_info(downloads_url_chains);
```

Exit SQLite with:

```sql
.quit
```

**Purpose:** Confirm which tables and fields are present before querying them. The source report lists the four tables above as central to the examination.

### Reconstruct the visit timeline

The source report provides this timeline query:

```sql
SELECT
  v.id AS visit_id,
  u.id AS url_id,
  u.url,
  u.title,
  datetime(
     v.visit_time/1000000 + strftime('%s','1601-01-01'),
     'unixepoch'
  ) AS visit_utc,
  v.from_visit,
  v.transition,
  v.visit_duration/1000.0 AS duration_ms
FROM visits AS v
JOIN urls AS u ON v.url = u.id
ORDER BY v.visit_time;
```

**Purpose:** Join each visit to its URL and title, convert `visit_time` to a formatted UTC timestamp, and show the navigation relationship and raw transition value.

**Interpretation note:** The SQL labels the converted value `visit_utc`, matching the source report. Preserve UTC when recording the timeline; the supplied findings do not document a separate local-time conversion.

### Decode basic transition types

To display the basic transition type alongside the raw value, rerun the timeline query with the following additional expression in the `SELECT` list:

```sql
(v.transition & 255) AS transition_core
```

For example, the relevant portion becomes:

```sql
SELECT
  v.id AS visit_id,
  u.id AS url_id,
  u.url,
  u.title,
  datetime(
     v.visit_time/1000000 + strftime('%s','1601-01-01'),
     'unixepoch'
  ) AS visit_utc,
  v.from_visit,
  v.transition,
  (v.transition & 255) AS transition_core,
  v.visit_duration/1000.0 AS duration_ms
FROM visits AS v
JOIN urls AS u ON v.url = u.id
ORDER BY v.visit_time;
```

**Purpose:** Separate the basic transition type from the raw stored value. Review qualifier bits and surrounding URLs before describing a visit as typed entry, link click, form submission, or redirect. This is a recommended follow-up based on the limitation identified in the source report; the decoded results were not included in the supplied findings.

### Inspect download records and URL chains

The source report identifies `downloads` and `downloads_url_chains` as the tables used to review the download and its source URL. The exact original download-query text is not preserved in the report excerpt. The following query is a practical way to inspect the reported fields; confirm column names against the database schema before relying on it.

```sql
SELECT
  id,
  current_path,
  target_path,
  start_time,
  received_bytes,
  total_bytes,
  state,
  referrer,
  site_url,
  tab_url,
  mime_type
FROM downloads
ORDER BY id;
```

Then inspect the URL chain:

```sql
SELECT
  id,
  chain_index,
  url
FROM downloads_url_chains
ORDER BY id, chain_index;
```

**Purpose:** Review the recorded download path, byte counts, MIME type, time field, and associated URL chain. The source findings report `proof_of_payment.png`, 33,844 bytes received and total, MIME type `image/png`, and the URL `https://i.imgur.com/dTgrkP7.png`.

**Outstanding check:** The exact UTC download time was not calculated in the supplied findings. Convert `start_time` according to the Chrome schema and verify the interpretation against the actual database before adding a timestamp to the final timeline.

## Findings and Results

### Evidence acquisition

The original database was recorded at:

`evidence/Chrome_History_Cryptocurrency_Lab.db`

The working copy was recorded at:

`working/working_Chrome_History_Cryptocurrency_Lab.db`

Both appeared as 192 KB in `ls -lh` output. The original SHA-256 recorded in the report is:

```text
b991b0fa69662c44caf599e468c8088576ebd9cebbf46d02c89cf51bcde3fd19
```

The supplied report does not establish that the working-copy hash matched the original because the comparison command contained a typing error.

### Database tables and fields

| Table | Relevant fields reported | Use in the examination |
|---|---|---|
| `urls` | `id`, `url`, `title`, `visit_count`, `typed_count`, `last_visit_time`, `hidden` | Connects URL IDs to recorded addresses and page titles |
| `visits` | `id`, `url`, `visit_time`, `from_visit`, `transition`, `visit_duration`, `opener_visit` | Provides visit records, timestamps, and navigation relationships |
| `downloads` | `id`, `guid`, `current_path`, `target_path`, `start_time`, `received_bytes`, `total_bytes`, `state`, `referrer`, `site_url`, `tab_url`, `mime_type` | Records download metadata and related URL information |
| `downloads_url_chains` | `id`, `chain_index`, `url` | Records URLs associated with a download chain |

The report also notes that the database contains other Chrome metadata tables, including `keyword_search_terms` and `visit_source`.

### Timeline of relevant activity

All times below are **UTC on 19 April 2022**, as displayed in the supplied SQL output. This table summarises the source report; it does not replace the underlying query output.

| Time (UTC) | Visit/URL records cited | Observed browser-history record | Interpretation and limitation |
|---|---|---|---|
| 13:54:18 | Visit 1 / URL 1 | Google search for Craigslist | Possible initial search; does not identify the operator |
| 13:54:19–13:55:03 | Visits 3–10 / URLs 2–6 | Craigslist homepage, regional pages, and account/login pages | Consistent with accessing Craigslist before posting |
| 13:55:05–13:58:27 | Visits 11–20 / URLs 7–15 | Choose type/category, create posting, add map, preview/confirmation, and manage-posting pages | Strong evidence of a posting workflow, but not proof of the complete form contents |
| About 13:58:31 | Visit 21 / URL 16 | Public listing titled “cheaper than Rx supplements” | Supports that the listing was visited; title alone does not prove illegal goods or a sale |
| 13:58:38–14:05:59 | Visits 22–47 / URLs 17–36 and 33 as cited in the report | Gmail access and an order-related page | Consistent with email activity related to the listing; message contents were not present in the supplied history output |
| 14:56:47 | Visit 48 / URL 37 | Email page title includes `txid` and a transaction hash | Shows the hash was present in the recorded page title/URL context; does not prove payment |
| 14:56:52 | Visits 49–50 / URLs 38–39 | Imgur image page, URL ending in `dTgrkP7` | Related to the image download chain; image contents were not verified from the history output |
| 14:58:59–14:59:21 | Visits 51–58 / URLs 40–44 | Blockchain.com explorer, transaction pages, and a Bitcoin address page | Consistent with checking a transaction/address; does not independently confirm ownership or payment |
| 14:59:38–15:02:05 | Visits 59–71 / URLs 45–55 | Kraken search, sign-in/device-approval, trade/funding/ledger pages, and redirect chain to mempool.space | Shows cryptocurrency-service browsing; does not prove successful exchange activity |
| 15:02:32–15:02:56 | Visits 72–74 / URL 37 | Return to the order-related Gmail page containing the transaction reference | May indicate reviewing the same email/thread after the explorer and exchange pages |
| Download time not established | Download ID 1 | `proof_of_payment.png`, 33,844 bytes; URL chain points to `https://i.imgur.com/dTgrkP7.png` | Download is recorded; exact timestamp requires conversion of the raw `start_time` field |

### Craigslist posting workflow

The URL table contains Craigslist posting pages with the following titles and IDs:

| URL ID | Page title or description in the report |
|---|---|
| 7 | `choose type` |
| 10 | `choose category` |
| 11, 13, 14 | `create posting` |
| 12 | `add map` |
| 8 | `posting confirmation` |
| 15 | `manage posting` |
| 16 | `cheaper than Rx supplements - health and beauty - by owner - …` |

The public listing URL includes identifier `7473121658`. Visits 11–20 show a sequence through posting and management pages between approximately 13:55:05 and 13:58:27 UTC, followed by a visit to the public listing at about 13:58:31 UTC.

**Finding:** This sequence supports the interpretation that the browser was used in a Craigslist posting workflow.

**Limit:** The listing title alone does not establish that the advertised products were controlled drugs or that a sale occurred. The supplied SQL output does not contain the complete listing form or full page contents.

**Evidence placeholder:** Add a readable crop of the URL query output showing URL IDs 11–21 and their titles at `screenshots/craigslist-posting-urls.png`. The source report references this material on its pages 8–9, Figure 8; the image is not embedded here as a separate screenshot file.

### Email and cryptocurrency-related browsing

The report identifies these Gmail URL records:

- URL ID 34: `cheaper than Rx supplements - order - unsub.fscs@gmail.com - Gmail`
- URL IDs 35 and 36: the same order-related title with compose parameters
- URL ID 37: a title containing `cheaper than Rx supplements -- txid:` followed by a transaction hash

Visits 43–46 show the order-related Gmail page between approximately 14:04:12 and 14:04:52 UTC. Visits 72–74 return to the email record between 15:02:32 and 15:02:56 UTC.

The transaction reference visible in the history is:

```text
517b2156914944339a96137ad8978408ea52b2fc144c98d3b0b16b21888afdc5
```

The browser history also records a Bitcoin address page with this address:

```text
38RcsURWCDmCbYocmsZCnGz1FtD7x477mt
```

The reported sequence includes a Google search for a blockchain explorer, visits to Blockchain.com, a transaction page, a Bitcoin address page, Kraken search/sign-in/device-approval pages, funding/ledger pages, and a transaction redirect that reaches mempool.space.

**Finding:** The browser accessed pages associated with a transaction reference and cryptocurrency services.

**Limit:** These records do not independently prove that a transaction was successfully made, who controlled the Bitcoin address, the source or destination of funds, or that any transaction paid for the Craigslist listing. The email body was not included in the supplied SQL output.

**Evidence placeholder:** Add readable URL-query and timeline crops at `screenshots/email-crypto-url-records.png` and `screenshots/email-crypto-timeline.png`. The report references URL-query output in Figures 9–11 and timeline output in Figures 12–13.

### Recorded PNG download

The `downloads` table records one file with a target path ending in:

```text
proof_of_payment.png
```

Reported metadata:

| Field | Reported value |
|---|---|
| Download record | ID 1 |
| Bytes received | 33,844 |
| Total bytes | 33,844 |
| MIME type | `image/png` |
| URL chain | `https://i.imgur.com/dTgrkP7.png` |
| Exact UTC download time | Not established in the supplied findings |

**Finding:** Chrome recorded a PNG download associated with the Imgur URL above.

**Limit:** The image was not supplied for examination. Its filename suggests payment evidence, but a filename does not verify what an image depicts. The exact download timestamp also remains to be calculated from `start_time`.

**Evidence placeholder:** Add a clear screenshot of the download query and URL-chain query at `screenshots/download-record-and-url-chain.png`. If the output in the original report is not legible, rerun the queries and capture fresh output.

## Analysis and Discussion

The records form a sequence that is consistent with a browser being used to create or manage a Craigslist listing, access an order-related Gmail page, view cryptocurrency transaction/address pages, visit Kraken pages, and download an Imgur-hosted PNG saved under a payment-related filename.

The strongest supported conclusions concern the browser activity itself:

- The recorded URLs and titles show a Craigslist posting workflow and a public listing visit.
- Gmail page titles include an order-related subject and a transaction reference.
- The history includes cryptocurrency explorer, Bitcoin-address, and Kraken-related URLs.
- Chrome records a PNG download whose URL chain points to `https://i.imgur.com/dTgrkP7.png`.

The evidence does **not** establish the full contents of the listing or email, the contents or authenticity of the downloaded image, wallet ownership, successful payment, a completed sale, or the identity of the person operating the browser. The source report assesses consistency with the case hypothesis as moderate confidence, while attribution and proof of an offence are low confidence on this evidence alone.

The `from_visit` links provide context for navigation relationships. Examples cited in the report include visit 13 following visit 12, visit 14 following visit 13, visit 50 linking from visit 49 to the Imgur page, visit 53 linking from visit 52 to Blockchain.com, visit 55 linking from visit 54 to a transaction search, visit 56 linking from visit 55 to the transaction page, and visit 70 linking from visit 69 to a mempool.space transaction URL. Visits 68–70 are described as a short redirect/navigation chain from Kraken-related transaction URLs to mempool.space.

These relationships should be reviewed alongside decoded transition types and the surrounding URLs. The raw transition values alone are not sufficient to label every action precisely.

## Attribution Assessment and Limitations

The following limitations were identified in the source report:

1. **Browser activity is not personal attribution.** A browser profile may have been shared, borrowed, or accessed remotely. The records do not establish who was at the keyboard.
2. **Page titles and URLs are incomplete evidence of content.** The email body, attachment contents, and full listing text were not included in the supplied SQL output.
3. **The downloaded image was not examined.** `proof_of_payment.png` is a filename, not proof of the image's contents.
4. **Cryptocurrency references do not establish ownership or purpose.** A transaction hash and Bitcoin address in URLs do not establish who controlled the address, the source or destination of funds, or a link between a transaction and the listing.
5. **Navigation types need further decoding.** Redirects and repeated visits can create multiple records for one navigation. Calculate `transition & 255` and review qualifier bits before assigning precise transition types.
6. **Hash equality remains unverified.** The reported comparison command contains a typing error; rerun it correctly.
7. **Some technical values are missing.** The exact byte size and the UTC conversion of the download `start_time` were not captured in the submitted findings.
8. **Browser history may be incomplete.** Activity in other browsers, applications, devices, private-browsing sessions, or cleared records may not appear in this database.
9. **No independent transaction verification was reported.** The history shows visits to transaction-related pages, not independent confirmation that a payment was completed.

## Evidence Log

The following log preserves the evidence references documented in the supplied report. The source file names refer to material cited within that report; separate screenshot files were not provided with the current upload.

| Evidence ID | Source record | Finding supported | Reference in source report |
|---|---|---|---|
| E01 | File-hash screenshot | Original filename and SHA-256 value | `Screenshot.pdf`, page 1 |
| E02 | File-listing screenshot | Evidence and working-copy paths and displayed sizes | `Screenshot.pdf`, page 2 |
| E03 | SQLite table list | Chrome History tables were listed | `Screenshot.pdf`, page 4 |
| E04 | `PRAGMA table_info(urls)` | URL table fields | `Screenshot.pdf`, page 6 |
| E05 | `PRAGMA table_info(visits)` | Visit table fields | `Screenshot.pdf`, page 7 |
| E06 | `PRAGMA table_info(downloads)` | Download fields | `Screenshot.pdf`, page 8 |
| E07 | `PRAGMA table_info(downloads_url_chains)` | Download URL-chain fields | `Screenshot.pdf`, page 9 |
| E08 | URL query output | Craigslist, Gmail, Imgur, blockchain, and Kraken URL records | `Screenshot.pdf`, pages 10–14; `table_info_1` as cited in source |
| E09 | Timeline SQL output | Visits, timestamps, `from_visit`, and transition values | `Screenshot.pdf`, pages 15–17; `table_info_1` as cited in source |
| E10 | Download query output | Download ID 1 and Imgur URL chain | `Screenshot.pdf`, page 17; `table_info_1` as cited in source |

## Conclusion

The supplied Chrome History database records a connected sequence of browser activity involving a Craigslist posting workflow, a listing titled “cheaper than Rx supplements,” Gmail pages referring to an order and a transaction hash, cryptocurrency explorer and address pages, Kraken pages, and a PNG download associated with Imgur.

This sequence is consistent with the case hypothesis that the browser was used to post an offer and later review transaction-related material. The conclusion remains limited to what the browser records show. Further confirmation would require examination of the actual listing content, email content, downloaded PNG, the download timestamp, and independently obtained transaction records under proper authority.

The history alone does not support a conclusion that the advertised item was a controlled drug, that a sale or payment was completed, or that a named individual performed the recorded actions.

## Recommendations and Outstanding Work

Before treating the investigation as complete, the following checks should be carried out and documented:

- [ ] Rerun `sha256sum` against the original and working-copy database and save the successful comparison output.
- [ ] Record the exact byte sizes of both files if required for the evidence log.
- [ ] Decode the basic transition types with `transition & 255` and inspect qualifier bits before making precise statements about navigation.
- [ ] Convert the download `start_time` using the correct Chrome timestamp representation and record the UTC result.
- [ ] Rerun and capture clear output for the `downloads` and `downloads_url_chains` queries.
- [ ] Add the actual screenshots referenced in the source report, with readable text and captions.
- [ ] If lawfully available and within the investigation scope, examine the actual listing, email content, downloaded PNG, and independently sourced transaction records. Record provenance and integrity information for any additional evidence.
- [ ] Keep observations separate from interpretations and record any new findings with the method used to obtain them.

## Screenshot Checklist

The image files listed below are **placeholders for repository preparation**. They are not included in this README unless you add them to the repository. The source report references pages in a file named `Screenshot.pdf`, but that PDF was not supplied as a separate attachment in this task.

| Required screenshot or evidence image | Recommended filename | README section |
|---|---|---|
| Original evidence hash and filename | `screenshots/evidence-hash-original.png` | Lab Environment and Evidence Handling |
| Original and working-copy paths and sizes | `screenshots/evidence-and-working-copy.png` | Lab Environment and Evidence Handling |
| Corrected hash comparison showing both outputs | `screenshots/hash-comparison-verified.png` | Integrity verification still required |
| SQLite table list | `screenshots/sqlite-table-list.png` | Methodology / Database discovery |
| `urls` table schema | `screenshots/schema-urls.png` | Database tables and fields |
| `visits` table schema | `screenshots/schema-visits.png` | Database tables and fields |
| `downloads` table schema | `screenshots/schema-downloads.png` | Database tables and fields |
| `downloads_url_chains` table schema | `screenshots/schema-downloads-url-chains.png` | Database tables and fields |
| URL query output for IDs 1–10 | `screenshots/url-query-ids-1-10.png` | Findings and Results |
| URL query output for IDs 11–21, including Craigslist posting pages | `screenshots/craigslist-posting-urls.png` | Craigslist posting workflow |
| URL query output for IDs 22–36, including Gmail pages | `screenshots/gmail-url-records.png` | Email and cryptocurrency-related browsing |
| URL query output for IDs 50–57 and relevant Kraken/transaction records | `screenshots/email-crypto-url-records.png` | Email and cryptocurrency-related browsing |
| Timeline output for visits 1–5 | `screenshots/timeline-visits-1-5.png` | Timeline of relevant activity |
| Timeline output for visits 71–74 | `screenshots/email-crypto-timeline.png` | Email and cryptocurrency-related browsing |
| Clear `downloads` query output | `screenshots/download-record.png` | Recorded PNG download |
| Clear `downloads_url_chains` query output | `screenshots/download-url-chain.png` | Recorded PNG download |
| Timeline/query output showing decoded transition type | `screenshots/timeline-transition-decoding.png` | Navigation relationships; add after rerunning query |

For each image added, use a relative Markdown image link near the relevant discussion, for example:

```markdown
![Craigslist posting URL records](screenshots/craigslist-posting-urls.png)

*Figure: URL query output showing the recorded Craigslist posting workflow.*
```

Do not add an image link until the corresponding file exists in the repository. If the source screenshot is a page in `Screenshot.pdf`, export or crop it legibly and preserve the original PDF in the evidence or supporting-files folder where appropriate.

## Suggested Repository Structure

A repository layout suited to the artefacts described in the report is:

```text
reconstructing-chrome-web-history/
├── README.md
├── report/
│   └── original-investigation-report.pdf
├── evidence/
│   └── Chrome_History_Cryptocurrency_Lab.db
├── working/
│   └── working_Chrome_History_Cryptocurrency_Lab.db
├── screenshots/
│   ├── evidence-hash-original.png
│   ├── evidence-and-working-copy.png
│   ├── hash-comparison-verified.png
│   ├── sqlite-table-list.png
│   ├── schema-urls.png
│   ├── schema-visits.png
│   ├── schema-downloads.png
│   ├── schema-downloads-url-chains.png
│   ├── craigslist-posting-urls.png
│   ├── gmail-url-records.png
│   ├── email-crypto-url-records.png
│   ├── timeline-visits-1-5.png
│   ├── email-crypto-timeline.png
│   ├── download-record.png
│   └── download-url-chain.png
├── queries/
│   ├── timeline.sql
│   └── download-records.sql
└── notes/
    └── evidence-log.md
```

**Repository safety note:** The structure above is a suggested layout, not a claim that all files are currently available. Do not publish a real evidence database, personal data, account information, or other restricted case artefacts to a public repository without permission and a review of the applicable privacy and evidence-handling requirements. If the database is restricted, replace it with a synthetic or approved lab dataset and explain that choice.

## References

The source report does not provide a formal bibliography or external citations. The primary source for this README is the supplied *Digital Forensics Investigation Report — Reconstructing Chrome Web History* (Course SBT-DF204, Case Study 2, dated 24 September 2026).

Tools documented in the report:
- Kali Linux
- SQLite command-line tool (`sqlite3`, version 3.53.4 in the report)
- `sha256sum`
- `ls -lh`

No additional external references have been added, and no external website or transaction service was visited as part of this reconstruction.

## Author Information

**Adekunle Ogunyemi**  
SBT-DF204 — Computer Forensics Case Studies  
Case Study 2 — Reconstructing Chrome Web History
