# SBT-DF204 – Computer Forensics Case Study 2  
## Reconstructing Chrome Web History

**Digital Forensics Investigation Report**

| Field | Details |
| --- | --- |
| Course Code | SBT-DF204 |
| Course Title | Computer Forensics Case Studies |
| Lab Title | Reconstructing Chrome Web History |
| Platform | Kali Linux Virtual Machine |
| Student | Adekunle Ogunyemi |
| Student ID | 2025/FWSD/11479 |
| Date | 12 October 2026 |
| Assessment | Individual browser history investigation |
| Evidence file | Chrome_History_Cryptocurrency_Lab.db |

---

## Table of Contents

1. [Executive Conclusion](#1-executive-conclusion)
2. [Scope and Objectives](#2-scope-and-objectives)
3. [Evidence Acquisition and Integrity](#3-evidence-acquisition-and-integrity)
4. [Database Method](#4-database-method)
5. [Database Structure](#5-database-structure)
6. [Findings: Answers to the Investigation Questions](#6-findings-answers-to-the-investigation-questions)
   - [6.1 What evidence file was acquired and how was it preserved?](#61-what-evidence-file-was-acquired-and-how-was-it-preserved)
   - [6.2 What tables and fields were relevant?](#62-what-tables-and-fields-were-relevant)
   - [6.3 When did the recorded web activity occur?](#63-when-did-the-recorded-web-activity-occur)
   - [6.4 How did the browser reach the material URLs?](#64-how-did-the-browser-reach-the-material-urls)
   - [6.5 What evidence relates to the suspected sale or posting?](#65-what-evidence-relates-to-the-suspected-sale-or-posting)
   - [6.6 What evidence relates to communications and payment activity?](#66-what-evidence-relates-to-communications-and-payment-activity)
   - [6.8 What conclusion can be defended?](#68-what-conclusion-can-be-defended)
7. [Attribution Assessment and Limitations](#7-attribution-assessment-and-limitations)
8. [Evidence Log](#8-evidence-log)
9. [Conclusion](#9-conclusion)
10. [Appendices and Screenshot Placement](#10-appendices-and-screenshot-placement)
11. [Author](#author)

---

## 1. Executive Conclusion

The supplied Chrome History database contains a sequence of recorded browser visits on 19 April 2022. The records show activity on Craigslist, including a posting workflow and a public listing titled “cheaper than Rx supplements”; later visits show Gmail pages associated with an order-related subject, an Imgur image page, a blockchain explorer transaction page, a Bitcoin address page, and Kraken pages including a transaction redirect and a ledger/history page. A download record identifies a file named proof_of_payment.png with a size of 33,844 bytes, and its URL chain points to an Imgur-hosted PNG image.

Taken together, these records are consistent with a workflow involving a posting, email-related activity, cryptocurrency-reference checking, and retention of an image described as proof of payment. However, Chrome history records browser-profile activity, not the identity of the person at the keyboard. The records do not by themselves prove the actual contents of the email or image, the transfer of drugs, the completion of a payment, or the identity of a buyer or seller. The finding that the browser history is consistent with the case hypothesis is assessed as moderate confidence; attribution to a specific person or proof of an offence is low confidence on this evidence alone.

## 2. Scope and Objectives

The purpose of this examination was to review the supplied Chrome History SQLite database and reconstruct a time-ordered account of relevant browser activity. The work focused on the urls, visits, downloads, and downloads_url_chains tables, Chrome WebKit timestamp conversion, navigation transitions, and links between records. No live websites, email accounts, cryptocurrency wallets, exchanges, or transaction services were visited as part of this analysis.

## 3. Evidence Acquisition and Integrity

<img width="1647" height="295" alt="Screenshot 2026-10-09 130108" src="https://github.com/user-attachments/assets/4758a82e-3896-4ba0-a714-ed2b975f7cb9" />

*Fig. 1 - Evidence and working-copy file listing*

The screenshots show the evidence file stored at evidence/Chrome_History_Cryptocurrency_Lab.db and a working copy stored at working/working_Chrome_History_Cryptocurrency_Lab.db in the case-study folder on Kali.

| Item | Recorded value |
| --- | --- |
| Original evidence filename | Chrome_History_Cryptocurrency_Lab.db |
| Evidence location shown | evidence/Chrome_History_Cryptocurrency_Lab.db |
| Working-copy location shown | working/working_Chrome_History_Cryptocurrency_Lab.db |
| Original file size | 192 KB as displayed by ls -lh (exact byte count not recorded in the supplied findings) |
| Working-copy size | 192 KB as displayed by ls -lh |
| SHA-256 shown for original | b991b0fa69662c44caf599e468c8088576ebd9cebbf46d02c89cf51bcde3fd19 |
| SHA-256 comparison | The screenshots show an attempted hash comparison. The command shown contains a typing error, so the comparison should be rerun correctly before claiming verified hash equality. |
| Tools | Kali Linux terminal; sqlite3 version 3.53.4; sha256sum; ls -lh |

<img width="1632" height="347" alt="Screenshot 2026-10-09 130534" src="https://github.com/user-attachments/assets/d4fc9115-267b-4953-8fbe-d0193e3322ec" />

*Figure 2. SHA-256 hash comparison.*

**Integrity note:** The original SHA-256 value is visible in the screenshot. For a complete chain-of-custody record, rerun `sha256sum evidence/Chrome_History_Cryptocurrency_Lab.db working/working_Chrome_History_Cryptocurrency_Lab.db` and record that both output hashes match.

## 4. Database Method

The database was opened with sqlite3. The table list and table structures were inspected before reviewing records. The supplied output lists tables including urls, visits, downloads, downloads_url_chains, keyword_search_terms, visit_source, and other Chrome metadata tables. The main analysis used urls and visits joined through visits.url = urls.id. I examined download information separately and correlated it with the URL chain.

The main timeline query used the following method:

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

Chrome stores visit_time as microseconds since 1 January 1601 UTC. The query converts this value to a Unix timestamp and formats it as a UTC date/time. The timeline below uses the times displayed in the supplied SQL output. The outputs do not document a separate local-time conversion, so no local-time values are reported.

The transition column contains large numeric values, including values displayed as negative numbers. These are raw stored values and include transition qualifiers. They should not be interpreted by their decimal value alone. The assignment method is to obtain the basic type using `transition & 255` and interpret qualifier bits together with from_visit and the surrounding URLs. The supplied findings do not show a separate result column for `transition & 255`; therefore, this report avoids assigning a specific navigation type to every row where that calculation is not directly visible.

## 5. Database Structure

| Table | Relevant fields shown | Use in this examination |
| --- | --- | --- |
| urls | id, url, title, visit_count, typed_count, last_visit_time, hidden | Connects URL IDs to recorded addresses and page titles. |
| visits | id, url, visit_time, from_visit, transition, visit_duration, opener_visit | Provides individual visit records, timestamps, and navigation relationships. |
| downloads | id, guid, current_path, target_path, start_time, received_bytes, total_bytes, state, referrer, site_url, tab_url, mime_type | Records the downloaded file, size, path, and related URL information. |
| downloads_url_chains | id, chain_index, url | Records the URL chain associated with a download. |

<img width="1656" height="472" alt="Screenshot 2026-10-12 085311" src="https://github.com/user-attachments/assets/d85b51df-96a3-4ede-a76e-5048ebee477a" />

*Figure 3. Schema of the **urls** table*

<img width="1661" height="541" alt="Screenshot 2026-10-12 085354" src="https://github.com/user-attachments/assets/74efd3c4-34c3-432c-80d9-af6150fc70c5" />

*Figure 4. Schema of the visits table*

<img width="1646" height="703" alt="Screenshot 2026-10-12 085511" src="https://github.com/user-attachments/assets/317cc518-4faa-4a94-9f93-a85c1a814eb8" />

*Figure 5. Schema of the downloads table*

<img width="1673" height="363" alt="Screenshot 2026-10-12 085610" src="https://github.com/user-attachments/assets/ccb596a2-3f8e-42b6-8b45-61cb3d8fef70" />

*Figure 6. Schema of the **downloads_url_chains** table*

## 6. Findings: Answers to the Investigation Questions

### 6.1 What evidence file was acquired and how was it preserved?

The supplied file was Chrome_History_Cryptocurrency_Lab.db. The screenshots show it in the evidence folder and a separate working copy in the working folder. The original SHA-256 shown is b991b0fa69662c44caf599e468c8088576ebd9cebbf46d02c89cf51bcde3fd19. Both files were displayed as 192 KB. The evidence was examined with sqlite3 rather than a browser.

### 6.2 What tables and fields were relevant?

The urls table supplied URL IDs, page titles, and visit counts. The visits table supplied visit IDs, URL IDs, visit_time, from_visit, transition, and visit_duration. The downloads table supplied a download record with a target path ending in proof_of_payment.png, received/total bytes of 33,844, MIME type image/png, and a referrer associated with Imgur.

The downloads_url_chains table contained the URL https://i.imgur.com/dTgrkP7.png. These tables allowed the URL visits and the download to be compared.

### 6.3 When did the recorded web activity occur?

The material activity in the supplied visit output occurred on 19 April 2022 between approximately 13:54:18 and 15:02:56 UTC. The key sequence began with Craigslist searches and a posting workflow at about 13:54–13:58, continued with Gmail activity from about 13:58, and later included a payment-related email subject, Imgur, Blockchain.com, Kraken, and mempool.space from 14:56 to 15:02. These times are taken from the supplied SQL output and should be preserved in UTC in the report.

### 6.4 How did the browser reach the material URLs?

The visits output contains from_visit links that connect related browser records. For example, visit 13 follows visit 12, visit 14 follows visit 13, and the Craigslist posting workflow continues through visits 11–20. Later, visit 50 links from visit 49 to the Imgur page; visit 53 links from visit 52 to Blockchain.com; visit 55 links from visit 54 to a transaction search; visit 56 links from visit 55 to the transaction page; and visit 70 links from visit 69 to a mempool.space transaction URL. Visits 68–70 show a short redirect/navigation chain from Kraken-related transaction URLs to mempool.space. The raw transition values should be decoded with `transition & 255` and their qualifier bits checked in a rerun query before making a precise statement about typed entry, link click, form submission, or redirect type.

### 6.5 What evidence relates to the suspected sale or posting?

The URL table contains Craigslist posting pages with titles such as “choose type” (URL ID 7), “choose category” (URL ID 10), “create posting” (URL IDs 11, 13, and 14), “add map” (URL ID 12), “posting confirmation” (URL ID 8), and “manage posting” (URL ID 15). The listing URL is URL ID 16 and its title is “cheaper than Rx supplements - health and beauty - by owner - …”; the URL includes the listing identifier 7473121658. Visits 11–20 show a chronological sequence through the posting pages and management page between 13:55:05 and 13:58:27 UTC, followed by a visit to the public listing at about 13:58:31 UTC. This supports the interpretation that the browser was used in a Craigslist posting workflow. The listing title alone does not establish that the advertised products were controlled drugs or that a sale took place.

<img width="1646" height="827" alt="Screenshot 2026-10-12 090545" src="https://github.com/user-attachments/assets/24beea0c-cfcb-455f-9875-6b0feaeddd66" />

*Figure 7. URL query output (URL IDs 1–10): Google search for Craigslist, Craigslist homepage, regional page, account login, and posting workflow pages.*

<img width="1653" height="872" alt="Screenshot 2026-10-12 090602" src="https://github.com/user-attachments/assets/4fba244b-164f-414c-959d-ed808f26832e" />

*Figure 8. URL query output (URL IDs 11–21): Craigslist "create posting", "add map", "preview", "manage posting", and the public listing "cheaper than Rx supplements"*

### 6.6 What evidence relates to communications and payment activity?

The browser history includes Gmail records. URL ID 34 has the title “cheaper than Rx supplements - order - unsub.fscs@gmail.com - Gmail”; URL IDs 35 and 36 have the same order-related title with compose parameters; URL ID 37 has a title containing “cheaper than Rx supplements -- txid:” followed by the transaction hash. Visits 43–46 show the order-related Gmail page between 14:04:12 and 14:04:52 UTC. Visits 72–74 return to the email record between 15:02:32 and 15:02:56 UTC.

<img width="1663" height="860" alt="Screenshot 2026-10-12 090617" src="https://github.com/user-attachments/assets/e51c91ea-713c-466a-a3ab-8f45559f9b6a" />

*Figure 9. URL query output (URL IDs 22–29): Gmail help/about pages and Google account sign-in flow.*

<img width="1680" height="882" alt="Screenshot 2026-10-12 090634" src="https://github.com/user-attachments/assets/4e7f0f4c-a974-4833-82ff-3b0cd844d5f9" />

*Figure 10. URL query output (URL IDs 30–36): Gmail inbox and the order-related email thread titled "cheaper than Rx supplements - order - unsub.fscs@gmail.com".*

The transaction reference visible in the history is 517b2156914944339a96137ad8978408ea52b2fc144c98d3b0b16b21888afdc5. The browser records show a Google search for a blockchain explorer, visits to Blockchain.com, a transaction page, and a Bitcoin address page with address 38RcsURWCDmCbYocmsZCnGz1FtD7x477mt. The browser then records a Kraken search, sign-in and device-approval pages, funding/ledger pages, and a transaction redirect that reaches mempool.space. These records show that the browser accessed pages associated with a transaction reference and cryptocurrency services. They do not independently prove that a transaction was successfully made, who controlled the address, or that any transaction was payment for the Craigslist listing.

<img width="1623" height="840" alt="Screenshot 2026-10-12 090655" src="https://github.com/user-attachments/assets/eaea822f-c7de-4ec6-a783-212f2c29f058" />

*Figure 11. URL query output (URL IDs 50–57): Kraken funding/ledger pages, Kraken redirect to mempool.space, and Gmail transaction-reference page.*

The downloads table records one file with a target path ending in proof_of_payment.png, 33,844 bytes received and total, and MIME type image/png. The associated download URL chain is https://i.imgur.com/dTgrkP7.png. This establishes that a PNG download was recorded from that URL. The filename suggests payment evidence, but the image contents were not provided for examination, and the filename alone does not verify what the image depicts.

| UTC time | Record IDs | Artifact | Observed fact | Interpretation and limit |
| --- | --- | --- | --- | --- |
| 13:54:18 | Visit 1 / URL 1 | Google search for Craigslist | Browser history records a Craigslist search. | Possible initial search; does not identify the operator. |
| 13:54:19–13:55:03 | Visits 3–10 / URLs 2–6 | Craigslist homepage and account pages | Visits to Craigslist regional pages and account login pages. | Consistent with accessing Craigslist before posting. |
| 13:55:05–13:58:27 | Visits 11–20 / URLs 7–15 | Craigslist posting workflow | Choose type/category, create posting, add map, preview/confirmation and manage-posting pages are recorded. | Strong evidence of a posting workflow, but not proof of the full form contents. |
| 13:58:31 | Visit 21 / URL 16 | Public listing | Listing titled “cheaper than Rx supplements” was visited. | Supports a listing being accessible; title alone does not prove illegal goods or a sale. |
| 13:58:38–14:05:59 | Visits 22–47 / URLs 17–36 and 33 | Gmail access and order-related page | Gmail pages are visited; order-related subject pages appear at 14:04:12–14:04:52. | Consistent with email activity related to the listing; message contents are not present in the supplied history output. |
| 14:56:47 | Visit 48 / URL 37 | Email with transaction reference | Gmail title includes “txid” and a transaction hash. | Shows the hash was present in the recorded page title/URL context; does not prove a payment. |
| 14:56:52 | Visits 49–50 / URLs 38–39 | Imgur image page | The browser opens an Imgur URL ending in dTgrkP7. | Related to the image download chain; content not verified from this history output. |
| 14:58:59–14:59:21 | Visits 51–58 / URLs 40–44 | Blockchain explorer, transaction and address | Blockchain.com search/transaction pages and a Bitcoin address page are recorded. | Consistent with checking a transaction/address; no independent confirmation of ownership or payment. |
| 14:59:38–15:02:05 | Visits 59–71 / URLs 45–55 | Kraken and transaction redirect | Kraken search, sign-in/device approval, trade/funding/ledger pages and a redirect chain to mempool.space appear. | Shows cryptocurrency-service browsing; does not prove successful exchange activity. |
| 15:02:32–15:02:56 | Visits 72–74 / URL 37 | Return to order-related email | Browser returns to the Gmail page containing the transaction reference. | May indicate reviewing the same email/thread after the explorer and exchange pages. |
| Download record time shown in DB output | Download ID 1 | proof_of_payment.png | A 33,844-byte PNG download is recorded; chain URL is https://i.imgur.com/dTgrkP7.png. | File name suggests payment proof, but the image itself was not examined. Exact download timestamp needs conversion from the raw start_time field. |

<img width="1655" height="863" alt="Screenshot 2026-10-12 091253" src="https://github.com/user-attachments/assets/96b1ee5d-6b94-41e1-b71c-598084fd7757" />

*Figure 12. Timeline SQL output (visits 1–5): Initial Google search for Craigslist and first Craigslist page visits, showing UTC timestamps and raw transition values.*

<img width="1661" height="855" alt="Screenshot 2026-10-12 091328" src="https://github.com/user-attachments/assets/67753956-5734-494e-9563-c2e21a6712cd" />

*Figure 13. Timeline SQL output (visits 71–74): Return to the order-related Gmail page containing the transaction hash, showing **from_visit** links and UTC timestamps.*

### 6.8 What conclusion can be defended?

The browser history supports a moderate-confidence conclusion that the Chrome profile recorded a sequence consistent with creating/managing a Craigslist listing, visiting an email page whose title refers to an order, viewing a cryptocurrency transaction/address, accessing Kraken pages, and downloading an Imgur PNG saved as proof_of_payment.png. The records do not establish, beyond the browser activity itself, that the advertised item was a controlled drug, that a sale occurred, or that the same human performed every action.

## 7. Attribution Assessment and Limitations

- Browser-profile activity is not the same as proof of a particular person’s actions. The device or profile may have been shared, borrowed, or accessed remotely.
- A URL and page title do not preserve the complete page contents. The email body, attachment contents, and actual listing text were not included in the supplied SQL output.
- The download filename proof_of_payment.png is a label, not proof of the image contents. The image file itself was not examined as part of the supplied findings.
- The transaction hash and Bitcoin address appearing in URLs do not establish wallet ownership, the source or destination of funds, or that the transaction related to the listing.
- Browser redirects and repeated visits can create several records for one navigation. The transition values include qualifier bits and some are displayed as signed values; the basic transition type should be calculated with `transition & 255` before assigning exact types.
- The supplied hash-comparison command contains a typing error, so equality between the original and working-copy hashes needs to be confirmed with a corrected command.
- The exact byte size and the download start_time converted to UTC were not captured in the submitted findings. These should be added from the database if available.
- The browser history may omit activity performed in other browsers, applications, devices, private-browsing sessions, or records that were cleared.

## 8. Evidence Log

| Evidence ID | Source record | Finding | Why it matters | Reference |
| --- | --- | --- | --- | --- |
| E01 | File hash screenshot, page 1 | Original file SHA-256 and filename | Supports evidence identification and integrity record | Screenshot.pdf, page 1 |
| E02 | File listing screenshot, page 2 | Original and working-copy paths and displayed sizes | Shows separate evidence and working folders | Screenshot.pdf, page 2 |
| E03 | SQLite table list, page 4 | Available Chrome History tables | Shows database opened and schema discovery started | Screenshot.pdf, page 4 |
| E04 | PRAGMA table_info(urls), page 6 | URLs table fields | Supports URL/title/visit-count analysis | Screenshot.pdf, page 6 |
| E05 | PRAGMA table_info(visits), page 7 | Visits table fields | Supports timeline and navigation analysis | Screenshot.pdf, page 7 |
| E06 | PRAGMA table_info(downloads), page 8 | Download fields | Supports download artefact analysis | Screenshot.pdf, page 8 |
| E07 | PRAGMA table_info(downloads_url_chains), page 9 | Download URL-chain fields | Supports correlation of download and source URL | Screenshot.pdf, page 9 |
| E08 | URL query output, pages 10–14 | Craigslist posting, Gmail, Imgur, blockchain and Kraken URL records | Identifies relevant URL IDs and titles | Screenshot.pdf, pages 10–14; table_info_1 |
| E09 | Timeline SQL output, pages 15–17 | Visits, timestamps, from_visit and transition values | Supports chronological reconstruction | Screenshot.pdf, pages 15–17; table_info_1 |
| E10 | Download query output, page 17 / text output | Download ID 1 and Imgur URL chain | Supports the recorded PNG download | Screenshot.pdf, page 17; table_info_1 |

## 9. Conclusion

The supplied Chrome History records provide a connected sequence of browser activity involving a Craigslist posting workflow, an advertisement titled “cheaper than Rx supplements,” Gmail pages referring to an order, a transaction hash, cryptocurrency explorer pages, Kraken pages, and a PNG download associated with Imgur. This sequence is consistent with the case hypothesis that the browser was used to post an offer and later review transaction-related material. The conclusion remains limited to what the browser records show. Further confirmation would require examination of the actual listing content, email content, downloaded PNG, relevant download timestamp, and independently obtained transaction records under proper authority. No conclusion about a named individual or completed drug sale can be made from the browser history alone.

## 10. Appendices and Screenshot Placement

- Appendix A – Evidence acquisition and SHA-256: insert Screenshot.pdf page 1.
- Appendix B – Original and working-copy locations/sizes: insert Screenshot.pdf page 2.
- Appendix C – Hash comparison attempt: insert Screenshot.pdf page 3 and add a new screenshot after rerunning the corrected hash comparison.
- Appendix D – Table list: insert Screenshot.pdf page 4.
- Appendix E – Schema outputs: insert Screenshot.pdf pages 6–9.
- Appendix F – URL query results: insert Screenshot.pdf pages 10–14. Crop or enlarge the pages where necessary so the URLs and titles are readable.
- Appendix G – Timeline query results: insert Screenshot.pdf pages 15–17. Include the saved text/CSV output where the terminal screenshot is too small to read.
- Appendix H – Download record: insert a clear screenshot of the downloads query and downloads_url_chains query. If not already legible on page 17, rerun the query and capture a clearer image.

---

## Author

**Adekunle Ogunyemi**  
SBT-DF204 – Computer Forensics Case Studies  
Case Study 2 – Reconstructing Chrome Web History
