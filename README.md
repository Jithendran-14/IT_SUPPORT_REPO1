# Repo 1 - IAT Ingestion SDK workshop

Repair three intentional SDK integration defects, then ingest four approved
IT support documents into local Oracle using `qwen3-embedding:0.6b`.
No retrieval, agent, service, frontend or evaluation implementation is needed
in this repo. `qwen3:4b` is used in the later agent exercise, not ingestion.

## What you will learn

`Named source -> Loader -> Parser -> Chunker -> Indexer -> Oracle`

- **Source and loader:** IAT selects the LocalStorage configuration by name.
- **Parser (supplied working):** `ReviewedJSONParser` turns approved cleaned
  page JSON into IAT `ParsedDocument` / `DocumentStructure` objects. It checks
  hashes and retains original PDF pages and source references.
- **Chunker:** the SDK's recursive text chunker uses the configured size,
  overlap and separators.
- **Indexer:** `with_indexer()` attaches the supplied `LocalOracleIndexer`.
  This application adapter replaces the default OCI embedding client with
  `LocalQwenEmbeddings`; native IAT indexing still performs Oracle inserts.
- **Pipeline:** the supplied assembly function is used to configure the
  actual `HelpdeskIngestionPipeline`, then `.build()` creates the toolkit.

The original PDFs remain in the repo because their hashes are checked, even
though ingestion reads the cleaned JSON. Do not re-clean the documents.

## Prerequisites and shared environment

Use Windows PowerShell in VS Code, Python 3.12, local Ollama, an Oracle
Database with VECTOR support, and access to Oracle's internal Python registry
(corporate network/VPN and registry access as required by your organization).
The four SDK versions are pinned to the working application baseline:

| SDK | Version |
| --- | --- |
| iat-agent-sdk | 2.6.3 |
| iat-rag-ingestion-sdk | 3.0.6 |
| iat-rag-retrieval-sdk | 1.1.22 |
| iat-service-sdk | 1.1.26 |

**No virtual environment is distributed.** Create one during this first
exercise, outside every repository, and reuse it for Repos 2-4. The supplied
`requirements.txt` is the full pinned Windows dependency baseline from the
working application, including the later SDKs. Install it once.

```powershell
$workshopRoot = Join-Path $env:USERPROFILE "Desktop\IAT_Workshop"
New-Item -ItemType Directory -Path $workshopRoot -Force
py -3.12 -m venv (Join-Path $workshopRoot ".venv")
& (Join-Path $workshopRoot ".venv\Scripts\Activate.ps1")

Set-Location -LiteralPath (Join-Path $env:USERPROFILE "Desktop\IT_SUPPORT_REPO1")
python -m pip install --index-url "https://artifactory.oci.oraclecorp.com/api/pypi/global-release-pypi/simple" -r requirements.txt
python -m pip check
```

If the organizer gives you a different extraction path, change only the
`Set-Location` path. In VS Code use **Ctrl+Shift+P -> Python: Select Interpreter
-> Enter interpreter path** and select the shared `.venv\Scripts\python.exe`.
Activate the same environment in each new terminal. Do not recreate it for
each repo. If PowerShell blocks activation, follow your organization's
execution-policy guidance or call that interpreter by its full path.

The environment is the same across repos; credentials remain environment
variables in the terminal running the exercise, not stored in the venv.

## Local model

Start Ollama. Check existing models before downloading anything:

```powershell
ollama list
```

If missing, obtain the embedding model:

```powershell
ollama pull qwen3-embedding:0.6b
```

The adapter uses `http://localhost:11434/api/embed` and validates finite,
nonzero 1024-dimensional vectors. You may set `OLLAMA_URL` if the organizer
uses another local address. Do not change model or embedding dimensions to
solve this exercise. In Repo 3 use the organizer's tested qwen3:4b setup.

## Oracle setup - a separate workshop schema

Use `IT_SUPPORT_WORKSHOP` in `FREEPDB1`. Do not use the main application's
`IT_SUPPORT_APP` user: the scripts explicitly reject it before connecting.
This isolates the workshop from the full application. The scripts do not
drop, truncate or alter existing tables.

