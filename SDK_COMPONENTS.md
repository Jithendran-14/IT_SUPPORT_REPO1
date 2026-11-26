# Repo 1: IAT Ingestion SDK components

## Purpose and scope

This guide explains the components used by this repository, the data each
component receives and produces, and how the supplied application adapters
connect the SDK to local Ollama and Oracle.

Repo 1 uses **IAT Ingestion SDK 3.0.6**. The shared environment also installs
the Agent, Retrieval and Service SDKs for the later exercises; installing
them does not mean this repository runs those SDK workflows.

Use [README.md](README.md) for setup and execution, and
[HINTS.md](HINTS.md) for progressive exercise hints. This guide explains the
components without providing a complete repaired pipeline.

## 1. The ingestion flow

```text
ingestion_config.yaml
        |
        v
ConfigManager / AppConfig
        |
        v
HelpdeskIngestionPipeline (application subclass of IATIngestionPipeline)
        |
        +-- LocalStorageLoader --> RawDocument
        |
        +-- ReviewedJSONParser --> ParsedDocument / DocumentStructure
        |
        +-- ChunkProcessor ----> DocumentChunk objects
        |
        +-- LocalOracleIndexer
                 |
                 +-- LocalQwenEmbeddings --> Ollama /api/embed
                 |
                 +-- SDK Oracle indexing --> Oracle vector table
```

The pipeline's builder attaches the components. After construction,
`RAGToolkit.process_documents_in_batches()` runs their work through SDK
queues. Text, document identity and source metadata travel together.

## 2. Configuration: ConfigManager and AppConfig

**Provided by the SDK:** `ConfigManager` loads configuration and exposes
component settings; `AppConfig` describes and validates the configuration
structure. Full configuration validation can register database settings and
validate connectivity, so it is different from simply reading a YAML file.

**Used in this repo:**
[ingestion_config.yaml](ingestion_pipeline/ingestion_config.yaml)
defines the inputs and processing policy.

| Configuration section | Role in this repository |
| --- | --- |
| `ingestion.sources` | Declares named input sources, their storage type, directory and file extensions. |
| `parser` | Configures parser processing; optional partitioning, normalization and contextual embedding are disabled in this small JSON-based exercise. |
| `chunking` | Selects chunking strategies, text size, overlap and separators. |
| `indexer` | Selects the vector-store database configuration, indexing batch size and vector precision. |
| `databases` | Describes the Oracle destination and how credentials are resolved. |
| `async_config` | Configures queue capacity and loader concurrency. |
| `logging` | Controls SDK logging for this run. |

The `vector_store` value refers to a key in `databases`; it is not the table
name. The selected database configuration contains `embeddings_table_name`.
With `secret_store: env_var`, credential fields refer to environment-variable
names. Password values do not belong in the repository.

**Why this matters:** source names, component settings and database labels
form a configuration contract. A component cannot reliably use settings that
were declared under a different name.

## 3. Pipeline assembly: IATIngestionPipeline and RAGToolkit

**Provided by the SDK:** `IATIngestionPipeline` is the builder used to attach
the ingestion components. Its main methods include:

| Method | Component attached or result produced |
| --- | --- |
| `with_loader(...)` | Creates the loader for a selected source. |
| `with_parser()` | Attaches parser processing. |
| `with_chunker()` | Attaches `ChunkProcessor`. |
| `with_indexer()` | Attaches the indexing component. |
| `build()` | Returns a `RAGToolkit` containing the attached components. |

`RAGToolkit` runs the loader, parser, chunker and indexer tasks, transfers data
between their queues and cleans up resources after processing.

**Application code:**
[HelpdeskIngestionPipeline](ingestion_pipeline/helpdesk_pipeline.py)
subclasses the SDK builder. It selects the supplied reviewed-JSON parser and
local Oracle indexer. The repository's `assemble_pipeline()` function in
[run_ingestion.py](ingestion_pipeline/run_ingestion.py) is where the exercise
connects the components. Normal ingestion passes a real SDK pipeline into
that function.

**Why this matters:** preparing text and persisting vectors are different
stages. A pipeline must contain the components needed for its intended
output. A successful document read alone is not proof of successful indexing.

## 4. Loading: LocalStorageLoader and RawDocument

**Provided by the SDK:** the builder uses `LoaderFactory` to select a loader
from the source configuration. This repository uses `LocalStorageLoader`
because its approved inputs are local files.

**Input:** the selected source's directory and file-extension configuration.

