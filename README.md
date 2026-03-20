# Semantic-Search-NOC

**Executive Summary:** *Semantic-Search-NOC* is an AI-driven semantic search system for National Occupational Classification (NOC) codes. It replaces keyword lookups in static documents with natural-language querying. Given a job title or description (typed or spoken), the system uses a pretrained SBERT model to embed the query and an Elasticsearch KNN index to retrieve the most semantically relevant occupation codes (NCO/NOC) along with confidence scores. It also supports voice input by transcribing audio with Google’s Gemini API【97†L465-L472】【85†L340-L347】. The project provides a Flask-based API and (optionally) a web frontend for users to search occupation codes easily without knowing exact classification hierarchies. 

| **Module/Script**        | **Purpose**                                                                                                                      |
|--------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| `app.py`                 | Main Flask application: defines REST endpoints for text search (`/submit_text`), voice search (`/submit_voice`), feedback and admin functions【102†L745-L753】【103†L1018-L1026】. It serves the frontend and handles user input, output, and feedback logging. |
| `search_nco.py`          | Core semantic search logic. Loads a SentenceTransformer model and connects to Elasticsearch; encodes user queries into vectors and performs a KNN search on the “nco2015” index【97†L465-L472】. Returns top-k matches (occupation titles and codes) with confidence scores and hierarchical info. |
| `index_nco.py`           | Index-building script. Reads `dataset/nco2015_full.csv`, encodes each occupation title into a dense vector (using SBERT), and bulk-loads these into an Elasticsearch index named `nco2015`【100†L415-L423】【100†L471-L480】. Re-creates the index from scratch for fresh data. |
| `gemini.py`              | Voice transcription/translation. Uploads a WAV file to Google’s Gemini API, transcribes speech, and translates it into English text【85†L340-L347】. The returned transcript is cleaned (non-alphanumerics removed) and used for semantic search. |
| `performance.py`         | Post-processing of logged results. Contains utility functions to load and clean feedback and voice-history JSON files. For example, `process_feedback_data()` extracts (`user_input`, `title`, `NCO2015`, `confidence`, `feedback`) from feedback logs【88†L609-L617】; `process_data()` cleans transcription logs for dashboard/charting. |
| *data* folders (`dataset/`, `feedback_data/`) | **Data and logs.** `dataset/` holds CSV files for the NOC hierarchy (division, subdivision, group, family, and the full NCO2015 list) used by the indexer. `feedback_data/` stores JSON history of user queries and feedback (populated at runtime)【102†L781-L790】【103†L1018-L1026】. |

## Features and Data Flow

- **Semantic Search** – Uses the *paraphrase-multilingual-mpnet-base-v2* model from SentenceTransformers. A free-text query is encoded into a vector, then Elasticsearch KNN search returns matching occupations with similarity scores【97†L465-L472】【97†L509-L517】. Results include the occupation title, NCO2015 code, legacy NCO2004 code, confidence (score), and the hierarchical division/subdivision/group/family (built via `parse_hierarchy`)【97†L469-L477】【94†L509-L517】.
- **Voice Input** – Users can upload a short audio clip (wav). The Flask app saves the file and calls `gemini.transcribe_and_translate()`, which uses Google Gemini to transcribe and translate the speech to English【85†L340-L347】. This text is then passed to `semantic_search()` as above.
- **Feedback Loop** – The `/feedback` endpoint allows collecting user feedback (“good”/“bad”) on specific results. Feedback entries are saved in `feedback_data/history.json`, which can be later retrieved or processed for model refinement. Similarly, voice query logs (timestamp, transcription, results) are saved to `feedback_data/voice_history.json`.
- **Admin Functions** – The `/add_entry` endpoint lets an admin add new entries to the CSV files (e.g. a new occupation code or classification). After adding, the code automatically re-runs `index_nco.indexing()` to rebuild the Elasticsearch index with updated data【103†L1018-L1026】.

