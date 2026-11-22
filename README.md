# Infosys Annual Report Q&A Assistant (RAG)

Ask questions in plain English about the **Infosys Integrated Annual Report 2023-24** (353 pages) and get an answer together with the **file and page** it came from. If the report doesn't contain the answer, the assistant says so instead of guessing.

Built as a Retrieval-Augmented Generation (RAG) project in a single Google Colab notebook: `rag_annual_report_assistant.ipynb`.

## How it works

```
Upload PDF -> Chunk -> Embed -> Hybrid search -> Re-rank -> Answer model -> Answer + sources
                               (meaning + keywords)  (reads question and chunk together)
```

| Stage | What it does | Tool |
|---|---|---|
| Chunking | Splits each page into overlapping ~80-word pieces | plain Python |
| Embeddings | Turns chunks into vectors that capture meaning | `BAAI/bge-small-en-v1.5` |
| Hybrid search | Combines meaning search with BM25 keyword search | `rank_bm25`, NumPy |
| Re-ranker | Re-reads the question with each candidate chunk to fix the order | `cross-encoder/ms-marco-MiniLM-L-6-v2` |
| Answer model | Writes the answer using only the top 5 chunks | Gemini API (`gemini-3.1-flash-lite`) |

The notebook also has an optional free local model (`flan-t5-large`) for people without an API key.

## Results

I wrote a 12-question test set from the report and measured each improvement.

**Search quality** (is the right chunk among the top results?)

| Search version | hit@1 | hit@3 | hit@5 |
|---|---|---|---|
| Dense (meaning only) | 6/12 | 9/12 | 9/12 |
| Hybrid (+ BM25 keywords) | 6/12 | 9/12 | 11/12 |
| Hybrid + re-ranker | **8/12** | **11/12** | **12/12** |

**Answer quality** (does the answer contain the correct value?)

| Answer model | Chunks given | Correct |
|---|---|---|
| flan-t5-large (free, local) | top 3 | about 9/12 |
| Gemini | top 3 | 11/12 (the 12th was an honest "I don't have that information") |
| Gemini | top 5 | 12/12 |

What I learned: keyword search rescued exact-term questions ("interim dividend", "attrition"), the re-ranker pushed the right chunk higher, and after that the answer model became the bottleneck. The small local model could not match numbers to labels in jumbled PDF tables, and a stronger model fixed most of it.

## Limitations (please read)

- **Small test set.** 12 questions, written by me, on one report. One question is about 8 percentage points, so treat the scores as indicative, not proof.
- **Partly tuned.** I increased the context from 3 to 5 chunks after one question failed, so the 12/12 is optimistic. A fresh question set is the honest test.
- **Rough auto-check.** The notebook checks whether the expected value appears in the answer. An answer can contain the right value and still be partly wrong, so I also read the answers myself.
- **Tables.** PDF tables become jumbled text, which is where answers go wrong most often.
- **Scanned PDFs** are not supported (no OCR).
- **Free API limits.** The free Gemini tier has a small daily request quota, and the API sometimes returns "high demand" errors. The notebook saves answers and stops cleanly when the quota runs out.

## Run it yourself

1. Download the report PDF from the NSE archive: https://nsearchives.nseindia.com/corporate/INFY_03062024153620_infosysar24.pdf
2. Open `rag_annual_report_assistant.ipynb` in [Google Colab](https://colab.research.google.com).
3. Create a Gemini API key in Google AI Studio. In Colab, open **Secrets** (key icon), add a secret named `LLM_API_KEY` with your key, and turn on **Notebook access**. Never paste the key into the notebook.
4. Run the cells one by one (not "Run all"). Upload the PDF in Step 3.
5. No key? Set `ANSWER_BACKEND = "local"` in Step 2 to use the free local model.

## Tech stack

Python, sentence-transformers, rank_bm25, pypdf, NumPy, Google Gemini API, Google Colab.

## Next steps

- Grow the test set with new, unseen questions and report the scores again
- Add a second company's report
- Wrap `ask()` in a small Gradio web app
- Extract tables separately to improve number questions