If this workshop user does not exist, connect an administrative SQL client
to **FREEPDB1** (not CDB$ROOT) and run the following. Replace the password
placeholder with your own chosen password; do not share or commit it.

```sql
CREATE USER IT_SUPPORT_WORKSHOP
    IDENTIFIED BY "<choose-your-workshop-password>"
    DEFAULT TABLESPACE USERS
    TEMPORARY TABLESPACE TEMP
    QUOTA 100M ON USERS;

GRANT CREATE SESSION, CREATE TABLE, CREATE SEQUENCE TO IT_SUPPORT_WORKSHOP;
```

The sequence privilege supports the identity column in this Oracle setup.
If the user already exists, use its existing credentials; do not recreate it.

Set these in the activated terminal used for the exercise:

```powershell
$env:IAT_IT_USER = "IT_SUPPORT_WORKSHOP"
$env:IAT_IT_DSN = "localhost:1521/FREEPDB1"
$securePassword = Read-Host "IT_SUPPORT_WORKSHOP password" -AsSecureString
$env:IAT_IT_PASSWORD = [System.Net.NetworkCredential]::new("", $securePassword).Password
Remove-Variable securePassword

python .\create_ingestion_table.py
```

Change the DSN host/port only if your local Oracle instance uses another
address. Expected initial result: workshop user, `FREEPDB1`, destination
`IT_SUPPORT_EMBEDDINGS`, and zero stored rows.

## The challenge

There are exactly three intentional defects:

| ID | Component | Repair location |
| --- | --- | --- |
| R1-1 | Source selection / loader | `assemble_pipeline()` in `run_ingestion.py` |
| R1-2 | Recursive chunking policy | `chunking.strategies.text` in the YAML |
| R1-3 | Indexer attachment | R1-3 marker in `assemble_pipeline()` |

Required text policy: **recursive chunking, 800 characters per chunk,
100 characters of overlap**, with the supplied separators.

Only repair these integration points. Do not change SDK versions, adapters,
approved source files or verifier expectations. `HINTS.md` has progressive
concept and location hints. Setup errors are not additional contest defects.

### 1. Check your fixes without touching Oracle or Ollama

```powershell
python -m ingestion_pipeline.run_ingestion --check
```

Initially all three challenge checks should report `FAIL`, exit nonzero,
and point to the corresponding hints. The check uses an offline builder
probe, then (after fixes) the real SDK parser/chunker for a 16-chunk preview.
It does not connect to the database, generate embeddings or insert documents.
Passing `--check` alone does **not** earn the completion message.

### 2. Run the real ingestion after all checks pass

```powershell
python -m ingestion_pipeline.run_ingestion
```

With an empty workshop table this runs the actual IAT loader, parser,
chunker and local Oracle indexer. It calls the local embedding model and
writes the resulting chunks and vectors to the workshop table.

If records already exist, the runner verifies them without inserting
duplicates. Incomplete or unexpected data fails verification and is left
unchanged; inspect it with the organizer before attempting another import.

### 3. Verify again without inserting documents

```powershell
python -m ingestion_pipeline.run_ingestion --verify
```

This still requires the three exercise fixes. It reads the existing workshop
records and does not generate embeddings or write document rows.

## Expected verified result

```text
Database verification passed.
MS_WINDOWS_VPN_001: 8 chunks
MS_WINDOWS_INSTALL_001: 3 chunks
MS_STORE_APPS_001: 2 chunks
MS_WORK_ACCOUNT_001: 3 chunks
Total stored chunks: 16
Vector dimensions: 1024

Congratulations! You completed Repo 1: IAT Ingestion SDK.
```

Success requires actual stored records: exact unique chunk text, source
metadata and 1024-dimensional finite nonzero vectors. The terminal then
explains each defect, its SDK component, the effect of the correction and
the evidence that passed. Feedback is executable code in `run_ingestion.py`;
there is no separate feedback Markdown file.

## Reuse for later repos

Keep the shared venv and workshop schema/data. Repos 2-4 must use this same
compatible baseline and workshop account; do not point them at the main
application's schema. Each later repo will provide completed earlier stages.
Never commit credentials or a virtual environment.
