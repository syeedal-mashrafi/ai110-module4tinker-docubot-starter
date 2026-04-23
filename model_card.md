# DocuBot Model Card

## 1. System Overview

**What is DocuBot trying to do?**

> DocuBot is a lightweight documentation assistant that answers developer questions about a codebase. It reads local markdown files and uses retrieval to find relevant information before generating an answer. The goal is to reduce hallucinations by grounding responses in actual project documentation.

**What inputs does DocuBot take?**

> DocuBot takes a developer question as input, reads documentation files from the docs/ folder, and uses the GEMINI_API_KEY environment variable to access the LLM for modes 1 and 3.

**What outputs does DocuBot produce?**

> DocuBot produces text answers. In Mode 1 it generates a free-form LLM response. In Mode 2 it returns raw retrieved document snippets with filenames. In Mode 3 it returns a grounded LLM answer based only on the retrieved snippets.

---

## 2. Retrieval Design

**How does your retrieval system work?**

> Documents are split into words, lowercased, and stored in an inverted index mapping each word to the filenames it appears in. Relevance is scored by counting how many query words appear in each document. The top 3 highest-scoring documents are returned as snippets.

**What tradeoffs did you make?**

> I chose simplicity over accuracy. Word-count scoring is fast and easy to understand but does not handle synonyms or paraphrased questions.

---

## 3. Use of the LLM (Gemini)

**When does DocuBot call the LLM and when does it not?**

- Naive LLM mode: Calls the LLM with the full docs and the question, no retrieval.
- Retrieval only mode: No LLM is called. Returns raw document snippets.
- RAG mode: Retrieval runs first, then the LLM is called with only those snippets.

**What instructions do you give the LLM to keep it grounded?**

> In RAG mode the prompt instructs the LLM to answer using only the provided snippets, not to invent new functions or endpoints, to say I do not know based on the docs I have if snippets are not sufficient, and to mention which files it relied on.

---

## 4. Experiments and Comparisons

| Query | Naive LLM | Retrieval only | RAG | Notes |
|-------|-----------|----------------|-----|-------|
| Where is the auth token generated? | Harmful - fabricated JWT/OAuth details | Helpful - returned AUTH.md | Helpful - short grounded answer | RAG gave clearest answer |
| How do I connect to the database? | Harmful - generic SQL advice | Helpful - returned DATABASE.md | Helpful - summarized DATABASE.md | Both retrieval and RAG worked |
| Which endpoint lists all users? | Harmful - invented endpoints | Helpful - returned API_REFERENCE.md | Helpful - correctly identified GET /api/users | RAG was most concise |
| How does a client refresh a token? | Harmful - mixed real and fabricated info | Helpful - returned AUTH.md | Helpful - cited AUTH.md correctly | Good RAG vs naive example |

**What patterns did you notice?**

> Naive LLM looks impressive but regularly invents details not in the docs. Retrieval only is best when you need to verify exact source content. RAG is best when you want a readable answer that is still grounded in actual documentation.

---

## 5. Failure Cases and Guardrails

**Describe at least two concrete failure cases you observed.**

> Failure case 1: Question - Where is the auth token generated in Naive LLM mode. The system described JWT signing and OAuth flows not found in the docs. It should have retrieved from AUTH.md first.

> Failure case 2: Question - What is the weather today in Retrieval only mode. The word today matched content in AUTH.md so it returned irrelevant docs instead of refusing. It should have returned I do not know based on these docs.

**When should DocuBot say I do not know based on the docs I have?**

> 1. When the query is completely unrelated to the documentation. 2. When the retrieved snippets have a score of zero meaning no query words matched any document.

**What guardrails did you implement?**

> The system returns I do not know based on these docs when retrieve returns an empty list. In RAG mode the LLM prompt also instructs the model to refuse if snippets do not support a confident answer.

---

## 6. Limitations and Future Improvements

**Current limitations**

1. Keyword scoring does not handle synonyms or paraphrased questions.
2. Retrieval unit is a whole document so snippets can be very long with irrelevant sections.
3. No memory of previous questions so it cannot handle follow-up conversations.

**Future improvements**

1. Split documents into paragraphs before indexing so snippets are smaller and more focused.
2. Use semantic embeddings instead of keyword matching to handle paraphrased queries.
3. Add a confidence score threshold so the system refuses when retrieval scores are too low.

---

## 7. Responsible Use

**Where could this system cause real world harm if used carelessly?**

> Developers trusting DocuBot without verifying answers could implement incorrect authentication logic or misconfigure environment variables. Naive LLM mode is especially dangerous because it sounds confident while fabricating details.

**What instructions would you give real developers who want to use DocuBot safely?**

- Always verify DocuBot answers against the original documentation files before implementing in production.
- Prefer RAG mode over Naive LLM mode since it cites source files.
- Treat any answer that does not cite a specific file as potentially hallucinated.
- Do not use DocuBot as the sole source of truth for security-sensitive topics.
