# Material Retrieval and Network Acquisition Audit Report

This report details the project's capabilities for automatically searching for and retrieving materials from the internet and local sources.

### **Provider & Endpoint Table**

This table summarizes all external and internal sources of media assets.

| Provider | Type | Endpoint / Source | Authentication | Method | Rate Limiting / Retry | Result Consumption |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Pexels** | Video | `https://api.pexels.com/videos/search` | `PEXELS_API_KEY` in `Authorization` header | `GET` | Retries up to 3 times on timeout for the API call. No rate limiting implemented in the code. | A single video URL is selected from the response. |
| **Kokoro** | TTS Audio | `kokoro-js` library (local) | N/A | N/A | N/A | An audio stream is generated locally from text. |
| **Whisper** | Subtitles | `@remotion/install-whisper-cpp` (local) | N/A | N/A | N/A | A caption file is generated locally from the TTS audio. |
| **YouTube Audio Library** | Music | Local files in `/static/music/` | N/A | N/A | N/A | A local MP3 file is chosen for the video's background music. |

### **Query Construction Pipeline (Pexels Video)**

The process for generating search queries for video assets is as follows:

*   **1. Initial Input**: The system receives a list of scenes from an external API call (`POST /short-video`). Each scene object contains an array of `searchTerms`.
*   **2. Randomization**: For a given scene, the `searchTerms` array is shuffled to vary the search order and increase the diversity of results across different video generations.
*   **3. Iterative Search**: The system iterates through the shuffled search terms one by one, making a distinct API call for each term. The loop breaks as soon as a suitable video is found.
*   **4. Fallback Mechanism**: If none of the provided `searchTerms` yield a result, the system uses a hardcoded array of generic "joker" terms (`["nature", "globe", "space", "ocean"]`), which is also shuffled and iterated through.
*   **5. API Request**: The final request to the Pexels API includes the chosen search term, the required `orientation` (portrait, landscape), a `size` filter (`medium`), and a request for up to `80` results (`per_page=80`).

### **Download/Cache/Filtering Logic Summary**

*   **Downloading**:
    *   Once a video is selected from the Pexels API results, its URL is downloaded using a standard Node.js `https.get` request.
    *   The video is streamed directly into a temporary file on the local disk.
    *   There is no advanced download logic (e.g., resumable downloads, speed limits).
*   **Caching**:
    *   There is **no persistent caching**. All assets (video, TTS audio) are treated as temporary.
    *   They are downloaded/generated into a temporary directory (`~/.ai-agents-az-video-generator/temp`) for each video creation job.
    *   All temporary files are **deleted** from this directory upon successful rendering of the final video.
*   **Filtering & Safety**:
    *   **De-duplication**: To avoid using the same Pexels clip multiple times within a *single* video, the ID of a used video is added to an `excludeIds` list. This list is then used to filter subsequent searches for that same video project. This does not prevent the same clip from being used in different video projects.
    *   **License**: The code does not perform any license checks. It relies on the Pexels license and the YouTube Audio Library license, which, according to the `README.md`, permits use without attribution.
    *   **NSFW**: There is no explicit NSFW filter implemented in the code. The project is dependent on Pexels' own content safety filters.

### **Material → Remotion Track Mapping**

The following table details how each asset is sourced and mapped to the Remotion timeline.

| Asset Type | Source | Selection Logic | Timeline Integration |
| :--- | :--- | :--- | :--- |
| **B-Roll Video** | Pexels API | Random selection from filtered results based on search term, orientation, and minimum duration. | Downloaded to a temp file, served via a local Express endpoint, and passed as a `video` URL prop to the Remotion component for each scene. |
| **Voiceover Audio** | Kokoro.js (TTS) | Generated locally from the `text` field provided for each scene in the API call. | Saved to a temp file, served locally, and passed as an `audio` URL prop. **The duration of this audio dictates the duration of its corresponding video scene.** |
| **Subtitles** | Whisper (local) | Generated locally from the voiceover audio file. | The generated caption data structure is passed as a `captions` prop to the Remotion component for each scene. |
| **Background Music** | Local Files (`static/music/`) | A random track is selected from the local library. It can be filtered by a `mood` tag if one is provided in the API call. | The chosen music file is passed as a `music` prop to the main Remotion composition. The music plays for the entire duration of the video. |

### **Unresolved Questions & Files to Check**

*   None. The audit of the automated network retrieval process is complete. The flow is self-contained and understood.

### **Reference Links**

*   **Pexels API Documentation**: [https://www.pexels.com/api/documentation/](https://www.pexels.com/api/documentation/)
*   **Kokoro.js Library**: [https://github.com/hexgrad/kokoro](https://github.com/hexgrad/kokoro)
