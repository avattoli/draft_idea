# Video Segment Search

## Goal
Upload a video, then search it with a text prompt to return the most relevant **segments (timestamps)**.

Example: 
1) Upload a video taken at the zoo
2) Enter the prompt: Giraffe
3) Find all clips within that video that match a giraffe



---

## Tech Stack

### Backend / Core
- **Python**
- **FastAPI** (upload + search API)
- **FFmpeg** (audio extraction, chunking, frame sampling)
- **faster-whisper** (transcribe audio → text)
- **OpenCLIP** (embeddings for text + images in the *same* space)
- **FAISS** (vector similarity search)

### Frontend
- **React** (upload UI + search + video player jump-to-time)

---

## Data Model (per chunk)
Each video is split into chunks. Each chunk stores:

- `chunk_id`
- `video_id`
- `start_time`, `end_time`
- `transcript_text`
- `frame_paths` (or frame ids)
- `text_embedding` (vector)
- `image_embedding` (vector; usually an aggregate over frames)

---

## Method (Initial Draft)

### 1) Upload
User uploads a video file to the backend.

### 2) Chunking
Split the video into fixed time windows, e.g.
- `chunk_len = 30s`
- `overlap = 5s`

Each chunk becomes:
- `chunk_id`, `start_time`, `end_time`

### 3) Extract chunk info

#### 3.1 Transcript (audio → text)
For each chunk:
- Extract chunk audio with FFmpeg
- Transcribe using **faster-whisper**
- Save transcript to the chunk record

#### 3.2 Frames (video → images)
For each chunk:
- Sample frames at a fixed rate (simple version), e.g.
  - 1 frame every 2 seconds
- Save sampled frames (paths or ids)

> Simple implementation: sample frames uniformly.  
> Better later: “keyframe” extraction or scene-change detection (optional).

---

## Embeddings

### Important rule
To compare **text queries** to **frames**, both must be embedded into the same space.

So:
- Transcript chunks use **OpenCLIP text encoder**
- Frames use **OpenCLIP image encoder**
- Query uses **OpenCLIP text encoder**

### 4) Create embeddings (per chunk)

#### 4.1 Text embedding
- Embed `transcript_text` using OpenCLIP **text encoder**
- Result: `text_vec[dim]`

#### 4.2 Image embedding
- Embed each sampled frame using OpenCLIP **image encoder**
- Aggregate into one vector for the chunk (simple):
  - `image_vec = mean(frame_vecs)`

Store:
- `text_vec`
- `image_vec`

---

## Indexing (FAISS)

### 5) Store vectors
Create FAISS indices:
- `faiss_text_index` for transcript vectors
- `faiss_image_index` for image vectors

Also store metadata mapping:
- `faiss_id -> chunk_id -> (video_id, start_time, end_time, transcript_text)`

---

## Query Flow

### 6) Query
User enters prompt text.

#### 6.1 Embed query
Compute:
- `q_vec = OpenCLIP_text_encoder(prompt)`

#### 6.2 Retrieve
- Search `faiss_text_index` with `q_vec` → top-K chunk candidates by transcript match
- Search `faiss_image_index` with `q_vec` → top-K chunk candidates by visual match

#### 6.3 Combine (simple)
Merge the two results by score:
- `final_score = 0.7 * text_score + 0.3 * image_score` (tune later)

Return top-K chunks:
- `(video_id, start_time, end_time, score, transcript_snippet)`

---

## Output
Frontend shows results:
- list of timestamps
- clicking result jumps video player to `start_time`

---

## Notes / Fixes vs the initial message
- Text embeddings should be created with **OpenCLIP** if you want to compare to frames.
- You generally don’t store *all* frames; you sample frames to control cost.
- Query should retrieve from text index and (optionally) image index, then merge.

---

## Next Improvements (Optional)
- Better frame selection (keyframes / scene change)
- Reranking step (cross-encoder or LLM rerank) on top 50 candidates
- Better chunking strategy (adaptive chunking based on pauses/sections)
- Evaluation set (labeled queries → correct timestamps) and metrics (Recall@K)
