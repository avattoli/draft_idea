# Video Segment Search

## Goal
Upload a video, then search it with a text prompt to return the most relevant clips.

Example:
1) Upload a video taken at the zoo  
2) Enter the prompt: Giraffe  
3) Return clips (timestamps) where a giraffe appears  

---

## Tech Stack

### Backend / Core
- **Python**
- **FastAPI** (upload + search API)
- **FFmpeg** (chunking, frame sampling, audio extraction)
- **faster-whisper** (audio → transcript)
- **OpenCLIP** (text + image embeddings in same space)
- **FAISS** (vector similarity search)

### Frontend
- **React**

---

## Data Model

### Chunk (optional but useful for transcript search)
- `chunk_id`
- `video_id`
- `start_time`
- `end_time`
- `transcript_text`

### Frame (primary retrieval unit)
- `frame_id`
- `video_id`
- `timestamp`
- `frame_path`
- `chunk_id` (optional reference)

---

## Method

### 1) Upload
User uploads a video file.

### 2) Chunking (for transcript support)
Split video into fixed windows:
- `chunk_len = 30s`
- `overlap = 5s`

Chunks are mainly used for transcript-based search.

### 3) Frame Extraction (primary search unit)
- Sample frames at fixed rate (e.g., 1 frame every 1–2 seconds)
- Store each frame with:
  - `video_id`
  - `timestamp`
  - `frame_path`

---

## Embeddings

### Important rule
To compare text prompts with frames, both must use **OpenCLIP**.

### 4.1 Text Embeddings (optional)
- Embed `transcript_text` using OpenCLIP text encoder
- Store one vector per chunk

### 4.2 Frame Embeddings (core)
- Embed each frame using OpenCLIP image encoder
- Store one vector per frame

---

## Indexing (FAISS)

### 5) Store vectors

Create two indices:

- `faiss_text_index`
  - Stores transcript chunk vectors
  - `text_faiss_id -> chunk metadata`

- `faiss_image_index`
  - Stores frame vectors
  - `image_faiss_id -> (video_id, timestamp, chunk_id)`

---

## Query Flow

### 6) Query

User enters prompt text.

#### 6.1 Embed query
q_vec = OpenCLIP_text_encoder(prompt)

#### 6.2 Retrieve

- Search `faiss_image_index` using `q_vec`
- Get top-N matching frames (timestamps)

(Optional)
- Search `faiss_text_index` for transcript matches

#### 6.3 Convert frames → clips

- Sort retrieved frame timestamps
- Merge nearby timestamps (e.g., within 5–10 seconds)
- Convert merged ranges into clips:
  - `(video_id, start_time, end_time)`

Return top-K clips.

---

## Output

Frontend shows:
- List of matching clips
- Clicking a clip jumps video player to `start_time`


