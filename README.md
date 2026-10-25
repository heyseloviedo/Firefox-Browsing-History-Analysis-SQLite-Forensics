# Firefox Browsing History Analysis | SQLite Forensics

## Objective

Analyze an exported Firefox browser history database to reconstruct a user's browsing activity, identify cryptocurrency-mining research, examine visited domains and search queries, and establish an activity timeline. The investigation used Linux terminal commands and SQL queries against `places.sqlite`.

### Skills Learned

- Located and identified a Firefox SQLite database.
- Examined Firefox history tables using SQLite.
- Joined `moz_places` and `moz_historyvisits` to reconstruct browsing activity.
- Converted Firefox visit timestamps into readable UTC dates and times.
- Used SQL filters to identify search queries, repositories, support pages, and potential software downloads.
- Calculated elapsed activity time and distinguished hostnames from base domains.
- Evaluated the limits of browser history as evidence of completed downloads.

### Tools Used

- Linux terminal
- `ls`
- `sqlite3`
- SQLite commands: `.tables`, `.quit`
- SQL: `SELECT`, `JOIN`, `WHERE`, `LIKE`, `ORDER BY`, `DISTINCT`, `MIN`, `MAX`, `CAST`

## Steps

### Step 1 - Locate the Firefox history database

The `ls -la` command showed a file named `places.sqlite` in the working directory. Firefox stores visited URLs and browser history metadata in this SQLite database.

```bash
ls -la
sqlite3 places.sqlite
```

<img src="images/01-locate-database.png" alt="Terminal listing showing places.sqlite" width="680">

*Ref 1: The exported Firefox history database was identified in the working directory.*

---

### Step 2 - Inspect the database tables

After opening the file with `sqlite3`, the `.tables` command listed its tables. The investigation focused on `moz_places` (URL records), `moz_historyvisits` (individual visit timestamps), and `moz_origins` (hostnames). The `.quit` command returned to the Linux shell.

```sql
.tables
.quit
```

<img src="images/02-sqlite-tables.png" alt="SQLite output listing Firefox browser history tables" width="680">

*Ref 2: The SQLite database contained the Firefox history and origin tables used in the analysis.*

---

### Step 3 - Reconstruct the first browsing activity

The history table was joined to the URL table to display visited URLs in chronological order. Firefox `visit_date` values are stored as microseconds since the Unix epoch; dividing by 1,000,000 converts them to seconds.

```bash
sqlite3 -header -column places.sqlite "SELECT datetime(h.visit_date/1000000, 'unixepoch') AS visit_time, p.url FROM moz_historyvisits h JOIN moz_places p ON h.place_id = p.id ORDER BY h.visit_date ASC LIMIT 20;"
```

The earliest recorded visit was **August 11, 2026, at 14:31:29 UTC**. Its Google search query was **`monero`**.

The chronological history also showed research into a Monero mining benchmark, XMRig configuration, GPU mining setup, and related Reddit discussions.

---

### Step 4 - Identify the GitHub repository and benchmark page

The browsing history included:

- GitHub repository: `https://github.com/xmrig/xmrig`
- Benchmark resource: `https://xmrig.com/benchmark`
- Configuration documentation: `https://xmrig.com/docs/miner/config`

These pages indicated that the user was researching XMRig and cryptocurrency-mining configuration.

---

### Step 5 - Investigate support visits and potential software downloads

The user visited an Awesome Miner support article on getting started with GPU mining and later visited its download page.

```bash
sqlite3 -header -column places.sqlite "SELECT p.url FROM moz_places p WHERE p.url LIKE '%download%' OR p.url LIKE '%.exe%' OR p.url LIKE '%.msi%';"
```

The query returned:

```text
https://www.awesomeminer.com/download
```

**Awesome Miner** was therefore the software the user *likely intended to download*. The recorded visit does **not** by itself establish that a download was completed. The visited support domain was **`support.awesomeminer.com`**.

The later Reddit activity included **`r/gpumining`**, following the Awesome Miner support visit.

---

### Step 6 - Calculate the browsing activity duration

The earliest and latest timestamps were extracted from `moz_historyvisits`. Subtracting the timestamps and dividing by 60,000,000 converted the duration from microseconds to minutes; `CAST` rounded it down to an integer.

```bash
sqlite3 -header -column places.sqlite "SELECT datetime(MIN(visit_date)/1000000, 'unixepoch') AS first_visit, datetime(MAX(visit_date)/1000000, 'unixepoch') AS last_visit, CAST((MAX(visit_date)-MIN(visit_date))/60000000 AS INTEGER) AS elapsed_minutes FROM moz_historyvisits;"
```

Results:

| Event | UTC timestamp |
| --- | --- |
| First recorded activity | 2026-08-11 14:31:29 |
| Last recorded activity | 2026-08-11 14:36:39 |
| Elapsed time | **5 minutes, 10 seconds** |
| Whole minutes (rounded down) | **5** |

---

### Step 7 - Review hostnames and base domains

The `moz_origins` table exposed the hostnames associated with records in Firefox's database.

```bash
sqlite3 -header -column places.sqlite "SELECT DISTINCT host FROM moz_origins ORDER BY host;"
```

<img src="images/03-investigation-results.png" alt="Terminal showing time calculations, search queries and Firefox origin hostnames" width="680">

*Ref 3: SQLite queries identified the activity window, recent searches, and hostnames stored in the database.*

The output included nine hosts, among them `support.awesomeminer.com` and `www.awesomeminer.com`, which share one base domain, as well as `support.mozilla.org` and `www.mozilla.org`, which also share a base domain.

**Important distinction:** The `moz_origins` table can contain stored origins beyond the visited entries in `moz_historyvisits`. A count of **7 distinct base domains from all stored origins was rejected by the challenge**, so it should not be presented as the confirmed visited-domain count. A visit-scoped query is needed to establish the challenge's intended answer:

```sql
SELECT DISTINCT o.host
FROM moz_historyvisits AS h
JOIN moz_places AS p ON h.place_id = p.id
JOIN moz_origins AS o ON p.origin_id = o.id
ORDER BY o.host;
```

---

### Step 8 - Identify the final search query

A SQL filter selected Google search URLs and sorted them from newest to oldest.

```bash
sqlite3 -header -column places.sqlite "SELECT datetime(h.visit_date/1000000, 'unixepoch') AS visit_time, p.url FROM moz_historyvisits h JOIN moz_places p ON h.place_id = p.id WHERE p.url LIKE '%google.com/search?%' ORDER BY h.visit_date DESC LIMIT 5;"
```

The final Google query occurred at **2026-08-11 14:36:32 UTC**:

```text
how to optimize gpu mining
```

### Findings

The browser history shows a short sequence of research into Monero, XMRig, mining benchmarks, GPU mining, Awesome Miner, and related Reddit communities. By reconstructing the history in chronological order, it was possible to identify the first and last searches, specific technical resources, likely download interest, and the duration of the recorded activity.

This exercise demonstrates how SQLite browser artifacts can support a digital forensic timeline while also showing why investigators should distinguish visited pages from completed actions and stored origins from confirmed visits.

---

*Evidence source: an exported Firefox `places.sqlite` database provided in a controlled forensic challenge. Screenshots document the investigation; the full browser database is not included in this repository.*
