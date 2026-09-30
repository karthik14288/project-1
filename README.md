# Unify — A Cross-Modal AI Understanding Platform

Unify is an end-to-end, production-grade cross-modal Retrieval-Augmented Generation (RAG) platform. It ingests fragmented data formats—**audio voice notes, video lectures, scanned lab records, PDF documents, and drone imagery**—into a shared 768-dimensional vector space powered by **Supabase PostgreSQL (pgvector)** and **Google Gemini 2.5 Pro**, returning synthesized reasoning with **verbatim, clickable, timestamped, and page-level citations**.

---

## 🌟 Core Architecture & Capabilities

```mermaid
graph TD
    A[Multimodal Inputs<br/>Audio .mp3, Video .mp4, Docs .pdf, Photos .jpg] --> B[Supabase Storage<br/>unify-sources bucket]
    A --> C[Gemini 2.5 Pro Multimodal Pipeline<br/>Transcribes with MM:SS & OCRs with Page X]
    C --> D[Intelligent Vector Chunking<br/>Metadata-aware segmenter]
    D --> E[Gemini text-embedding-004<br/>768-dim embeddings]
    E --> F[(PostgreSQL + pgvector<br/>HNSW Cosine Distance Index)]
    
    G[User Query + Domain Lens<br/>Education | Healthcare | Agriculture] --> H[text-embedding-004<br/>Query Vector]
    H --> I[match_document_chunks RPC<br/>Cross-Modal Similarity Search]
    F --> I
    I --> J[Gemini 2.5 Pro Reasoning<br/>System Prompt + Structured Output Schema]
    J --> K[Synthesized Answer + Interactive Citations<br/>Clickable Badges to Exact Timestamp / Page]
```

### 🎯 3 Specialized Domain Lenses
- **🌾 Agriculture Lens**: Synthesizes farmer voice memos, soil sensor reports, and drone aerial NDVI photos for diagnosing crop health issues (e.g. nitrogen deficiency vs moisture stress).
- **🩺 Healthcare Lens**: Synthesizes physician voice dictations, scanned laboratory panels, and progress notes with clinical disclaimers and chronological biomarker tracking.
- **🎓 Education Lens**: Synthesizes lecture recordings (with `[MM:SS]` timecodes), scanned handouts, and whiteboard notes for study questions and formula breakdowns.

---

## 🏗️ Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Frontend** | React 18, Vite, TypeScript | High-performance SPA with client-side routing |
| **Styling** | Tailwind CSS + Lucide Icons | Polished glassmorphism, responsive dark-mode palette |
| **Backend** | Node.js, Express.js (TypeScript) | RESTful API with Zod validation, Helmet & rate-limiting |
| **Vector DB & Auth** | Supabase (PostgreSQL + pgvector) | 768-dim HNSW vector index, Row-Level Security (RLS) |
| **AI Reasoning** | `@google/genai` (Gemini 2.5 Pro) | Multimodal extraction & structured JSON citation reasoning |
| **Embeddings** | `@google/genai` (text-embedding-004) | 768-dimensional semantic embedding space |
| **Cloud Deployment** | Vercel Serverless Function & SPA | Turnkey production hosting via `vercel.json` |

---

## 🗄️ Database Schema & Setup

Execute `database/schema.sql` directly in the **Supabase SQL Editor**:

