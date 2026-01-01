# New User & BOD Process User Guide

## This Document Includes

This document provides a complete, step-by-step guide for new users and developers to understand and work with the BOD process.

It includes:


- How to add a new file to the BOD process  
- How to add a new process to the BOD execution flow  
- How to verify changes using the UI  
- Common mistakes and how to avoid them  
- What to do if a BOD process fails  


This section explains the **project folder structure** and describes **exactly where and how a new file-parsing script should be added** so that it becomes part of the BOD process.

If you want to add a new file to the BOD flow, **this is the first section you must read**.

## 1.1 High-Level Project Structure

At the root level, the project contains the following important items:

```
project-root/
│
├── lib/
│ ├── config.py
│ ├── email.py
│ └── logger.py
│
├── utils/
│ ├── (multiple BOD and EOD scripts)
│
├── process_master.py
├── process.json
└── requirement.txt
```



Each of these plays a specific role in the BOD process.

---

## 1.2 Purpose of Each Folder and File

### 1.2.1 `lib/` Folder (Shared Utilities)

The `lib` folder contains **shared support files** used by all BOD and EOD scripts.

```
lib/
├── config.py
├── email.py
└── logger.py
```

**What this folder is for:**
- Common configuration values
- Common logging behavior
- Common alert or notification handling

**Important rules:**
- New BOD scripts should **use** these files
- New BOD scripts should **not duplicate** logic already present here
- This folder should not contain business logic for individual files

📌 Think of this folder as **system support**, not processing logic.

---

### 1.2.2 `utils/` Folder (Actual BOD & EOD Work)

The `utils` folder contains **all scripts that perform real BOD and EOD work**.

This is where **new file-parsing scripts must be added**.

📸 **Screenshot Placeholder**  
> utils folder showing existing BOD and EOD scripts

From the existing structure, you can observe:
- Each file handles **one specific responsibility**
- Most files are named clearly based on what they do
- Each script is designed to run independently

---

### 1.2.3 `process_master.py` (Central Runner)

`process_master.py` is the **central controller** of the system.

**What it does:**
- Reads `process.json`
- Identifies which processes exist
- Decides execution order
- Executes scripts from the `utils` folder
- Updates execution status for the UI

⚠️ Important:
- You **never add logic directly** into `process_master.py`
- You only **register** new scripts so that `process_master.py` can execute them

---

### 1.2.4 `process.json` (Process Registration File)

`process.json` is the **most important configuration file** for BOD.

**What it defines:**
- Which scripts are part of BOD
- Execution order
- Script locations
- Required parameters

If a script is **not present in `process.json`**, it will:
- Never run
- Never appear in the UI
- Never be part of BOD

---

### 1.2.5 `requirement.txt`

This file lists all required dependencies for the project.

**Rules:**
- If your new script requires something new, it must be added here
- Existing scripts rely on this file to work consistently


## 1.3 Script Responsibilities (Purpose, Input, Output & Dependencies)

### 📄 Script: download_nse_most_active_scripts.py

`copy_files_src_dst.py` is a **foundational pre-processing script** in the BOD workflow.
Its sole responsibility is to **guarantee the availability of input files and directories**
at the correct locations **before any downstream processing begins**.

In a production BOD system, the most frequent causes of failure are not code issues,
but operational issues such as:
- input files arriving late or in unexpected locations
- files being required by multiple downstream processes
- partial or inconsistent file availability

This script is designed to **isolate and eliminate these risks** by acting as a
**gatekeeper for file availability**.

### 📄 Script: db_backup.py

`db_backup.py` is a **critical safety and risk-mitigation script** in the BOD workflow.
Its primary purpose is to create a **consistent, recoverable backup of the database**
*before any BOD process modifies data*.

In production BOD systems, data corruption or partial updates can occur due to:
- unexpected script failures
- incorrect input data
- infrastructure or network issues

This script ensures that a **known-good database state** is always available for recovery
in case any downstream BOD step fails.

> 🔒 This script must always run **before** any database-modifying BOD scripts.
---

### 2️⃣ Execution Context

The script is designed as a **standalone command-line utility** and can be executed in two ways:

- Automatically by `process_master.py` during the BOD startup phase
- Manually by operations or developers for emergency or ad-hoc backups

The script performs **read-only operations** on the database and does not depend on
any external application state.

---

`docker exec <mysql_container> mysqldump ...`

3️⃣ Runtime Dependencies

#### 3.1 System-Level Dependencies

The following must be available at runtime:

- Docker installed and running
- MySQL Docker container running
- `mysqldump` available inside the container
- `gzip` available for compression

If any of these dependencies are missing, the script will fail.

#### 3.2 Python Dependencies

The script uses only Python standard library modules:

- `os` – directory validation and file handling
- `subprocess` – command execution and piping
- `datetime` – timestamp generation
- `sys` – platform checks
- `argparse` – command-line argument parsing

  #### Required Arguments
