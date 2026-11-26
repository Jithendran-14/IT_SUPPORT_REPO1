# Repo 1 hints

Use these progressively. Start with the concept hint before opening the
location hint. Setup problems are described in README.md and are not
intentional exercise defects.

Run `python -m ingestion_pipeline.run_ingestion --check` after each change.
It reports R1-1, R1-2 and R1-3 without connecting to Oracle or Ollama.
The final congratulations require real stored-data verification.

## R1-1 - Source configuration and loader

**Concept hint:** A loader selects a named source configuration, not a PDF
filename or database table. The selected source describes where input
documents are located and which extensions to load.

<details>
<summary>Location hint</summary>

Compare `source_name` in `assemble_pipeline()` in
`ingestion_pipeline/run_ingestion.py` with the keys under
`ingestion.sources` in `ingestion_pipeline/ingestion_config.yaml`.
They must refer to the same supplied source.

</details>

## R1-2 - Recursive chunking configuration

**Concept hint:** Chunk size limits a text unit. Overlap carries some text
between neighbouring units to retain context. Overlap must fit inside the
chunk; the exercise also specifies an exact chunking policy.

<details>
<summary>Location hint</summary>

Inspect `chunking.strategies.text` in the YAML. Compare its two numeric
settings with the 800-character size and 100-character overlap required
by README.md. Leave the supplied recursive strategy and separators intact.

</details>

## R1-3 - Attaching the indexer

**Concept hint:** Loading, parsing and chunking prepare text. Another
component calls the embedding adapter and persists the chunks and vectors.
The pipeline builder must attach that component before it is built.

<details>
<summary>Location hint</summary>

Inspect `assemble_pipeline()` and the R1-3 marker. Read the corresponding
method supplied by `HelpdeskIngestionPipeline` in `helpdesk_pipeline.py`.
Keep the supplied local indexer adapter; do not edit site-packages.

</details>

## After all three checks pass

Run the ingestion command without `--check`. Completion requires four
approved documents, 16 exact unique chunks, 1024-dimensional vectors and
valid source metadata in the workshop schema. The program then prints
congratulations and an explanation of every corrected SDK integration.

Do not change the source register, source files, verification functions or
expected counts to make a check pass. They define the exercise contract.
