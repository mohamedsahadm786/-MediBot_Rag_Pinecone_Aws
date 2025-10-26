# MediBot — Medical RAG Chatbot (AWS • Pinecone • LangChain • OpenAI)

A production-ready Retrieval-Augmented Generation (RAG) chatbot that answers medical questions from a **700+ page PDF corpus** with fast, grounded, and citeable responses.  
It uses **HuggingFace embeddings** → **Pinecone** for vector search → **LangChain** orchestration → **OpenAI (GPT-4o)** for generation, served with a **Flask** web app, automated by **CI/CD (GitHub Actions)**, and deployed on **AWS**.

---

## ✨ Features

- **End-to-end RAG pipeline**: ingest PDF(s) → chunk → embed (HuggingFace) → store (Pinecone) → retrieve (k=3) → generate answer (OpenAI via LangChain).  
- **Scalable vector search** on **Pinecone Serverless** with cosine similarity and 384-dim embeddings.  
- **Fast Flask web app** with a minimal chat UI (`/`), and a `/get` endpoint wired to the RAG chain.  
- **OpenAI GPT-4o** as the chat model with a custom **system prompt** and LangChain **stuff-combine** document chain.  
- **Production-ready deployment** on AWS + **Docker** + **GitHub Actions** CI/CD (structure prepared in `.github/workflows/`).  
- **12-Factor style** config via environment variables (`.env` with `PINECONE_API_KEY`, `OPENAI_API_KEY`).

---

## 🗂 Repository Structure

```
.
├── app.py                # Flask app: retrieval + generation API + web route
├── store_index.py        # Data ingestion: load PDFs → chunk → embed → upsert to Pinecone
├── src/
│   ├── helper.py         # load_pdf_file, text_split, download_hugging_face_embeddings, etc.
│   └── prompt.py         # system_prompt used by LangChain ChatPromptTemplate
├── templates/
│   └── chat.html         # Minimal chat UI for Flask
├── static/               # Frontend assets (CSS/JS) if any
├── data/                 # Place PDFs here for ingestion
├── requirements.txt
├── Dockerfile
└── .github/workflows/    # CI/CD pipeline(s) for test/build/deploy
```

Key entry points:
- **`store_index.py`** builds/updates the Pinecone index named **`medical-chatbot`** (create if missing) and pushes embeddings.  
- **`app.py`** loads the same index, composes a **retrieval chain** (`k=3`) with **GPT-4o**, and serves `/get`.

---

## 🧱 Architecture (RAG)

```
flowchart LR
    A[PDF Corpus (700+ pages)] --> B[Loader & Cleaner<br/>src.helper.load_pdf_file]
    B --> C[Chunker<br/>text_split()]
    C --> D[HuggingFace Embeddings]
    D --> E[Pinecone Index<br/>medical-chatbot]
    U[User Query] --> F[Retriever (k=3)<br/>similarity search]
    E --> F
    F --> G[LangChain Stuff-Combine<br/>+ System Prompt]
    G --> H[OpenAI GPT-4o]
    H --> I[Flask API (/get) + UI]
```

- **Index settings**: cosine metric, 384-dim embeddings, Pinecone Serverless (AWS us-east-1).  
- **Retrieval config**: similarity search, `k=3`.  
- **Model**: `ChatOpenAI(model="gpt-4o")`.

---

## 🚀 Quickstart

### 1) Prerequisites
- Python 3.10+
- Accounts/keys: **OpenAI**, **Pinecone**
- (Optional) Docker & AWS CLI (for containerized deployment)

### 2) Install & Configure
```bash
git clone https://github.com/<you>/MediBot_RAG_Pinecone_Aws.git
cd MediBot_RAG_Pinecone_Aws
python -m venv .venv && source .venv/bin/activate  # or .venv\Scripts\activate on Windows
pip install -r requirements.txt
```

Create a `.env` file at repo root:
```bash
PINECONE_API_KEY=xxxxxxxxxxxxxxxxxxxxxxxx
OPENAI_API_KEY=xxxxxxxxxxxxxxxxxxxxxxxx
```
The app reads these at runtime.

### 3) Prepare Data & Build the Index
- Put your **PDFs** into `./data/`.
- Run the ingestion script:
```bash
python store_index.py
```
This will:
1) Load PDFs  
2) Clean & chunk  
3) Compute **HuggingFace embeddings**  
4) Create (if needed) and upsert to Pinecone index **`medical-chatbot`**

