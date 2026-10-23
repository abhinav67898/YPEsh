### System Architecture 

### User's Web Broswer (FrontEnd)

- Video Player Preview 
- B-Roll Approval Cards 
- Caption Text/Timings Editor
- Job Progress Poller

> HTTP Requests
 
### Python Backend Service (FastAPI)

- API Endpoints
- Job State Manager

> Read / Writes 

### Local NVMe SSD Storage

- /workshpace -> B-Roll Library, Project assets

> Spawns & Controls

### Processing & AI Workers

- FFprobe / FFmpeg (Media)
- faster-whisper (CPU ASR)
- Sentence-transformers
- ASS Subtitle Generator


