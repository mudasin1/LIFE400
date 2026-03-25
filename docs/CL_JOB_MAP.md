# LIFE400 CL Job Map

**System:** LIFE400 — ACME Life Insurance Co.
**Platform:** IBM AS/400 (IBM i), OS/400 V4R2
**Purpose:** Maps every ILE CL program to the COBOL program(s) it invokes.
Provided as supplemental documentation for tools (such as Swimm) that expect JCL-style job definitions.

---

## Background: CL vs JCL

LIFE400 runs on IBM AS/400. It uses **ILE Control Language (CL)** for job control — the AS/400 equivalent of JCL on IBM z/OS mainframes. CL programs are stored in `QCLSRC/` with the `.clle` extension. They perform the same role as JCL: submitting batch jobs, setting up the library list, and managing database file overrides.

---

## Interactive Session Entry Point

### STRTLIFE.clle → MAINMENU.cbl

| Property | Value |
|----------|-------|
| Source | `QCLSRC/STRTLIFE.clle` |
| Invocation | Set as user profile initial program: `CHGUSRPRF INLPGM(LIFE400/STRTLIFE)` |
| Execution type | Interactive (foreground) |
| COBOL program called | `LIFE400/MAINMENU` |
| Parameters passed | None |

**What it does:**
1. Adds `LIFE400` library to job library list (ignores error if already present)
2. Sends an informational message to `LIFE400/LIFEMSGQ`
3. Calls `MAINMENU` — the interactive COBOL menu program
4. On return (user signs off), removes `LIFE400` from library list

---

## Interactive COBOL Menu Router

### MAINMENU.cbl

| Property | Value |
|----------|-------|
| Source | `QCBLLESRC/MAINMENU.cbl` |
| Called by | `STRTLIFE.clle` |
| Display file | `MNUDSPF` (5250 terminal screen) |

**Menu options and their COBOL targets:**

| Selection | COBOL Program | Description |
|-----------|--------------|-------------|
| 1 | `NBUWMNT.cbl` | New Business & Underwriting — online maintenance |
| 2 | `SVCMNT.cbl` | Policy Servicing — online maintenance |
| 3 | `CLMMNT.cbl` | Claims — online maintenance |
| 4 | `POLMSTINQ.cbl` | Policy Master Inquiry — read-only |
| 5 | *(not implemented)* | Reports menu — stub only |
| 90 | *(sign off)* | Exits the menu loop |

Online programs (`NBUWMNT`, `SVCMNT`, `CLMMNT`) stage records in the database and then invoke their corresponding batch CL programs to submit asynchronous processing jobs.

---

## Batch Job Submission CL Programs

### RUNNBUW.clle → NBUWB.cbl

| Property | Value |
|----------|-------|
| Source | `QCLSRC/RUNNBUW.clle` |
| Invocation | Called from `NBUWMNT` after staging a policy application, or manually via `CALL RUNNBUW PARM('POLTEST00001')` |
| Execution type | Submits batch job asynchronously |
| Batch job name | `NBUWBAT` |
| Job description | `LIFE400/LIFEJD` |
| Job queue | `LIFE400/LIFEQ` |
| Output queue | `LIFE400/LIFEOUTQ` |
| Message queue | `LIFE400/LIFEMSGQ` |
| COBOL program called | `LIFE400/NBUWB` |

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `&POLID` | CHAR(12) | Policy ID to process — must not be blank |

**Database files overridden (OVRDBF):**

| File | Override target |
|------|----------------|
| POLMST | LIFE400/POLMST |
| CLMPF | LIFE400/CLMPF |

**What NBUWB.cbl does:** Validates application eligibility, loads plan parameters, determines underwriting class, calculates premium, validates riders, and issues/refers/declines the policy. Return codes 00–26 (see `QCBLLESRC/NBUWB.cbl` header).

---

### RUNCLM.clle → CLMADJB.cbl

| Property | Value |
|----------|-------|
| Source | `QCLSRC/RUNCLM.clle` |
| Invocation | Called from `CLMMNT` after staging a claim, or manually via `CALL RUNCLM PARM('CLM000000001' 'POLTEST00001')` |
| Execution type | Submits batch job asynchronously |
| Batch job name | `CLMBAT` |
| Job description | `LIFE400/LIFEJD` |
| Job queue | `LIFE400/LIFEQ` |
| Output queue | `LIFE400/LIFEOUTQ` |
| Message queue | `LIFE400/LIFEMSGQ` |
| COBOL program called | `LIFE400/CLMADJB` |

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `&CLMID` | CHAR(12) | Claim ID to adjudicate — must not be blank |
| `&POLID` | CHAR(12) | Associated policy ID — must not be blank |

**Database files overridden (OVRDBF):**

