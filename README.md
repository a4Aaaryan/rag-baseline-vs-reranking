# Baseline vs. reranking RAG

Comparison of dense retrieval and cross-encoder reranking over one complex PDF, including generated answers, source evidence, rank shifts, and retrieval latency measurements.

## Source document

The experiment uses one document: *The Law of Root* (October 13, 2025), published by Leder Games. It is a 32-page rulebook with general rules, faction-specific exceptions, and constraints suitable for retrieval stress tests.

Download the PDF from the [publisher's resources page](https://ledergames.com/pages/resources) (the link labelled “Law of Root, Oct 2025”) and save it as `data/law_of_root_oct_2025.pdf`. The direct [PDF link](https://cdn.shopify.com/s/files/1/0106/0162/7706/files/Root_Base_Law_Oct_2025.pdf?v=1774903756) is also available.

Expected SHA-256: `d32c8c0681d5bac6d2b483fb75d77efb47e78deaa36cb798bc527a605909c045`.

The PDF is excluded from Git; the notebook verifies its checksum before use.

## Current stage

The notebook extracts and chunks the rulebook, checks chunk lengths against the embedding tokenizer, builds a FAISS cosine index, reranks each top-10 result with a cross-encoder, and compares the two top-three selections. Both pipelines use the same Groq-hosted Qwen answer settings. The evaluation includes 15 exploratory questions, full retrieved excerpts, response metadata, and repeated retrieval timings.

The corpus uses printed pages 2–29. Page 1 is the cover and table of contents; pages 30–31 contain credits and the index; page 32 is a condensed quick reference that repeats rules. These exclusions keep the experiment focused on the full rules and reduce duplicate search hits.

Chunking starts with a 260-word target, 320-word maximum, and about 60 words of overlap. Before indexing, the notebook reduces these sizes if needed until every chunk fits the bi-encoder's token limit. The final chunk list is shared by both pipelines.

## Run on Kaggle

1. Create a Kaggle Notebook and import `RAG_Baseline_vs_Reranking.ipynb`.
2. Enable Internet under Session options. GPU is optional.
3. Create a key in the [Groq Console](https://console.groq.com/keys). Add a Kaggle Secret named `GROQ_API_KEY` under Add-ons → Secrets and grant the notebook access to it. The old `GEMINI_API_KEY` secret is not used.
4. Run all cells. The notebook installs dependencies, downloads and verifies the official PDF, then writes the retrieval CSVs and `run_config.json` to `/kaggle/working`. The final cell generates or resumes answers in `answers_completed.csv`, including finish reasons and token usage.

The final cell attempts both answers for all 15 questions, skipping complete saved answers. It spaces requests by 20 seconds and retries short 429 limits using Groq's `retry-after` value. Groq limits vary by model and account; check your [current limits](https://console.groq.com/docs/rate-limits). If a long quota wait or another error prevents progress, it saves completed answers so you can rerun the final cell later.

## Resume the existing Kaggle run

Copy only the final code cell (section 8) from the updated notebook into a new cell at the end of your existing Kaggle notebook, then run that cell. Keep the previous output CSVs in `/kaggle/working` and enable the `GROQ_API_KEY` secret. There is no need to repeat the earlier cells while the session is still active.

The cell reads `retrieved_evidence.csv` and any existing `answers.csv` or `answers_completed.csv`. It writes `answers_completed.csv` without replacing the original `answers.csv`. It checks completion status, model, citations, and saved context fingerprints before reusing answers. The original export predates fingerprints, so its complete answers are adopted when the model and cited context match.

With the retrieval models/index still loaded, it screens additional questions from distinct rule families, then generates up to three candidate answer pairs. Download `answers_completed.csv`, `stress_screening.csv`, `stress_evidence.csv`, `stress_answers.csv`, `stress_timing.csv`, and `completion_run.json` when present. The screening uses exact source-rule anchors and can miss paraphrases or partial supporting evidence; it is not an automatic relevance judgment. Review the full contexts and answers before claiming two repaired failures. If the session has restarted, the original saved evidence is enough to recover answers, but sections 0-6 must be run to load the models/index for the extra search.

The notebook is self-contained on Kaggle; the local `data/` folder and `requirements.txt` are not needed there.

## Run locally

Use Python 3.11 or newer. Create a virtual environment, install dependencies with `pip install -r requirements.txt`, then open `RAG_Baseline_vs_Reranking.ipynb` in Jupyter. The first run downloads the two public Hugging Face models.

To generate answers locally, set `GROQ_API_KEY` in the environment used by the notebook. The configured model is `qwen/qwen3.8-27b`; check its [availability](https://console.groq.com/docs/models) in your Groq account before running. Keep API keys out of notebook cells and committed files.

## Included results

The committed results contain 15 original answer pairs and three additional stress-test answer pairs. All 36 generated answers finished with `stop` and no recorded API errors. The answer context fingerprints match the exported excerpts. `answers_completed.csv` supersedes the incomplete `answers.csv`, which is excluded from Git.

Two reviewed examples show a missing rule entering the top three and improving the answer:

| Query | Critical chunk | Rank change | Observed improvement |
| --- | --- | --- | --- |
| Can a hireling use its controller's Boat Builders effect? | `chunk_0292` | 5 to 1 | Baseline lacks evidence; reranked answer identifies the prohibition in H.3.5. |
| Can Exposure be used after starting the final draw-and-discard step? | `chunk_0154` | 8 to 1 | Baseline lacks evidence; reranked answer identifies the timing restriction in 13.2.4. |

The Hostile coalition stress test provides another improvement (`chunk_0092`, rank 4 to 1). The Winter-map case remains a failure after reranking: the retrieved chunk omits its parent Winter Map heading, and the answer misinterprets the exception's scope. Retain this case as a limitation rather than a successful repair.

| Files | Contents |
| --- | --- |
| `answers_completed.csv` | Completed answers and generation metadata for the original 15 questions |
| `rank_shifts.csv`, `retrieved_evidence.csv` | Original top-ten ranks, scores, and full retrieved excerpts |
| `corpus_chunks.csv` | The shared 317-chunk corpus with source-page metadata |
| `retrieval_timing.csv`, `retrieval_timing_summary.csv` | Five retrieval samples per original query and per-query medians |
| `stress_answers.csv`, `stress_evidence.csv` | Three additional answer pairs and evidence for all screened questions |
| `stress_screening.csv`, `stress_timing.csv` | Candidate screening and five timing samples per selected stress query |

Use the CSVs and printed rules to assess relevance; an automatic anchor match or rank change alone does not prove an answer improvement. The assignment report must be written by the student in their own words and is not included here yet. The notebook's `run_config.json` and `completion_run.json` are generated on execution; those files were not supplied with the downloaded results and are not reconstructed retroactively.