```sql
-- 1. Enable pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. Sources table
CREATE TABLE IF NOT EXISTS sources (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    file_name TEXT NOT NULL,
    file_type TEXT NOT NULL,
    storage_path TEXT NOT NULL,
    lens_category TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. Document chunks table (768-dim vector)
CREATE TABLE IF NOT EXISTS document_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id UUID NOT NULL REFERENCES sources(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    page_number INT,
    start_time NUMERIC,
    end_time NUMERIC,
    embedding vector(768) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 4. HNSW Index for fast approximate cosine similarity
CREATE INDEX IF NOT EXISTS document_chunks_embedding_hnsw_idx 
ON document_chunks USING hnsw (embedding vector_cosine_ops);

-- 5. Row-Level Security (RLS)
ALTER TABLE sources ENABLE ROW LEVEL SECURITY;
ALTER TABLE document_chunks ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can only view their own sources" ON sources FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can insert their own sources" ON sources FOR INSERT WITH CHECK (auth.uid() = user_id);
CREATE POLICY "Users can delete their own sources" ON sources FOR DELETE USING (auth.uid() = user_id);

CREATE POLICY "Users can only view their own chunks" ON document_chunks FOR SELECT USING (auth.uid() = user_id);
CREATE POLICY "Users can insert their own chunks" ON document_chunks FOR INSERT WITH CHECK (auth.uid() = user_id);
CREATE POLICY "Users can delete their own chunks" ON document_chunks FOR DELETE USING (auth.uid() = user_id);
```

---

## 🚀 Getting Started Locally

### 1. Prerequisites
- **Node.js** v18+ (tested on Node v24)
- **Supabase Account** (Credentials pre-configured in `.env`)
- **Google Gemini API Key**

### 2. Configure Environment Variables

**Backend (`server/.env`):**
```env
PORT=8080
NODE_ENV=development
SUPABASE_URL=https://eqtapthujqoonvpwipgv.supabase.co
SUPABASE_SERVICE_ROLE_KEY=sb_secret_HqAOBOSpP7uh6S7Dq04ykw_pqUL0YuM
SUPABASE_JWKS_URL=https://eqtapthujqoonvpwipgv.supabase.co/auth/v1/.well-known/jwks.json
GEMINI_API_KEY=your_google_gemini_api_key_here
```

**Frontend (`client/.env`):**
```env
VITE_SUPABASE_URL=https://eqtapthujqoonvpwipgv.supabase.co
VITE_SUPABASE_ANON_KEY=sb_publishable_rBt3VP0s5tAkbFie2yVshA_021DhOXM
VITE_API_URL=
```

### 3. Run Development Servers

**Run Server:**
```bash
cd server
npm install
npm run dev
# Server runs on http://localhost:8080
```

**Run Client:**
```bash
cd client
npm install
npm run dev
# Frontend runs on http://localhost:5173
```

---

## ☁️ Deploying to Vercel

The repository is pre-configured with a root `vercel.json` and `/api/index.ts` serverless adapter.

### One-Click Vercel Setup:
1. Push this repository to GitHub.
2. In the Vercel Dashboard, import the repository.
3. Configure the following **Environment Variables** in Vercel Project Settings:
   - `SUPABASE_URL`
   - `SUPABASE_SERVICE_ROLE_KEY`
   - `GEMINI_API_KEY`
   - `VITE_SUPABASE_URL`
   - `VITE_SUPABASE_ANON_KEY`
4. Click **Deploy**. Vercel will automatically build the React Vite SPA into `client/dist` and deploy the Express API via the serverless handler in `/api/index.ts`.

---

## 🧪 Testing the User Flow

1. Open `http://localhost:5173/`.
2. Click **"Enter Instant Sandbox (Guest Mode)"** or sign in with your Supabase account.
3. You will land on the **Workspace & Lens Selector (`/dashboard`)**.
4. Go to **Knowledge Ingest (`/ingest`)**:
   - Drag & drop an audio `.mp3`, video `.mp4`, photo `.jpg`, or PDF `.pdf`.
   - Watch the 4-stage pipeline: `Extracting -> Chunking -> Embedding -> Saved`.
   - Or click **"Seed Sample Knowledge Base"** to immediately populate realistic multimodal test sources for Agriculture, Healthcare, and Education.
5. Go to **Cross-Modal Chat (`/chat`)**:
   - Select the **Agriculture** lens.
   - Click the prompt: *"What is causing the leaf yellowing in the north quadrant?"*
   - Observe how Gemini synthesizes context from both the farmer voice memo and drone aerial scan.
   - Click any citation badge like `[1]` or `[2]` to open the **Citation Source Inspector Drawer** and see the exact file name, timestamp (`0:15 - 0:35`), and snippet quote.