- `--user` : MySQL username  
- `--password` : MySQL password  
- `--database` : Database name (ignored if `--all-databases` is used)  
- `--output_dir` : Directory where backup files are stored  

- `--host` (default: `localhost`)
- `--port` (default: `3306`)
- `--all-databases` : Backup all databases instead of a single database
- `--no_compress` : Disable gzip compression
- `--mysql_container_name` (default: `blitz_mysql`)
---
### 📄 Script: db_backup.py

## 1️⃣ Purpose and Design Intent

`db_clean.py` is a **database preparation and hygiene script** executed during the early
phase of the BOD workflow.  
Its primary responsibility is to ensure that the database is in a **clean, predictable state**
before fresh BOD data is loaded.

Unlike a full database reset, this script supports **two controlled cleanup strategies**:
1. **Hard cleanup (TRUNCATE)** – permanently removes all data from selected tables
2. **Soft cleanup (DELETE WHERE IsDeleted = TRUE)** – removes logically deleted rows only

This dual-mode design allows the system to:
- preserve important reference or configuration data
- safely remove stale or transient records from previous runs
- avoid unintended data loss

> This script must always run **after `db_backup.py`**  
>  This script must run **before any data ingestion or upload scripts**

### 2️⃣ Execution Context

The script is implemented as a **standalone command-line utility** and can be executed:

- Automatically by `process_master.py` as part of the BOD flow
- Manually by operations or developers for controlled database cleanup

The script performs **write operations** on the database and therefore must be treated
as a **high-impact script**.
### 3️⃣ Runtime Dependencies

#### 3.1 System and Environment Dependencies

- MySQL database must be running and accessible
- Network connectivity to the database host
- Sufficient privileges to:
  - TRUNCATE tables
  - DELETE rows
  - READ table metadata

---

#### 3.2 Python Dependencies

This script depends on:

- `mysql-connector-python`  
  (must be installed via `pip install mysql-connector-python`)

Standard library modules:
- `argparse` – command-line argument parsing

4️⃣ Input Contract (Command-Line Arguments)

The script relies entirely on explicit command-line inputs.

#### Required Arguments
- `--user` : Database username  
- `--password` : Database password  
- `--database` : Database name  
- `--host` (default: `localhost`)
- `--port` (default: `3306`)
- `--tables`
- `--blitz_server` (currently informational)


## 📄 Script : download_and_process_bhavcopy.py
`download_and_process_bhavcopy.py` is a **core market data ingestion script** in the BOD workflow.
Its primary responsibility is to **download, extract, validate, and process daily NSE Bhavcopy data**
and make it available for downstream analytics, reporting, and trading-related processes.

This script acts as the **single source of truth for daily market reference data**, including:
- Cash Market (CM) equity prices
- Futures & Options (FO) contract data
- Index OHLC data (e.g., NIFTY 50, BANK NIFTY)

>  This script should be executed **after market close**  
>  Downstream systems rely on the correctness of this data

### 2️⃣ Data Sources (From Where the Files Are Downloaded)

This script downloads data **only from official NSE-controlled endpoints**.
No third-party or intermediary data providers are used.

---

#### 2.1 NSE Bhavcopy Files (CM & FO)