### 4) Run the App (Local)
```bash
python app.py
# Flask will start on 0.0.0.0:8080 (debug mode enabled)
```
Open the browser at `http://localhost:8080` and ask a question. The UI posts to `/get`, which calls the RAG chain and returns the response.

---

## 🧪 API Endpoints

- `GET /` → renders `templates/chat.html` (simple chat page).  
- `POST /get` (form field `msg`) → runs retrieval + generation and returns final string answer.

---

## ⚙️ Configuration Details

- **Index name**: `medical-chatbot`  
- **Retriever**: `docsearch.as_retriever(search_type="similarity", k=3)`  
- **Model**: `ChatOpenAI(model="gpt-4o")`  
- **Prompting**: `ChatPromptTemplate` with a custom `system_prompt` pulled from `src.prompt`  
- **Combiner**: `create_stuff_documents_chain` + `create_retrieval_chain`  
- **Server**: Flask on `0.0.0.0:8080` in debug (change for production)

---

## 🐳 Docker (Optional)

Build and run:
```bash
docker build -t medibot:latest .
docker run --env-file .env -p 8080:8080 medibot:latest
```

---

## ☁️ AWS Deployment (One Simple Path)

> Many options exist; below is a minimal EC2 path that mirrors local behavior.

1. **Provision EC2** (Amazon Linux 2023 / Ubuntu 22.04), open **Security Group** ports `80` (HTTP) and `8080` (if needed).  
2. **Install Docker** on EC2 and copy your **`.env`** securely (never commit secrets).  
3. **Push image** to ECR or **build on EC2**:
   ```bash
   docker build -t medibot:latest .
   docker run -d --restart=always --env-file .env -p 80:8080 medibot:latest
   ```
4. **Point a domain** (Route 53) or access via public IP.  
5. **Hardening** (recommended): run via **nginx reverse proxy**, enable HTTPS (ACM/Certbot).

---

## 🔁 CI/CD (GitHub Actions)

- Add a workflow under `.github/workflows/` to:
  - run tests/lint  
  - build Docker image  
  - push to ECR (or GHCR)  
  - deploy to EC2 via SSH or ECS/EKS  
- Store **OpenAI** and **Pinecone** keys in **GitHub Secrets**; never hardcode in repo.

---

## 📊 Evaluation (Optional but Recommended)

- **Retrieval quality**: recall@k, MRR; spot-check nearest neighbors for representative queries.  
- **Answer quality**: manual rubric + small golden set; track hallucination rate.  
- **Latency**: p50/p95 for end-to-end query.  
- **Observability**: structured logs for query → retrieved docs → final answer (never log secrets/PII).

---

## 🔒 Security & Safety Notes

- Keep **API keys** in environment variables/secret managers (AWS Secrets Manager).  
- For medical content, add:  
  - **disclaimer** banner  
  - guardrail prompt (“informational, not medical advice; consult professionals”)  
  - optionally moderate inputs/outputs  
- Limit prompt injection by constraining system prompt and post-processing references.

---

## 🧰 Tech Stack

**Python**, **Flask**, **LangChain**, **OpenAI GPT-4o**, **HuggingFace Embeddings**, **Pinecone (Serverless)**, **Docker**, **GitHub Actions**, **AWS (EC2/S3/IAM)**.  
Configuration and API keys loaded via **dotenv**.

---

## 🔧 Troubleshooting

- **Index not found / dimension mismatch** → ensure `store_index.py` ran and `dimension=384` matches your embedding model.  
- **401/403 from Pinecone** → confirm `PINECONE_API_KEY` in `.env` and region `us-east-1` for Serverless.  
- **OpenAI key errors** → verify `OPENAI_API_KEY` present and usage limits.  
- **Flask not reachable** → open the correct port (`8080` locally; map to `80` on EC2).

---

## 🗺 Roadmap

- Source citations in answers (return retrieved doc snippets + links).  
- Streaming responses to UI; typing effect.  
- Evaluation harness & regression tests for retrieval changes.  
- Switch to **batch embeddings** + **ingestion CLI**.  
- Add **Auth** for the admin dashboard and usage analytics.


---

## 🙌 Acknowledgments

- Built with ❤️ using **LangChain**, **Pinecone**, **HuggingFace**, and **OpenAI**.  
- Special thanks to open-source contributors and documentation references.

---

### Appendix — Key Implementation References

- Flask app routes and RAG chain wiring in `app.py` (env keys, retriever k=3, GPT-4o, `/get`).  
- Index creation and ingestion in `store_index.py` (ServerlessSpec, cosine, dim=384).
