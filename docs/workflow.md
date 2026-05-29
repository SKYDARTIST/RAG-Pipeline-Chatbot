# Workflow Writeup

This document records how the project was built, what was validated, and what a reviewer should check when importing the workflow.

## Build Method

The project was built as an n8n-first automation rather than a custom app. The main deliverable is the workflow export in `workflow/rag-pipeline-chatbot.json`, plus documentation and a screenshot that explain the architecture.

The implementation path:

1. Build the ingestion lane: Google Drive trigger, file download, document loader, Gemini embeddings, Pinecone insert.
2. Build the retrieval lane: chat trigger, Gemini chat model, Pinecone retriever tool, AI Agent.
3. Connect the two lanes through the same Pinecone index and namespace.
4. Harden the AI Agent prompt so the model calls retrieval before answering.
5. Export and sanitize the workflow for GitHub.

## Debugging Notes

### PDF parser compatibility

n8n v2.22.5 and the document loader path were sensitive to `pdf-parse` versioning. The stable local setup pinned `pdf-parse@^1` and used npm overrides so the loader received the compatible parser version.

### Pinecone dimension matching

Pinecone rejects vectors when the index dimension does not match the embedding model output. The index was aligned to the Gemini embedding output before successful ingestion.

### Gemini tool calling

The agent needed a strict system message to avoid answering from model knowledge without first calling the retriever. The workflow prompt requires the `access the Book` retrieval tool on every question.

## Local Setup

1. Start n8n locally:

   ```bash
   npx n8n
   ```

2. Open `http://localhost:5678`.
3. Import `workflow/rag-pipeline-chatbot.json`.
4. Create and map credentials in n8n:
   - Google Drive OAuth2
   - Google Gemini / PaLM API
   - Pinecone API
5. Create a Pinecone dense index with cosine similarity and the correct Gemini embedding dimension.
6. Update the Google Drive folder node to watch your own test folder.
7. Drop a non-sensitive PDF into the folder and run an ingestion test.
8. Ask a question through the chat trigger and confirm the Pinecone retriever tool fires before the answer.

## Verification Checklist

- `jq empty workflow/rag-pipeline-chatbot.json` passes.
- Workflow imports into n8n without missing-node errors.
- Every credential reference is mapped to a credential in the local n8n instance.
- Google Drive trigger points to a test folder.
- Pinecone insert node writes records into the expected namespace.
- Chat execution log shows the retriever tool being called.
- Answer text is grounded in the uploaded PDF, or the agent says it could not find the answer.

## GitHub Safety Checklist

- No raw Gemini, Pinecone, or Google OAuth secrets are committed.
- n8n credential IDs are placeholders.
- Google Drive folder IDs and file IDs are placeholders.
- Webhook IDs, workflow IDs, version IDs, and n8n instance IDs are removed.
- Local n8n data folders, `.env` files, SQLite databases, and credential files are ignored.

## Known Limits

- This is an n8n workflow export, so there is no app build, unit test suite, or deployment pipeline in this repo.
- Validation depends on external services: Google Drive, Google Gemini, and Pinecone.
- The current template is single-namespace. Production multi-user usage needs namespace or metadata isolation.
- Large folders can create model and vector database costs; run only against test documents until cost controls are added.