```mermaid
flowchart LR
    U[User] -->|Text or Voice Query| API[Flask API (app.py)]
    API -->|If Voice →| Gemini[Google Gemini API] 
    Gemini --> Transcription[Transcribed English Text]
    Transcription --> Semantic[Semantic Search (search_nco.py)]
    API -->|If Text →| Semantic
    Semantic -->|Vector Query →| ES[Elasticsearch Index (NCO2015)]
    ES -->|Hits →| Semantic
    Semantic --> API
    API -->|JSON Results| U
```

## Installation and Setup

1. **Prerequisites:** Install a recent Python 3.x. Ensure an Elasticsearch server is running (default http://localhost:9200). We recommend Elasticsearch 7.x or higher; for example, run [Elasticsearch Docker image](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html) or install natively.   
2. **Clone the repository:**  
   ```bash
   git clone https://github.com/Nemaleshh/Semantic-Search-NOC.git
   cd Semantic-Search-NOC
   ```  
3. **Install dependencies:** There is no provided `requirements.txt`, so manually install the needed packages. For example:  
   ```bash
   pip install flask flask-cors elasticsearch sentence-transformers torch google-generativeai pandas
   ```  
   - *Note:* SentenceTransformers will pull in PyTorch or TensorFlow; PyTorch (CPU/GPU) is recommended for performance.  
4. **Environment Variables:** The Gemini API key is currently hardcoded in `gemini.py`, but you should store it as an environment variable or config. For example:  
   ```bash
   export GEMINI_API_KEY="your-google-gen-api-key"
   ```  
   Then modify `gemini.configure(api_key=...)` to `api_key=os.getenv("GEMINI_API_KEY")`.  
5. **Prepare dataset:** The `dataset/` folder already contains CSVs (`division.csv`, `subdivision.csv`, `group.csv`, `family.csv`, `nco2015_full.csv`). Ensure these files are present; they define the occupation hierarchy and titles.  
6. **Build the index:** Run the indexer to create the Elasticsearch index:  
   ```bash
   python index_nco.py
   ```  
   This reads `dataset/nco2015_full.csv`, encodes titles with SBERT, and populates the `nco2015` index【100†L415-L423】【100†L471-L480】. If successful, you should see “Indexing NCO data... ✅ Indexing complete.”  
7. **Run the server:** Start the Flask app:  
   ```bash
   python app.py
   ```  
   By default, it will listen on port 5000 (`http://localhost:5000`). If you have a frontend, place its build in the `frontend/` folder (not included here).  

**Directory Layout After Setup:**  
```
Semantic-Search-NOC/
├── app.py                  # Flask app (API endpoints)
├── index_nco.py           # Index building script
├── search_nco.py          # Semantic search module
├── gemini.py              # Voice transcription module
├── performance.py         # Data-processing utilities
├── dataset/               # CSVs for NCO hierarchy & titles
│   ├── division.csv
│   ├── subdivision.csv
│   ├── group.csv
│   ├── family.csv
│   └── nco2015_full.csv
├── feedback_data/         # (empty initially) Feedback & query logs
│   ├── history.json
│   └── voice_history.json
└── [venv/] (optional)     # Python virtual environment (if created)
```  

## Usage Examples

- **Text Search (via API):** Submit a JSON POST to the `/submit_text` endpoint:  
  ```bash
  curl -X POST http://localhost:5000/submit_text \
       -H "Content-Type: application/json" \
       -d '{"text": "Tailor sewing factory"}'
  ```  
  *Sample Response:*  
  ```json
  {
    "query": "Tailor sewing factory",
    "embedding_time": 0.024,
    "results": [
      {"title": "Sewing Machine Operator", "NCO2015": "8153.0100", "NCO2004": "8153.0100", "confidence": 0.87, "hierarchy": {"division":"...","subdivision":"...","group":"...","family":"..."}},
      {"title": "Hand Embroiderer",     "NCO2015": "7318.0100", "NCO2004": "7318.0100", "confidence": 0.65, "hierarchy": {...}},
      // ...
    ]
  }
  ```  
  (The actual titles/codes will vary; above is illustrative.) The `confidence` is the Elasticsearch score (cosine similarity) rounded to 4 decimals【97†L509-L517】.  

- **Voice Search:** Use `/submit_voice` with a WAV file upload (field name `file`). For example, with `curl`:  
  ```bash
  curl -X POST http://localhost:5000/submit_voice \
       -F "file=@sample.wav" 
  ```  
  The server will respond with `{ "status": "success", "transcription": "...", "results": [...] }`. Internally, the voice clip is saved, sent to Gemini, then fed to `semantic_search()`【102†L781-L790】【102†L833-L842】.  

- **Feedback & History:** You can retrieve user feedback or voice query logs via:  
  - `GET /feedback-data` → returns processed feedback history (list of `{user_input, title, NCO2015, confidence, feedback}`)【88†L609-L617】.  
  - `GET /voice-data` → returns voice query logs (`timestamp`, `transcription`, top results) for performance analysis【88†L503-L512】.  

- **Adding Entries:** To append a new occupation or classification, send a POST to `/add_entry` with JSON `{"type":"division","code":"10","name":"New Division"}` (where type is `division`, `subdivision`, `group`, `family`, or `occupation`). The code updates the appropriate CSV and **rebuilds** the Elasticsearch index automatically【103†L999-L1008】.  

## Troubleshooting and Tips

- **Elasticsearch Connection:** If `index_nco.py` fails, ensure Elasticsearch is running on `localhost:9200`. You may need to configure ES memory or docker. Start Elasticsearch before indexing and before running the app (see [semantic-search instructions](https://github.com/Nemaleshh/semantic-search) for example of starting ES【107†L373-L380】).  
- **Gemini API:** Without a valid Gemini API key, `/submit_voice` will error. Set and configure your `GEMINI_API_KEY` as described.  
- **Missing Frontend:** The code assumes a `frontend/` directory for static files (e.g. `index.html`)【102†L739-L743】. If no UI is provided, you can interact via the API.  
- **Duplicates:** The admin `/add_entry` checks for duplicate codes and will reject if found【103†L989-L1000】.  
- **Performance:** Indexing can be slow for large datasets. It encodes text with SBERT (a transformer model) in batches of 200【100†L495-L503】. A GPU will accelerate this. Likewise, Elasticsearch (especially with dense vectors) benefits from adequate RAM (>=8-16GB). For a few thousand occupations, a modern laptop suffices, but scale up if needed.  

## Performance Considerations

- **Indexing:** The SBERT model `paraphrase-multilingual-mpnet-base-v2` has a vector dimension (`dims`) that is determined at runtime【100†L425-L434】. Encoding thousands of titles can take minutes. You can reduce batch size or use GPU to speed up.  
- **Query Latency:** Vector search in ES is fast (milliseconds) for top-k results. The returned JSON also includes the time to encode the query (`embedding_time`) in seconds【97†L523-L529】.  
- **Hardware:** For best performance, use a machine with at least 16GB RAM to run Elasticsearch. A CPU with multiple cores or a GPU is recommended for SBERT. Low-resource devices may run slowly.  

## Contribution and Development

Contributions are welcome! As this repo has no formal license or contribution guide, please fork the project and submit pull requests for improvements. Suggestions for future extensions include: *fine-tuning the embedding model on domain data*, *adding multilingual support*, or *optimizing the Elasticsearch schema*. Before contributing, ensure your code passes any local tests (no tests provided; verify functionality manually). You can also log issues on GitHub if bugs are found. 

## License

No license file is provided in this repository. Use and modification are at your own discretion. (A license such as MIT or Apache could be added to clarify permissions.) 

## Contact / Maintainer

*Maintained by Nemaleshwar H.* (GitHub: [Nemaleshh](https://github.com/Nemaleshh)). For questions or issues, you may file GitHub issues or reach out via the profile. 

**Sources:** Project code and documentation (see `app.py`, `search_nco.py`, `index_nco.py`, `gemini.py`)【97†L465-L472】【100†L415-L423】【85†L340-L347】【94†L509-L517】【88†L609-L617】, which detail architecture, algorithms, and data handling. This README is a comprehensive analysis of the repository structure and functionality.
