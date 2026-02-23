# AutoRAG (Technology Preview)

**AutoRAG** on Red Hat OpenShift AI lets you run and evaluate **Retrieval-Augmented Generation (RAG)** over your documents from a **notebook** in the workbench. You provide documents and test questions; the notebook drives execution (using [IBM ai4rag](https://github.com/IBM/ai4rag)-style workflows) against a **Llama-stack RAG server** and lets you explore answers, retrieval, and metrics directly in the notebook. See [Example scenarios](#4-example-scenarios) for a typical use case and a step-by-step tutorial.

**Status:** Technology Preview — This feature is not yet supported with Red Hat production service level agreements (SLAs) and may change. It provides early access for testing and feedback.

---

## 1. About AutoRAG

### 1.1. What AutoRAG gives you

AutoRAG in this preview is **notebook-driven**: you run a notebook in an OpenShift AI workbench that executes RAG against your documents and a prepared RAG stack.

- **Document-based Q&A** — Your documents (e.g., PDFs or text) are stored in S3. The notebook uses them as the knowledge base for retrieval-augmented question answering.
- **Test data** — A `test_data.json` file (also in S3) defines the questions (and optionally expected answers) used to run and evaluate RAG.
- **RAG stack** — A **Llama-stack server** with the RAG stack (chat model, embedding model, vector store such as Milvus) is a prerequisite. The notebook sends requests to this stack for embedding, retrieval, and generation.
- **Results in the notebook** — You run the notebook cell-by-cell, then explore answers, retrieved chunks, and evaluation results directly in the notebook (no separate pipeline UI).

You do not need to deploy the RAG application yourself for this flow; the notebook orchestrates runs against the existing RAG stack and lets you inspect outcomes.

### 1.2. What AutoRAG supports (Technology Preview)

In this preview, AutoRAG is exposed as a **workbench notebook** that uses IBM ai4rag-style execution against Red Hat OpenShift AI’s Llama-stack RAG infrastructure.

| Area | Support |
|------|--------|
| **Documents** | Stored in S3-compatible object storage (via RHOAI Connections); e.g. PDF or text (IBM financial reports). |
| **Test data** | JSON file in S3 (e.g. `test_data.json`) with questions (and optional references) for evaluation. |
| **RAG stack** | Llama-stack server with RAG stack (chat model, embedding model, vector store e.g. Milvus). |
| **Execution** | Notebook in OpenShift AI workbench (ai4rag-style runs). |
| **What you get** | Answers, retrieved context, and evaluation results explored in the notebook. |

**Not in scope (this preview):** Fully automated RAG pipeline runs (e.g. via Kubeflow Pipelines).

### 1.3. How it works under the hood

The notebook runs in your **workbench** and uses **RHOAI Connections** to read documents and test data from S3. It calls the **Llama-stack RAG server** (deployed as a prerequisite in your project) for embeddings, retrieval, and LLM responses. The flow follows patterns similar to [IBM ai4rag](https://github.com/IBM/ai4rag): ingest documents, run queries from test data, and evaluate results. Implementation details depend on the notebook and the RAG stack configuration (see [References](#references)).

### 1.4. Sample notebook and experiment flow (ai4rag)

The scenario is based on the [IBM ai4rag](https://github.com/IBM/ai4rag) sample notebook and experiment script. In that pattern:

1. **Connect to the RAG stack** — The notebook uses a client (e.g. `LlamaStackClient`) configured with the **Llama-stack base URL** (the RAG server endpoint). You set this URL in the notebook so it targets your deployed Llama-stack RAG server.
2. **Load documents** — Documents are loaded from a path (e.g. via `FileStore` or from an S3-mounted/local path). In the workbench, you point this path to where your S3 connection exposes the documents (e.g. after syncing or mounting the bucket).
3. **Load benchmark/test data** — The notebook loads the question set from a JSON file (e.g. `read_benchmark_from_json` for `test_data.json` or `benchmark_data.json`). The JSON contains the questions and optionally expected answers or references for evaluation.
4. **Configure search space** — You can define a search space (e.g. foundation model, embedding model, retrieval method) and optional optimizer settings (e.g. number of evaluations). The sample uses `AI4RAGSearchSpace` and `GAMOptSettings`; the notebook may use defaults or allow you to tune these.
5. **Run the experiment** — The notebook runs an **AI4RAG experiment** (e.g. `AI4RAGExperiment`) against the Llama-stack, typically with a vector store type such as `ls_milvus`. It runs the search/optimization and writes results to an output path.
6. **Explore results** — You get a **best configuration** (e.g. best model/embedding/retrieval combination) and can explore answers, retrieved chunks, and evaluation metrics in the notebook. Results may also be written to a local or S3 output directory.

For the exact cells and code, see the [run_ai4rag.ipynb](https://github.com/IBM/ai4rag/blob/dev-samples/samples/run_ai4rag.ipynb) notebook and [run_experiment.py](https://github.com/IBM/ai4rag/blob/dev-samples/dev_utils/run_experiment.py) script in the ai4rag repository (`dev-samples` branch).

---

## 2. What you need to provide

To run the AutoRAG notebook, you provide:

### Required

| Item | Description |
|------|-------------|
| **Documents** | Your source documents (e.g. IBM 2025 quarterly financial reports, one file per quarter) uploaded to an S3-compatible bucket. You can download quarterly earnings presentations (PDFs) from [IBM Financial Reporting](https://www.ibm.com/investor/financial-reporting) (select year 2025 and Q1–Q4). The notebook or RAG stack ingests them for retrieval. |
| **Test data** | A `test_data.json` file in S3 (in the ai4rag sample this is used as **benchmark data**, e.g. loaded via `read_benchmark_from_json`). It should contain the questions and optionally expected answers or references for evaluation. |
| **S3 connections** | RHOAI Connections so the workbench can read from the bucket(s) where documents and `test_data.json` are stored. |
| **RAG stack** | A Llama-stack server with the RAG stack enabled (chat model, embedding model, vector store such as Milvus), deployed in the project or accessible from the workbench. You will need the **RAG stack base URL** (e.g. for `LlamaStackClient` in the notebook) so the notebook can call the server. |

### Optional

| Item | Description |
|------|-------------|
| **Notebook config** | Paths for documents and `test_data.json` (or `benchmark_data.json`) in the notebook; bucket name and object keys if reading from S3. Optimizer settings (e.g. `max_evals`, `n_random_nodes`) and search space (foundation model, embedding model, retrieval method) if the notebook exposes them. |

---

## 3. What you get from a run

When you run the AutoRAG notebook:

- **Best configuration** — If the notebook runs an ai4rag experiment with search/optimization (e.g. GAMOpt), you get a **best configuration** (e.g. best foundation model, embedding model, and retrieval method combination) printed or displayed in the notebook.
- **Answers** — Model-generated answers for each question in `test_data.json` (benchmark data), visible in the notebook.
- **Retrieved context** — The chunks/documents retrieved for each query, so you can inspect what the RAG stack used.
- **Evaluation results** — Metrics or comparisons (e.g. against expected answers, if provided in test data) that you can explore in the notebook.
- **Output path** — The sample experiment script writes results to an output path (e.g. a local or S3 directory); the notebook may display or link to these artifacts.

All outcomes are explored **inside the notebook** (tables, markdown, or plots as defined by the notebook).

---

## 4. Example scenarios

AutoRAG in this preview is aimed at **document Q&A and evaluation**: you have a set of documents (e.g. reports, manuals) and a list of questions; you run the notebook to get answers and inspect retrieval and quality.

| Scenario | Your data | What you do | Outcome |
|----------|-----------|-------------|---------|
| **Financial reports Q&A** | IBM 2025 quarterly financial reports (PDF/text) in S3—download from [IBM Financial Reporting](https://www.ibm.com/investor/financial-reporting); `test_data.json` with questions | Run the notebook against the Llama-stack RAG server | Answers and retrieved snippets in the notebook; evaluate consistency and relevance. |
| **Internal docs Q&A** | Policy or product docs in S3; test questions in JSON | Same flow | Inspect answers and retrieval for each question in the notebook. |
| **Evaluation run** | Documents + test set with expected answers | Run notebook | Compare model answers to references and review metrics in the notebook. |

To try this yourself, follow the [Tutorial: Ask questions against 2025 IBM financial reports](#7-tutorial-ask-questions-against-2025-ibm-financial-reports): download the reports from [IBM Financial Reporting](https://www.ibm.com/investor/financial-reporting), upload IBM 2025 quarterly reports and `test_data.json` to S3, ensure the Llama-stack RAG stack is ready, run the notebook, and explore the results.

---

## 5. Prerequisites

- **Red Hat OpenShift AI** installed and accessible.
- A **data science project** and a **workbench** (notebook environment) where you will run the AutoRAG notebook.
- **Llama-stack server with RAG stack** — Deploy and configure a Llama-stack server in the project with the RAG stack (chat model, embedding model, and vector store such as Milvus). The notebook will use this server for retrieval and generation. See [Deploying a RAG stack in a project](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.0/html/working_with_llama_stack/deploying-a-rag-stack-in-a-project_rag) and [Build AI/Agentic Applications with Llama Stack](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.2/html-single/working_with_llama_stack/working_with_llama_stack).
- **S3 connection(s)** (RHOAI Connections) for:
  - The bucket containing your **documents** (e.g. IBM 2025 quarterly financial reports).
  - The bucket (or same bucket) containing **test_data.json**.
- Attach the S3 connection(s) to your workbench so the notebook can read documents and test data.

---

## 6. Running AutoRAG

You run AutoRAG by **running the notebook** in your workbench:

1. Ensure the **Llama-stack RAG stack** is deployed and that the notebook is configured to use it (e.g. endpoint, API key if required).
2. Ensure **documents** and **test_data.json** are uploaded to S3 and that the workbench has an S3 connection to those locations.
3. Open the AutoRAG notebook in the workbench, set any required paths (bucket names, object keys for documents and `test_data.json`) if needed.
4. Run the notebook cells in order. The notebook will drive ai4rag-style execution: load/ingest documents (or use existing vector store), run questions from `test_data.json`, call the RAG stack, and display answers and retrieval results.
5. **Explore the results** in the notebook: answers, retrieved chunks, and any evaluation metrics or plots.

There is no separate “AutoRAG pipeline run” in this preview; execution is entirely notebook-driven.

---

## 7. Tutorial: Ask questions against 2025 IBM financial reports

**Scenario:** You have **IBM financial reports from 2025** (one document per quarter) and a **test_data.json** file with questions about them. The goal is to run a RAG workflow from a notebook in OpenShift AI (using ai4rag-style execution) against a **Llama-stack RAG server**, then explore answers and retrieval results in the notebook.

This tutorial walks you through: creating a project and workbench, preparing S3 with the documents and test data, ensuring the Llama-stack RAG stack is deployed, running the AutoRAG notebook, and exploring the results in the notebook.

### 7.1. Create a project and workbench

1. Log in to Red Hat OpenShift AI.
2. Go to **Data science projects** and create a new project (e.g. `ibm-reports-rag`).
3. Create a **workbench** (notebook environment) in the project. Choose an image that includes the dependencies required by the AutoRAG notebook (e.g. Python, ai4rag-related libraries if used). For full steps, see [Creating a project and workbench](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.8/html/getting_started_with_red_hat_openshift_ai_self-managed/creating-a-project-workbench_get-started).

### 7.2. Deploy Llama-stack server with RAG stack

1. In the project, deploy a **Llama-stack server** with the **RAG stack** enabled (chat model, embedding model, vector store such as Milvus). Configure it according to your environment (see [Deploying a RAG stack in a project](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.0/html/working_with_llama_stack/deploying-a-rag-stack-in-a-project_rag)).
2. Note the RAG/API endpoint and any credentials the notebook will need to call the stack.

### 7.3. Create S3 connection and upload documents

1. In the project, open **Connections** and add an **S3 compatible object storage** connection to a bucket you will use for documents and test data.
2. Download **IBM financial reports from 2025** from [IBM Financial Reporting](https://www.ibm.com/investor/financial-reporting): under **Find a quarterly earnings presentation**, select year **2025** and download the PDFs for **Q1**, **Q2**, **Q3**, and **Q4** (e.g. Press Release, Charts, or Prepared Remarks per quarter, as needed).
3. Upload those PDFs to your S3 bucket — one file per quarter (or one combined set). Place them in a known path (e.g. `documents/2025/` or `documents/`). Use the format expected by your RAG stack/notebook.
4. Upload **test_data.json** to the same bucket (or a path the notebook will use). The JSON should contain the list of questions (and optionally expected answers or references) for evaluation. Example structure: `{"questions": [{"id": "1", "question": "..."}, ...]}` or similar as required by the notebook.
5. Note the **bucket name** and **object keys** (paths) for the documents and for `test_data.json`; you will set these in the notebook if required.

### 7.4. Attach S3 connection to the workbench

1. Open **Workbenches**, edit your workbench, and **attach the S3 connection** you created in 7.3 so the notebook can read from the bucket.
2. Save and restart the workbench if prompted.

### 7.5. Open and configure the AutoRAG notebook

1. Upload or clone the **AutoRAG notebook** into the workbench. You can use the [run_ai4rag.ipynb](https://github.com/IBM/ai4rag/blob/dev-samples/samples/run_ai4rag.ipynb) sample from the ai4rag repository (`dev-samples` branch) or an adapted version that uses S3 and your Llama-stack endpoint.
2. In the notebook, set the **document location** (path or bucket/key for the IBM 2025 reports). In the ai4rag sample, documents are loaded via a path (e.g. `FileStore`); in the workbench, use the path where your S3 connection exposes the documents (e.g. mounted bucket or a local copy).
3. Set the **benchmark/test data path** to your `test_data.json` (the sample uses `read_benchmark_from_json`; ensure the JSON format matches what the notebook expects).
4. Set the **RAG stack base URL** (e.g. the Llama-stack API base URL). In the sample this is used by `LlamaStackClient(base_url=...)`; use the URL of the Llama-stack RAG server you deployed in 7.2 (e.g. `http://<service>:8321` or the route URL). Add any API key or auth if required.

### 7.6. Run the notebook and explore results

1. Run the notebook **cell by cell** from top to bottom. The notebook will:
   - Create a client to the Llama-stack RAG server (e.g. `LlamaStackClient`) and load documents from the configured path (or ingest them into the vector store).
   - Load the benchmark data from `test_data.json` (e.g. `read_benchmark_from_json`).
   - Run the ai4rag experiment (e.g. `AI4RAGExperiment` with search space and optimizer settings). The sample uses a vector store type such as `ls_milvus` and may run a search to find a best configuration.
   - Display the **best configuration**, answers, retrieved chunks, and evaluation metrics in the notebook; results may also be written to an output path.
2. **Explore the results** in the notebook: inspect the best configuration, the generated answers, the retrieved document snippets for each question, and any tables or plots showing quality or relevance. Use this to assess RAG behavior and tune documents or test data as needed.

---

## References

- [IBM ai4rag](https://github.com/IBM/ai4rag) — RAG templates and optimization patterns
- [run_ai4rag.ipynb (ai4rag samples, dev-samples branch)](https://github.com/IBM/ai4rag/blob/dev-samples/samples/run_ai4rag.ipynb) — Sample notebook used in the scenario
- [run_experiment.py (ai4rag dev_utils, dev-samples branch)](https://github.com/IBM/ai4rag/blob/dev-samples/dev_utils/run_experiment.py) — Sample experiment script (LlamaStackClient, FileStore, benchmark JSON, AI4RAGExperiment, GAMOpt, ls_milvus)
- [Deploying a RAG stack in a project (Red Hat OpenShift AI)](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.0/html/working_with_llama_stack/deploying-a-rag-stack-in-a-project_rag)
- [Build AI/Agentic Applications with Llama Stack (Red Hat OpenShift AI)](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.2/html-single/working_with_llama_stack/working_with_llama_stack)
- [Using connections (Red Hat OpenShift AI)](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/2.22/html/working_on_data_science_projects/using-connections_projects)
