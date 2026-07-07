# ✨ Youtube+

**Enhance discovery of YouTube contents.**

Youtube+ is a [Streamlit](https://streamlit.io/) web app that finds videos *semantically similar* to a seed video. Given a YouTube video URL, it extracts keywords from the video's title and description, queries the YouTube Data API for candidate videos, downloads their captions, embeds the captions with a sentence encoder, and ranks every candidate by cosine similarity to the seed video. This helps gauge how original a video is and surfaces related content that keyword search alone would miss.

## How it works

```
                        ┌──────────────────────────────────────────────┐
                        │            streamlit_app.py (UI)             │
                        │  video URL + API key in → ranked grid out    │
                        └──┬──────────┬──────────┬──────────┬──────────┘
                           │          │          │          │
             metadata via  │          │          │          │  seed-video info
                pytube     │          │          │          │  & captions
                           ▼          ▼          ▼          ▼
                    ┌───────────┐ ┌─────────┐ ┌─────────┐ ┌──────────────────────┐
                    │service.py │ │feature. │ │process. │ │semantic_similarity   │
                    │ build YT  │ │py       │ │py       │ │(_lite).py            │
                    │ API client│ │ keyword │ │ clean & │ │ caption → embedding  │
                    └─────┬─────┘ │ extract │ │ shape   │ └──────────────────────┘
                          │       │ (KeyBERT│ │ data    │
                          ▼       └─────────┘ └────┬────┘
                    ┌────────────┐                 │
                    │ingestion.py│◄────────────────┘
                    │ YT Data API│   process.py calls ingestion.getVideoDetail
                    │ search/    │   and fetches captions via
                    │ videos/    │   youtube-transcript-api
                    │ channels   │
                    └────────────┘
```

### Pipeline (what happens on one run)

1. **Input** — the user pastes a YouTube Data API key and a seed video URL into the Streamlit UI. `pytube` fetches the seed video's title, description, and IDs; `youtube-transcript-api` verifies English captions exist.
2. **Description cleaning** — `src/process.py::process_description` strips call-to-action text, URLs, and @-mentions from the description.
3. **Keyword extraction** — `src/feature.py` uses **KeyBERT** with a `KeyphraseCountVectorizer` on top of a local **SentenceTransformer** model (`all-mpnet-base-v2`, falling back to `all-MiniLM-L6-v2`) to suggest query keywords.
4. **Ingestion** — `src/ingestion.py` calls the **YouTube Data API v3** (client built in `src/service.py`) to search videos by the chosen keywords (50 per page, 2–10 pages) and optionally fetch videos related to the seed video.
5. **Processing** — `src/process.py::processVideoIds` batch-fetches video details (stats, topics, hashtags, recording location) and pulls each video's transcript with `youtube-transcript-api`, translating non-English captions to English. `videoDetails_df` shapes everything into pandas DataFrames.
6. **Embedding & similarity** — the caption text is embedded by `src/semantic_similarity.py` if available, otherwise `src/semantic_similarity_lite.py` (Google's **Universal Sentence Encoder Lite v2** on TensorFlow + SentencePiece, loaded from `model/`). Similarity to the seed video is the inner product of the embeddings.
7. **Output** — results are shown in an interactive, sortable **AgGrid** table (title, view/like/comment counts, similarity %, clickable URL) and can be exported as a multi-sheet Excel workbook (similarity, stats, locations, hashtags, topics, captions) via **XlsxWriter**.

## Software components

| Component | File(s) | Role |
|---|---|---|
| Web UI / orchestrator | `streamlit_app.py` | Streamlit front end; wires all modules together and renders results with `streamlit-aggrid` |
| YouTube API client | `src/service.py` | Builds the `googleapiclient` YouTube Data API v3 service and validates the API key |
| Data ingestion | `src/ingestion.py` | Keyword search, related-video search, video/channel detail lookups against the YouTube Data API; recent channel videos via `pytube` |
| Data processing | `src/process.py` | Description/caption cleaning, hashtag & duration parsing, transcript fetching/translation, DataFrame assembly |
| Keyword extraction | `src/feature.py` | KeyBERT + keyphrase vectorizer over a local SentenceTransformer model |
| Embedding (lite) | `src/semantic_similarity_lite.py` | Universal Sentence Encoder Lite v2 (TensorFlow 1.x graph mode + SentencePiece) — the fallback encoder used in low-memory deployments |
| Local models | `model/` | `all-MiniLM-L6-v2` (SentenceTransformer for KeyBERT), `universal-sentence-encoder-lite_2` + `universal_encoder_8k_spm.model` (caption embeddings) |
| Deployment | `Procfile`, `setup.sh`, `runtime.txt` | Heroku-style deployment: `setup.sh` writes the Streamlit server config, then `streamlit run streamlit_app.py` on Python 3.9.13 |

> Note: `streamlit_app.py` first tries to import `src.semantic_similarity` (a full Universal Sentence Encoder) and falls back to `src/semantic_similarity_lite.py`, which is what ships in this repository.

## Getting started

### Prerequisites

- Python 3.9 (see `runtime.txt`)
- A [YouTube Data API v3 key](https://developers.google.com/youtube/v3/getting-started)

### Run locally

```bash
pip install -r requirements.txt
streamlit run streamlit_app.py
```

Open the app in your browser, paste your API key, and enter a video URL (the video must have English closed captions).

### Deploy

The repo is ready for Heroku-style platforms:

```
web: sh setup.sh && streamlit run streamlit_app.py
```

`setup.sh` generates `~/.streamlit/config.toml` (headless mode, `$PORT` binding); `runtime.txt` pins Python 3.9.13. CPU-only TensorFlow (`tensorflow-cpu`) is used to keep the slug small.

## Key dependencies

- **streamlit** / **streamlit-aggrid** — UI and interactive results grid
- **google-api-python-client** — YouTube Data API v3
- **pytube** — seed video/channel metadata without API quota
- **youtube-transcript-api** — captions and translations
- **keybert** + **keyphrase-vectorizers** + **sentence-transformers** — keyword suggestion
- **tensorflow-cpu** + **tensorflow-hub** + **sentencepiece** — Universal Sentence Encoder Lite embeddings
- **pandas** / **numpy** / **XlsxWriter** — data shaping, similarity math, Excel export

## License

Apache License 2.0 — see [LICENSE.txt](LICENSE.txt).
