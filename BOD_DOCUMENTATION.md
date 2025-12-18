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






## 1.3 First Step: Writing a New File-Parsing Script

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


![Process Master File](images/utils.png)

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

📸 **Screenshot Placeholder**  
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

📸 **Screenshot Placeholder**  
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