**Output:** `RawDocument` objects containing file content, its format, an ID
and metadata such as the file path.

The configured inputs are the four files under
[`knowledge-base/cleaned/`](knowledge-base/cleaned/). The loader reads these
cleaned JSON files. Their contents were prepared before this exercise.

The original PDFs are also supplied because the application parser verifies
their hashes against the source register. The current ingestion path does
not re-extract the PDF text.

**Why this matters:** the loader identifies and reads the intended inputs.
It does not decide whether a passage is relevant to a future question or
generate embeddings.

## 5. Parsing: ParserProcessor and the reviewed-JSON adapter

**Provided by the SDK:** `ParserProcessor` supplies parser-stage processing.
`ParsedDocument` and `DocumentStructure` are SDK data models used by later
stages.

**Application code:** `ReviewedJSONParser` subclasses `ParserProcessor` and
overrides the parsing operation. Its helper, `parse_reviewed_document()`,
understands this workshop's cleaned-page JSON format.

For each document, the helper:

1. Finds the matching entry in `sources.json`.
2. Checks its approval status and ingestion hold.
3. Verifies the original PDF and cleaned JSON hashes.
4. Checks document identity, source URL and original-page mapping.
5. Creates a text `DocumentStructure` for each reviewed page.
6. Returns the structures in a `ParsedDocument`.

The original page is retained in structure metadata as `page_number`.
For example, original PDF page 4 remains page 4 even when it is not the
fourth page in the cleaned file.

**Why this matters:** the parser gives the chunker structured text and its
provenance. Document IDs and original pages are essential for citations in
the later retrieval and agent exercises.

The approval and hash checks are **application validation**, not a claim
that the SDK automatically understands this particular source-register format.

## 6. Chunking: ChunkProcessor, ChunkerFactory and RecursiveChunker

**Provided by the SDK:**

- `ChunkProcessor` coordinates chunking of parsed structures.
- `ChunkerFactory` selects a chunker for the content type and strategy.
- `RecursiveChunker` splits text using the configured size, overlap and
  separators, producing SDK `DocumentChunk` objects.

**Input:** the text structures in a `ParsedDocument`.

**Output:** chunks containing text and metadata inherited from the source
structure, including its document ID, original page and source reference.

Chunk size controls the maximum text-unit size for this character-based
strategy. Overlap repeats some text between neighbouring chunks to preserve
context near boundaries. The required policy is specified in README.md.

A chunk is not a PDF page: one page may produce several chunks. Generated
chunk identifiers are also different from the document ID and original page.

**Why this matters:** chunking determines the text units embedded and later
retrieved. Correct source metadata must survive the split so each result
can still be traced to its document.

## 7. Local embeddings: LocalQwenEmbeddings

**Application code:**
[local_embeddings.py](local_embeddings.py) supplies `LocalQwenEmbeddings`.
This class is a local adapter, not an IAT SDK class.

The indexer passes chunk text to `aembed_documents()`. The adapter calls
Ollama's local `/api/embed` endpoint using `qwen3-embedding:0.6b`. It checks
the number of returned vectors and validates that each vector has 1024
dimensions, finite values and at least one nonzero value.

This step creates numerical representations of text. It does not generate
a conversational answer. The `qwen3:4b` chat model is used in the later
Agent SDK exercise and is not needed for Repo 1 ingestion.

**Why this matters:** document vectors and future query vectors must be
compatible. Repo 2 reuses the same embedding model and vector dimensions.

## 8. Indexing: OracleVectorIndexer and the local indexer adapter

**Provided by the SDK:** `OracleVectorIndexer` performs Oracle indexing,
including batching, vector conversion, SQL insertion and indexing statistics.
It uses `OracleDBConnectionManager` for SDK database operations.

**Application code:** `LocalOracleIndexer` subclasses that indexer and
preserves its Oracle indexing implementation. In SDK 3.0.6, the default
indexer constructor creates an OCI embedding client. The supplied adapter
initializes the base component, supplies the constructor state needed by
the inherited implementation and installs `LocalQwenEmbeddings` instead.
Its embedding hook calls the local adapter rather than the OCI retry path.

The application subclass's `with_indexer()` selects this local indexer.
There is no built-in Ollama switch being enabled by the empty
`indexer.embeddings` mapping. That mapping works here because the supplied
application adapter replaces construction of the default embedding client.

**Input:** document chunks from the chunker queue.

**Output:** Oracle rows containing chunk text, its vector and serialized
metadata, with indexing statistics tracking processed, indexed and failed
chunks. This repository selects `float32` vector precision.