| File | Override target |
|------|----------------|
| POLMST | LIFE400/POLMST |
| CLMPF | LIFE400/CLMPF |

**What CLMADJB.cbl does:** Validates claim intake, identifies investigation triggers (contestability window, suicide clause), adjudicates coverage, and calculates net settlement amount. Return codes 00–22 (see `QCBLLESRC/CLMADJB.cbl` header).

---

### RUNSVC.clle → SVCBILB.cbl

| Property | Value |
|----------|-------|
| Source | `QCLSRC/RUNSVC.clle` |
| Invocation | Called from `SVCMNT` after staging a service request, or manually via `CALL RUNSVC PARM('POLTEST00001' 'SVC000000001')` |
| Execution type | Submits batch job asynchronously |
| Batch job name | `SVCBAT` |
| Job description | `LIFE400/LIFEJD` |
| Job queue | `LIFE400/LIFEQ` |
| Output queue | `LIFE400/LIFEOUTQ` |
| Message queue | `LIFE400/LIFEMSGQ` |
| COBOL program called | `LIFE400/SVCBILB` |

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `&POLID` | CHAR(12) | Policy ID to service — must not be blank |
| `&SVCID` | CHAR(12) | Service request ID — must not be blank |

**Database files overridden (OVRDBF):**

| File | Override target |
|------|----------------|
| POLMST | LIFE400/POLMST |
| SVCPF | LIFE400/SVCPF |

**What SVCBILB.cbl does:** Re-prices policy changes, controls status transitions (active → grace → lapsed → reinstated), processes amendment types (plan change, sum assured change, billing mode change, rider add/remove, reinstatement), and calculates amendment fees. Return codes 00–33 (see `QCBLLESRC/SVCBILB.cbl` header).

---

## Scheduled Nightly Batch

### DLYUPD.clle → SVCBILB.cbl (sweep mode)

| Property | Value |
|----------|-------|
| Source | `QCLSRC/DLYUPD.clle` |
| Invocation | OS/400 job scheduler: `ADDJOBSCDE JOB(DLYUPD) CMD(CALL LIFE400/DLYUPD) FRQ(*WEEKLY) SCDDAY(*ALL) SCDTIME(233000)` |
| Execution type | Scheduled batch (runs nightly at 23:30 in QBATCH subsystem) |
| Job description | `LIFE400/LIFEJD` |
| COBOL program called | `LIFE400/SVCBILB` (direct `CALL`, not via SBMJOB) |

**Special parameters (sweep mode):**

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `&POLID` | `*SWEEP     ` | Signals SVCBILB to run in portfolio sweep mode |
| `&SVCID` | `*DLYUPD    ` | Identifies the nightly job as the caller |

**What it does:**
1. Retrieves the current system date from `QDATE` sysval
2. Adds `LIFE400` to library list and overrides POLMST and SVCPF
3. Calls `SVCBILB` with `*SWEEP` to evaluate all policies for grace/lapse transitions (policies where PAIDTO < current date transition: ACTIVE → GRACE → LAPSED)
4. Sends completion summary to `QSYSOPR` and `LIFE400/LIFEMSGQ`

**Database files overridden (OVRDBF):**

| File | Override target |
|------|----------------|
| POLMST | LIFE400/POLMST |
| SVCPF | LIFE400/SVCPF |

---

## Summary: CL → COBOL Call Graph

```
STRTLIFE.clle
└── MAINMENU.cbl (interactive)
    ├── NBUWMNT.cbl (interactive) ──> RUNNBUW.clle ──[SBMJOB NBUWBAT]──> NBUWB.cbl
    ├── SVCMNT.cbl  (interactive) ──> RUNSVC.clle  ──[SBMJOB SVCBAT] ──> SVCBILB.cbl
    ├── CLMMNT.cbl  (interactive) ──> RUNCLM.clle  ──[SBMJOB CLMBAT] ──> CLMADJB.cbl
    └── POLMSTINQ.cbl (interactive, read-only)

DLYUPD.clle (scheduled 23:30 nightly)
└── SVCBILB.cbl (direct CALL, sweep mode)
```

---

## Database Files Accessed Per Batch Job

| Batch Job | CL Submitter | COBOL Program | POLMST | CLMPF | SVCPF |
|-----------|-------------|---------------|--------|-------|-------|
| NBUWBAT | RUNNBUW.clle | NBUWB.cbl | I-O | — | — |
| CLMBAT | RUNCLM.clle | CLMADJB.cbl | I-O | I-O | — |
| SVCBAT | RUNSVC.clle | SVCBILB.cbl | I-O | — | I-O |
| DLYUPD | DLYUPD.clle | SVCBILB.cbl | I-O | — | I-O |

DB2 DDL equivalents for POLMST, CLMPF, and SVCPF are in `QSQLSRC/LIFE400DDL.sql`.