**Primary domain:**
[https://nsearchives.nseindia.com](https://nsearchives.nseindia.com)
#####  Cash Market (CM) Bhavcopy
[https://nsearchives.nseindia.com/content/cm/](https://nsearchives.nseindia.com/content/cm/)
`BhavCopy_NSE_CM_0_0_0_<YYYYMMDD>_F_0000.csv.zip`


**URL pattern:**
**Contains:**
- Equity OHLC prices
- Traded volume and value
- Settlement prices

---

##### 📌 Futures & Options (FO) Bhavcopy

**URL pattern:**
[https://nsearchives.nseindia.com/content/fo/](https://nsearchives.nseindia.com/content/fo/)
`BhavCopy_NSE_FO_0_0_0_<YYYYMMDD>_F_0000.csv.zip`

**Contains:**
- Futures and Options contracts
- Expiry dates, strike prices
- Volume, Open Interest, settlement prices

---

#### 2.2 Index OHLC Data

Index data is fetched using an NSE-backed API hosted on:
[https://www.niftyindices.com](https://www.niftyindices.com)
**API Endpoint:**
[https://www.niftyindices.com/Backpage.aspx/getHistoricaldatatabletoString](https://www.niftyindices.com/Backpage.aspx/getHistoricaldatatabletoString)
### 3️⃣ Execution Context

The script is designed as a **standalone executable utility** and can be run:

- Automatically via `process_master.py` during BOD
- Manually for validation, reprocessing, or historical backfill

The script performs **both network I/O and database writes**, making it a **high-impact script**.

---
### 4️⃣ Runtime Dependencies
The script depends on **live availability of official NSE services**.

Mandatory external endpoints:
- `https://nsearchives.nseindia.com`  
  (For CM & FO Bhavcopy ZIP files)
- `https://www.niftyindices.com`  
  (For index OHLC data API)

SE enforces request filtering and anti-bot mechanisms.

The script **explicitly depends on**:
- Valid `User-Agent` header
- Valid `Referer: https://www.nseindia.com/`
- Persistent HTTP session usage

If headers are modified or removed:
- NSE may return HTTP 403 / 401
- Download will fail even if URL is correct

### 5️⃣ Failure Handling and Edge Cases

#### Common Failure Scenarios
- NSE endpoint unavailable
- Bhavcopy not yet published for the day
- Partial or corrupt ZIP downloads
- CSV format changes
- Database connectivity issues

#### Failure Behavior
- Script logs detailed error messages
- Processing stops on critical failures
- No partial database writes are committed

This ensures **data integrity and consistency**.

---
## 📄 Script: download_banned_instruements.py
### 1️⃣ Purpose and Design Intent

`download_banned_instruements.py` is a **regulatory compliance and risk-control script**
executed as part of the BOD workflow.

Its primary responsibility is to:
- Download the **official NSE list of banned F&O instruments**
- Persist the data into the internal database
- Ensure the trading system is aware of instruments that must be blocked

This script directly supports **exchange compliance**, **risk management**, and
**order validation logic**.

> 🔒 If this script is skipped or fails, the system may allow trading
> in instruments that are officially banned by the exchange.

---
### 2️⃣ Data Source (From Where the File Is Downloaded)

The script downloads data **directly from an official NSE archive endpoint**.

#### 📥 Source URL
[https://nsearchives.nseindia.com/content/fo/fo_secban.csv](https://nsearchives.nseindia.com/content/fo/fo_secban.csv)
#### 📄 File Description
- File Name: `fo_secban.csv`
- Published by: **National Stock Exchange (NSE)**
- Update Frequency: Daily (post exchange updates)
- Content Type: CSV

#### 📌 File Contents
| Column | Description |
|-----|-------------|
| Index | Row index (not used in DB) |
| Symbol | Trading symbol banned by NSE |

This file represents the **authoritative list** of banned F&O instruments.

---

### 3️⃣ Execution Context

The script is implemented as a **standalone CLI utility** and is executed:

- Automatically via `process_master.py` during BOD
- Manually for validation or recovery

The script performs **both network I/O and database writes**, making it
**high-impact and non-optional** in the BOD flow.

---
### 4️⃣ Genuine Runtime Dependencies (Actual & Enforced)

This script has **hard, non-negotiable dependencies** that must be satisfied.

---

#### 4.1 External Network Dependencies

- Internet connectivity must be available
- NSE archive endpoint must be reachable:

[https://nsearchives.nseindia.com](https://nsearchives.nseindia.com)
- Host IP must not be blocked by NSE

The script depends on:
- A valid `User-Agent` header
- Direct HTTP GET access

If NSE blocks the request:
- HTTP 403 / 401 is returned
- Script fails immediately

---
#### 4.3 Database Dependencies (Hard Requirement)

The script performs **direct INSERT operations** into MySQL.

Required conditions:
- MySQL server must be running
- Database must be reachable
- Target database must exist
- Table `BannedInstruments` must exist
- DB user must have `INSERT` privilege

Expected table schema (minimum):
```sql
BannedInstruments(
Symbol VARCHAR(...),
Reason VARCHAR(...),
IsBanned BOOLEAN,
IsUserCreated BOOLEAN
)
If table or privileges are missing:

Script fails during insertion

No fallback logic exists
```

## 📄 Script Deep Dive: download_bse_contract_master.py

---

### 1️⃣ Purpose and Business Intent

`download_bse_contract_master.py` is a **reference data acquisition script**
used in the BOD workflow to download and prepare **BSE contract master files**
required for instrument mapping and downstream processing.

Its primary responsibility is to:
- Download **official contract reference files** from BSE
- Extract and normalize filenames into **fixed, predictable names**
- Ensure downstream scripts can consume BSE reference data
  without handling date-based filenames

This script does **not** parse or load data into the database.
It strictly prepares **clean, standardized input files**.

---

### 2️⃣ Authoritative Data Sources (From Where the Files Are Downloaded)

All data is downloaded **directly from official BSE-controlled endpoints**.

#### 📥 Source 1: SCRIP Master File

**URL:**
[https://www.bseindia.com/downloads/Help/file/scrip.zip](https://www.bseindia.com/downloads/Help/file/scrip.zip)


**Provided by:**  
:contentReference[oaicite:1]{index=1}

**Contents after extraction:**
- Folder: `SCRIP/`
- File pattern:
SCRIP_<ddMMyy>.TXT


This file contains **BSE instrument reference data**
(e.g. scrip codes, symbols, ISINs).

---

#### 📥 Source 2: EQD (Equity Details) File

**URL:**


[https://www.bseindia.com/downloads1/CO_BSE.zip](https://www.bseindia.com/downloads1/CO_BSE.zip)
**Contents after extraction:**
- File pattern:
EQD_CO<ddMMyy>.csv

yaml
Copy code

This file contains **BSE equity master information**
used for equity-level mapping.

---

### 3️⃣ Execution Context

The script is implemented as a **standalone CLI utility** and is executed:

- Automatically via `process_master.py` during BOD
- Manually for recovery, testing, or file regeneration

The script performs:
- External network downloads
- ZIP extraction
- File renaming and movement

It does **not**:
- Modify databases
- Touch Redis
- Depend on other BOD scripts

---

### 4️⃣ Date Resolution Logic (Important)

The script resolves filenames using:
current_date - 1 day

yaml
Copy code

Reason:
- BSE publishes contract files **with previous trading date**
- Running the script on the same day before file publication
  would result in missing files

⚠️ This introduces a **time dependency**:
- Script should be run **after BSE publishes files**
- Weekend / holiday execution may require manual verification

---

### 5️⃣ Failure Handling and Edge Cases

#### 5.1 BSE Endpoint Unavailable
- HTTP request fails
- Script exits with non-zero code
- No files prepared

---

#### 5.2 ZIP Downloaded but Not a Valid ZIP
- Script checks `zipfile.is_zipfile`
- If invalid, extraction is skipped
- Downstream file rename fails

---

#### 5.3 Date-Based File Not Found
- Expected dated file does not exist
- Rename step is skipped silently
- Output files (`SCRIP.TXT`, `EQD.csv`) are missing

This may break downstream consumers.

`download_bse_contract_master.py` prepares authoritative BSE contract reference files
by downloading them from official BSE endpoints, extracting them, and normalizing
their filenames for downstream consumption.  
Its behavior is highly dependent on BSE publication timing and file naming conventions,
and it plays a critical role in ensuring reliable BSE instrument reference data
for the entire BOD workflow.

## 📄 Script Deep Dive: download_contract_master.py

---

### 1️⃣ Purpose and Business Intent

`download_contract_master.py` is a **core reference-data ingestion script**
responsible for downloading the **NSE Contract Master files** required for
instrument definition, validation, and downstream market-data processing.

Its primary responsibilities are:
- Authenticate with NSE’s secure extranet API
- Download **compressed contract master files**
- Decompress and normalize them into usable text files
- Store raw compressed data in Redis for fast access and audit

This script is **foundational** for any system that relies on:
- Instrument master data
- Contract definitions
- Symbol–token–expiry mappings

> 🔒 If this script fails, **instrument-related processing must not proceed**.

---

### 2️⃣ Authoritative Data Source (From Where the Files Are Downloaded)

This script downloads data from **NSE’s official secure extranet APIs**.

**Provided by:**  
:contentReference[oaicite:1]{index=1}

---

#### 2.1 Base API Endpoints
Base URL : [https://www.connect2nse.com/extranet-api](https://www.connect2nse.com/extranet-api)

```Login API : /login/2.0
File Download API : /common/file/download/2.0


```


These endpoints are **authenticated**, **member-restricted**, and **not public**.

---

#### 2.2 Files Downloaded

The script downloads the following **compressed reference files**:

| File Name | Segment | Folder Path | Description |
|---------|--------|------------|-------------|
| `security.gz` | CM | `/ntneat` | Security master data |
| `contract.gz` | CM | `/ntneat` | Contract master data |

Both files are published and maintained by NSE and represent the
**authoritative contract definitions**.

---

### 3️⃣ Authentication Model (Critical Dependency)

Before any download, the script performs an **authenticated login**.

**Login API:**
POST [https://www.connect2nse.com/extranet-api/login/2.0](https://www.connect2nse.com/extranet-api/login/2.0)
**Required credentials:**
- `memberCode`
- `loginId`
- `encryptedPassword`

On successful login:
- NSE returns a **Bearer token**
- The token is stored in memory and used for all download requests

If authentication fails:
- No file download is attempted
- Script exits immediately

### 4️⃣ Execution Context

The script is implemented as a **standalone CLI utility** and is executed:

- Automatically via `process_master.py` during BOD
- Manually for validation, re-runs, or recovery

The script performs:
- Secure network calls
- File system writes
- GZIP decompression
- Redis writes

Because of this, it is considered a **high-impact infrastructure script**.

### 4️⃣ Dependencies (Critical, Genuine & Enforced)

`download_nse_most_active_scripts.py` is a **network- and cache-dependent script**.
It has no database dependency, but it is **highly sensitive to external services**.
Any failure in these dependencies directly impacts real-time data availability.

---

#### 4.1 NSE Public API Dependency (Primary Dependency)

The script depends entirely on the availability of the NSE public API endpoint:

[https://www.nseindia.com/api/live-analysis-variations](https://www.nseindia.com/api/live-analysis-variations)

Provided by:  
:contentReference[oaicite:0]{index=0}

This dependency has multiple implicit constraints:

- NSE must allow access from the host IP
- The API must be publicly reachable (no maintenance / outage)
- The API must return data in CSV format
- The API response structure must remain stable

If NSE:
- blocks the IP  
- changes the API path  
- removes CSV support  

then the script **cannot function**, regardless of internal correctness.


## 1.4 First Step: Writing a New File-Parsing Script

If you want to parse a **new file** as part of BOD, follow these steps.

---

### Step 1: Identify the File Clearly

Before writing any code, identify:
- File name pattern
- Fixed daily location
- Purpose of the file
- Whether other processes depend on it

This helps decide **where it fits in the BOD order**.

---

### Step 2: Create a New Script Inside `utils/`

All new file-parsing scripts must be added inside the `utils/` folder.

Example naming rules:
- Name should clearly describe the file
- One script should handle **one file type only**


![utils](https://raw.githubusercontent.com/priyanshu-quantxpress/MD-files/main/images/utils.png)


> New script added inside utils folder



### Step 3: What the Script Is Expected to Do

Every script inside `utils/` must follow these responsibilities:

1. Locate the file from its expected location  
2. Validate that the file exists  
3. Validate basic file correctness  
4. Process the file content  
5. Exit clearly with success or failure  

⚠️ Important rules:
- Script must be runnable on its own
- Script must not assume other scripts have already run unless explicitly ordered
- Script must fail clearly if the file is missing or invalid


### 6️⃣ Input Contract (CLI Arguments)

| Argument | Required | Description |
|------|--------|-------------|
| `--member_code` | ✅ | NSE member code |
| `--login_id` | ✅ | NSE login ID |
| `--encrypted_password` | ✅ | Encrypted NSE password |
| `--download_folder` | ❌ | Output folder (default: `downloads`) |

Missing any required argument causes **immediate script termination**.

`download_contract_master.py` securely downloads and prepares the NSE contract
and security master files using authenticated extranet APIs.
It enforces strict authentication, retry, and decompression logic while also
caching raw compressed data in Redis for performance and audit purposes.
The script is a critical dependency for all instrument-driven BOD processes
and must execute successfully for a valid trading day setup.

---


## 📄 Script Deep Dive: download_nse_most_active_scripts.py

---

### 1️⃣ Purpose and Business Intent

`download_nse_most_active_scripts.py` is a **real-time market snapshot ingestion script**
used to fetch **NSE “Most Active / Gainers” market data** and store it in Redis for
fast, in-memory consumption.

Its primary objective is to:
- Download live market CSV data from NSE APIs
- Store the raw CSV response in Redis
- Make the data instantly available for UI, analytics, or monitoring use cases

This script is **read-only with respect to NSE data** and **write-only with respect to Redis**.

> 🔒 This script does **not** modify databases  
> 🔒 It acts as a lightweight data feeder for real-time views

---

### 2️⃣ Authoritative Data Source (From Where the Data Is Downloaded)

The script fetches CSV data from an **official NSE public API endpoint**.

**Primary API Endpoint:**
[https://www.nseindia.com/api/live-analysis-variations?index=gainers&type=NIFTY&csv=true](https://www.nseindia.com/api/live-analysis-variations?index=gainers&type=NIFTY&csv=true)

yaml
Copy code

**Provided by:**  
:contentReference[oaicite:1]{index=1}

---

#### 📌 Nature of the Data

- Format: CSV
- Data Type: Live / near-real-time market snapshot
- Segment: Equity (NIFTY index gainers)
- Update Frequency: Intraday (frequently updated by NSE)

⚠️ This data is **not end-of-day (EOD)** data and should not be treated as historical truth.

---

### 3️⃣ Execution Context

The script is implemented as a **standalone command-line utility** and can be executed:

- Automatically as part of auxiliary BOD/EOD or intraday jobs
- Manually for cache refresh or troubleshooting

The script performs:
- One HTTP GET request to NSE
- One Redis `SET` operation

It is intentionally **stateless and lightweight**.


### 4️⃣ Runtime Dependencies (Actual & Enforced)
- NSE public API must be reachable:
[https://www.nseindia.com](https://www.nseindia.com)


#### 4.2 HTTP Header Dependency (Anti-Bot Enforcement)

NSE applies **anti-bot and traffic filtering rules**.

The script explicitly depends on:
- A valid `User-Agent` HTTP header

Without this:
- NSE responds with **HTTP 403 Forbidden**
- The request is rejected even though the endpoint is public

#### 4.3 Data Freshness Dependency (Time-Sensitive)

This script does **not validate timestamps** inside the CSV.

It implicitly assumes:
- NSE is serving **current intraday data**
- Redis overwrite equals data refresh

If the script does not run:
- Redis retains old data
- No automatic expiry or TTL is enforced

Operationally, this creates a dependency on:
- Proper scheduling
- Regular execution frequency

---
### 🔍 Dependency Risk Summary 

This script’s reliability depends more on **external systems** than internal logic:

- NSE availability & access rules
- Network stability
- Redis uptime
- Scheduled execution discipline

It is intentionally lightweight but **operationally sensitive**.
Failures do not block BOD, but they directly impact **real-time visibility** and
user-facing dashboards.

---



## 📄 Script : ftp_downloader.py

---

### 1️⃣ Purpose and Business Intent

`ftp_downloader.py` is a **generic file-ingestion utility script** designed to
download a **single file from a remote FTP server** and store it at a specified
local location.

Its primary responsibility is to:
- Establish a secure session with an FTP server
- Authenticate using provided credentials
- Download a specified remote file in binary-safe mode
- Persist the file locally for downstream BOD/EOD processing

This script is intentionally **generic and reusable** and does not contain
any exchange-specific or business-specific logic.

---

### 2️⃣ Execution Context

The script is implemented as a **standalone command-line utility** and is executed:

- As part of BOD/EOD workflows where data is delivered via FTP
- Manually by operations teams for recovery or ad-hoc downloads

The script performs:
- Network I/O (FTP protocol)
- File system writes

It does **not**:
- Modify databases
- Use Redis
- Depend on other BOD scripts

---

### 3️⃣  Runtime Dependencies (Critical Focus)

This script is **entirely dependency-driven**.
Its success depends more on **external infrastructure** than on internal logic.

---

#### 3.1 FTP Server Dependency (Primary Dependency)

The script depends on the availability and behavior of an **external FTP server**.

Required conditions:
- FTP server must be reachable on the network
- FTP service must be running
- FTP server must support standard `RETR` command
- Remote file path must exist exactly as provided

If the FTP server:
- is down
- rejects connections
- changes directory structure
- removes the file

the script **cannot recover or retry automatically**.

---

#### 3.2 FTP Authentication Dependency

The script uses **username + password authentication**.

Hard requirements:
- Credentials must be valid
- User must have permission to:
  - access the remote directory
  - retrieve the specified file

If authentication fails:
- FTP login is rejected
- Script exits without downloading the file

No anonymous or key-based authentication is supported.

---


#### 3.3 File System Dependencies

Local file system requirements:
- Write permission on the target directory
- Ability to create directories recursively
- Sufficient disk space for the downloaded file

Behavior enforced by code:
- Parent directory is auto-created if missing
- Script exits immediately if directory creation fails

If disk space is exhausted or permissions are missing:
- File write fails
- Download is aborted

---
### 4️⃣ Input Contract (CLI Arguments)

The script enforces **explicit input parameters**.

| Argument | Required | Description |
|------|--------|-------------|
| `--host` | ✅ | FTP server hostname or IP |
| `--user` | ✅ | FTP username |
| `--password` | ✅ | FTP password |
| `--remote-file` | ✅ | Absolute path of remote file |
| `--local-file` | ✅ | Absolute path for local save |

Missing any argument results in **immediate termination**.


### 5️⃣ Edge Cases and Failure Scenarios

#### 5.1 Remote File Does Not Exist
- FTP server returns error
- Script logs FTP error
- No local file is created

---

#### 5.2 Partial Download / Connection Drop
- FTP session terminates mid-transfer
- Local file may be incomplete
- No checksum or validation is performed

Operational cleanup may be required.

---



#### 5.3 Incorrect Local Path
- Directory creation fails
- Script exits immediately using `sys.exit(1)`

---

#### 5.4 Invalid Credentials
- Login fails
- Script exits before file transfer begins

---
### 🔚 Summary 

`ftp_downloader.py` is a lightweight file-ingestion utility whose successful
execution is **highly dependent on external systems rather than internal logic**.
Its reliability is primarily governed by:

- Availability and responsiveness of the remote FTP server  
- Validity and permissions of the provided FTP credentials  
- Stability of the underlying network connection  
- Write access and available space on the local file system  

The script intentionally avoids advanced mechanisms such as retries, resume
support, or file validation in order to keep its behavior **simple, predictable,
and transparent**.

Because of this design choice, the script requires **active operational monitoring**
when used in automated BOD/EOD workflows to ensure that external dependencies
are healthy and that any failures are detected and addressed promptly.

## 📄 Script Deep Dive: nse_span_downloader.py

---

### 1️⃣ Purpose and Business Intent

`nse_span_downloader.py` is a **risk-critical reference data ingestion script**
responsible for downloading **NSE SPAN margin files** and preparing them for
downstream margin calculation and risk-management processes.

SPAN (Standard Portfolio Analysis of Risk) files are used to:
- Calculate margin requirements
- Evaluate portfolio risk
- Enforce exchange-mandated risk controls

> 🔒 If this script fails or consumes incorrect data,  
> **margin calculations may be wrong**, leading to serious risk exposure.

---

### 2️⃣ Authoritative Data Source (From Where the Files Are Downloaded)

All SPAN files are downloaded **directly from NSE’s official archive**.

**Primary source domain:**
https://nsearchives.nseindia.com

css
Copy code

**SPAN directory:**
[https://nsearchives.nseindia.com/archives/nsccl/span/](https://nsearchives.nseindia.com/archives/nsccl/span/)



**Published by:**  
:contentReference[oaicite:0]{index=0}

---

#### 2.1 SPAN File Naming Convention

For a given trading date (`YYYYMMDD`), the script attempts to download files
in the following **priority order**:
```
nsccl.YYYYMMDD.i5.zip
nsccl.YYYYMMDD.i4.zip
nsccl.YYYYMMDD.i3.zip
nsccl.YYYYMMDD.i2.zip
nsccl.YYYYMMDD.i1.zip
nsccl.YYYYMMDD.s.zip
```


Meaning:
- `i1`–`i5` → Intraday SPAN snapshots
- `s` → Final settlement SPAN file

The script **stops at the first successfully downloaded file**.

---

### 3️⃣ Execution Context

The script is implemented as a **standalone CLI utility** and is executed:

- Automatically as part of the BOD / pre-risk setup
- Manually by operations teams when SPAN publication is delayed

The script performs:
- External HTTP downloads
- ZIP validation and extraction
- File renaming and normalization

It does **not**:
- Write to the database
- Write to Redis
- Perform margin calculations itself

---

### 4️⃣ Input Contract (CLI Arguments)

| Argument | Required | Description |
|------|--------|-------------|
| `--date` | ❌ | Trading date in `YYYYMMDD` format (defaults to current date) |
| `--output-dir` | ❌ | Local directory to store SPAN files (default: `downloads/`) |

If no date is provided, the script assumes **current system date**.

### 5️⃣ Genuine Runtime Dependencies (Critical & Enforced)

This script is **heavily dependent on external systems**.

---

#### 5.1 NSE Archive Availability (Primary Dependency)

- NSE archive must be reachable
- SPAN files must be published for the given date
- File naming conventions must remain unchanged

If NSE:
- delays publication
- skips intraday snapshots
- changes filenames  

the script **cannot recover automatically**.

---

#### 5.2 NSE Publication Timing Dependency

SPAN files are **time-sensitive**.

Operational realities:
- Intraday files (`i*`) may not exist on holidays
- Settlement file (`s`) may be delayed
- Early execution can result in HTTP 404

Correct scheduling is **mandatory**.

---

#### 5.3 File System Dependencies

Local requirements:
- Write permission in output directory
- Sufficient disk space
- Ability to extract ZIP archives

Failure in any step results in script termination.

---

#### 5.4 ZIP Structure Dependency

The script assumes:
- ZIP contains at least one file
- The first file is the SPAN data file

Any structural change in ZIP content may break extraction logic.

---


### 🔚 Summary (Manager / Risk View)

`nse_span_downloader.py` ingests official NSE-published SPAN margin files
and prepares them for downstream risk and margin systems.
Its execution is highly sensitive to NSE publication timing, archive availability,
and network stability.  
Because SPAN data directly affects margin computation, this script is
**risk-critical** and must be scheduled, monitored, and validated carefully
during every BOD cycle.

---


## 📄 Script Deep Dive: process_monitor.py

---

### 1️⃣ Purpose and Business Intent

`process_monitor.py` is a **continuous runtime monitoring and alerting script**
responsible for tracking the health and resource usage of **critical system
processes and Docker containers** defined in a configuration file.

Its primary objectives are to:
- Monitor whether critical processes are **running or stopped**
- Capture **CPU, memory, disk I/O, PID, and start time**
- Persist time-series process metrics into **Redis**
- Trigger **email alerts** when a monitored process is not running

This script is designed to run **continuously in the background**
and acts as an **operational safety net** for BOD/EOD and long-running services.

> 🔒 If this script is not running,  
> **process failures may go undetected**, leading to silent system outages.

---

### 2️⃣ What Processes Are Monitored (Source of Truth)

The list of processes to monitor is defined in:

**Processes.json**


This file must contain a key:

```json
"process_monitor": [
  {
    "title": "Readable Name",
    "process": "actual_process_or_container_name"
  }
]

```

3️⃣ Execution Context

**The script runs as a background monitoring thread.**

Execution characteristics:

  -Uses threading.Thread

  -Runs in a continuous loop

  -Sleeps for 5 seconds between monitoring cycles

  -Starts and stops via:

  -start_monitoring()

  -stop_monitoring()

  -It is intended to run:

  -Alongside the Process Manager

  -As a long-running service compone




## 1.4 Registering the New Script in `process.json` 

This section explains **how file paths and locations are defined inside `process.json`**
and **why paying attention to paths is the most critical part of adding a new BOD process**.

Most BOD failures do **not** happen because of script logic.
They happen because of **incorrect paths, wrong folders, or misunderstood locations**.

---

## What is `process.json`?

`process.json` is the **main configuration file** that defines the complete daily process flow.

It tells the system:
- which processes exist
- which scripts should be executed
- whether a process is BOD or EOD
- the order in which processes run
- where scripts are located
- where input files are expected
- where output files should be written

If a script or file path is **not defined here**, it will **never run**.

---

## Understanding `base_path`

At the top of `process.json`, there is a field called `base_path`.

Example:

  `base_path: C:\Users\Navin Kumar\Desktop\QXT\1 New Blitz Projects\Backend\QX.ProcessManager\utils`

  ### What `base_path` means
- It is the **common folder** where most processing scripts are stored
- It avoids repeating long folder paths again and again

### When to use `base_path`
Use `${base_path}` when:
- your script is placed inside the `utils` folder
- your script is part of the regular BOD or EOD flow

Example:

`"application_path": "${base_path}\copy_files_src_dst.py"`




This means:
> Run `copy_files_src_dst.py` from the `utils` folder.

---

## Understanding `application_path`

`application_path` defines **exactly which script will be executed**.

Examples:
`"${base_path}\download_nse_most_active_scripts.py"`


### Rules for `application_path`
- It must be a **full absolute path**
- It must point to **one script file**
- The script must exist at that location
- File name and folder name must match exactly

If `application_path` is wrong:
- the script will not start
- the process will fail immediately

---

## Where Input Files Should Be Placed

Input file locations are usually passed inside the `parameter` section.

Example:

`"parameter": {
"source":"C:\Users\Navin Kumar\Desktop\QXT\Backend\QX.ProcessManager\downloads\nsccl.20250627*"
}`


### What this means
- The script will look for input files in this folder
- File names must match the given pattern
- Files must already exist before the process runs

### Rules for input file paths
- Use a **fixed folder** for incoming files
- Files should arrive in the same location every day
- File name format should be consistent
- Do not change paths daily

---

## Destination Paths (Output Files)

Some scripts copy or generate files and need a destination folder.

Example:
`
"destination": [
"C:\Users\Navin Kumar\Desktop\QXT\BOD Project\"
]
`

### Rules for destination paths
- The folder must exist or be created by the script
- Use separate folders for output files
- Do not overwrite unrelated files

---

## Path Format Rules (Very Important)

Always follow these rules while writing paths:

- Use **absolute paths only**
- Use double backslashes `\\`
- Do not use relative paths
- Do not add extra spaces
- Avoid temporary folders unless required

Correct:
`C:\\Users\\Navin Kumar\\Deskto\\QXT\\Backend\\QX.ProcessManager\\downloads`



---

## Next Step: Verifying the New Process from the UI

After adding or updating a process, verification is mandatory.

---

### Step 1: Restart the System

Changes in `process.json` are loaded only at startup.

- Stop the running system
- Start it again using the main runner (run `process_master.py`)

If this step is skipped, the new process will not appear.

---

### Step 2: Open the UI

- Open the Admin UI
- Navigate to Monitor Section 
- Navigate to the BOD/EOD section

You should see the new process listed.

![AdminUI](https://raw.githubusercontent.com/priyanshu-quantxpress/MD-files/main/images/AdminUI.png)

> UI showing the BOD process list

---

### Step 3: Verify the Process Details

Confirm:
- Process name is correct
- Execution order is correct
- Process is marked active

If anything looks wrong, stop and fix `process.json`.

---

### Step 4: Run BOD and Observe

- Trigger the BOD run
- Watch the process execution
- Ensure the process runs at the expected time
- Confirm it completes successfully

![RunBOD_EOD](https://raw.githubusercontent.com/priyanshu-quantxpress/MD-files/main/images/RunBOD_EOD.png)


> UI showing successful execution of the new process

---

### Step 5: If the Process Fails

If the process fails:
1. Check file availability
2. Verify input paths
3. Review logs
4. Fix the issue
5. Restart the system
6. Run BOD again

---

## Final Summary of the Flow



```
Script Added
↓
Registered in process.json
↓
System Restarted
↓
Process Appears in UI
↓
BOD Run Triggered
↓
Process Executed

```


Following this flow ensures:
- Safe execution
- Predictable behavior
- Easy troubleshooting