**Why this matters:** the indexer is the component that turns prepared text
into persisted vector-search data. Generating chunks alone does not store
anything in the destination table.

The parser/indexer overrides rely on version-specific SDK extension points
and internal attributes. Keep the SDK baseline pinned and review the
adapters before an upgrade. The workshop does not edit installed SDK files.

## 9. Oracle setup and data models

The workshop uses the `IT_SUPPORT_WORKSHOP` schema in `FREEPDB1`, with table
`IT_SUPPORT_EMBEDDINGS`. This is separate from the full application's schema.

| Stored field | Purpose |
| --- | --- |
| `ID` | Database row identifier. |
| `TEXT` | The indexed chunk text. |
| `PRESERVED_ORIGINAL_TEXT` | SDK-supported original-text field for indexing paths that preserve a separate version. |
| `EMBEDDING` | The 1024-dimensional vector for the chunk. |
| `METADATA` | JSON source and chunk information. |
| `CREATED_AT` | Row creation time. |

The source URL and original page are in metadata, not dedicated SQL columns
in this table. The row ID is not the original PDF page number.

The complete SDK data-model progression is:

| Model | Represents |
| --- | --- |
| `RawDocument` | Loaded file content and file metadata. |
| `ParsedDocument` | A document containing parsed structures. |
| `DocumentStructure` | A text, table or image structure; this exercise creates text structures for reviewed pages. |
| `DocumentChunk` | An indexing unit with text and metadata. |

[create_ingestion_table.py](create_ingestion_table.py) is an application setup
script using `python-oracledb` directly. The runner also uses that driver
for independent stored-data verification. Those utilities are separate from
the SDK's database connection manager used during actual indexing.

## 10. Checks, actual ingestion and completion feedback

The repository has three execution modes:

| Command | What executes |
| --- | --- |
| `python -m ingestion_pipeline.run_ingestion --check` | Validates the corpus and exercise configuration. After the three checks pass, generates a preview with the supplied parser and native SDK chunker. No Oracle connection or Ollama request is made. |
| `python -m ingestion_pipeline.run_ingestion` | Runs preflight, then runs the real IAT pipeline if the workshop table is empty. If records already exist, verifies them without inserting duplicates. |
| `python -m ingestion_pipeline.run_ingestion --verify` | Runs preflight and independently reads the stored workshop records. It does not generate embeddings or insert document rows. |

`_AssemblyProbe` is **workshop checking code**, not an SDK component. It
records which builder methods the shared assembly function calls during
offline checks. Actual ingestion passes a real `HelpdeskIngestionPipeline`
through that same assembly function.

The verification and completion feedback are also workshop application code.
Verification compares stored records against the supplied corpus and native
chunk preview, checking exact unique text, source metadata and finite,
nonzero 1024-dimensional vectors. Configuration checks alone do not trigger
the congratulations message.

For the frozen corpus and required policy, the verified result is:

| Document | Chunks |
| --- | ---: |
| `MS_WINDOWS_VPN_001` | 8 |
| `MS_WINDOWS_INSTALL_001` | 3 |
| `MS_STORE_APPS_001` | 2 |
| `MS_WORK_ACCOUNT_001` | 3 |
| **Total** | **16** |

Sixteen is the expected result for these supplied documents and settings,
not a universal SDK chunk count. Retrieval relevance and answer quality are
tested in later repositories.

## Component ownership at a glance

| SDK components | Application/workshop components |
| --- | --- |
| `ConfigManager`, `AppConfig` | The repository's YAML settings and source register |
| `IATIngestionPipeline`, `RAGToolkit` | `HelpdeskIngestionPipeline`, the assembly function and runner |
| `LoaderFactory`, `LocalStorageLoader`, `RawDocument` | The supplied approved cleaned JSON and original PDFs |
| `ParserProcessor`, `ParsedDocument`, `DocumentStructure` | `ReviewedJSONParser` and approval/hash/page validation |
| `ChunkProcessor`, `ChunkerFactory`, `RecursiveChunker`, `DocumentChunk` | The exercise's required chunking policy |
| `OracleVectorIndexer`, `BaseIndexer`, `OracleDBConnectionManager` | `LocalOracleIndexer`, `LocalQwenEmbeddings`, table setup and verification |

Ollama and Oracle Database are external local services. The application
adapters connect them to the SDK processing flow; they do not replace the
SDK's loader, chunking, queue orchestration or Oracle indexing implementation.
