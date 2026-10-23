# Processing Pipeline

1. Injest & Validate
2. Probe Metadata
3. Extract 16kHz Audio
4. ASR Transcription (AI)
5. Word Alignment 
6. Build Caption Segments 
7. Analyze Topics & Windows
8. Semantic B-Roll Search (AI)
9. Select & Trim Clips
10. EDL
11. ASS Subtitles
12. FFmpeg Composite Render
13. Validate & Serve

## Stage 1: Accepting & Validating The Uploaded Video

- **Category:** Ordinary Programming 
- **Consumes:** User-Selected Video file from the browser 
- **Operation:** Verifies the file extension, checks the file is above 0 bytes and below a reasonable limit, creates a unique project folder, and saves the file as `raw_input.mp4`
- **Produces:** File Path and Initial `project_state.json`
- **Failure Handling:** Rejects non-video files immediately with a clear error message before touching any processing tools


## Stage 2: Reading Video and Audio Metadata 
- **Category:** Media Tool (`ffprobe`)
- **Consumes:** `raw_input.mp4`
- **Operation:** Inspects the container headers without decoding the whole video. Extracts width, height, frame rate, duration in seconds, video codec, and verifies that an audio stream actually exists
- **Produces:** A `metadata` object inside `project_state.json`
- **Failure Handling:** If no audio Stream is present, halt the pipeline Immediately and alerts the user that "This video has no audio track to transcribe"

## Stage 3: Extracting and Normalising Audio
- **Category:** Media Tool (`ffmpeg`)
- **Consumes:** `raw_input.mp4`
- **Operation:** Strips the Audio Stream and converta the audio track into an uncompressed, single-channel(mono) WAV file sampled at exactly 16000 Hz(16kHz PCM). 
- **Produces:** `audio_16kHz.wav`

## Stage 4 & 5: Transcribing Speech and generating word-Level TimeStamps
- **Category:** AI Inference (`faster-whisper`)  
- **Consumes:** audio_16kHz.wav
- **Operation:** Runs a Voice Activity Detector (VAD) to ignore silent Sections, transcribes spoken words and calculat the exact start and end millisecond of every individusal word using cross-attention alignment
- **Produces:** A raw Array of sentences and indivisual word objects, saved to `project_state.json`
- **Failure Handling:** If the video contains only music or silence, returns an **empty transcript array** rather than hallucinating repeated phrases (***Known as Whisper quick when VAD is turned off***)

## Stage 6: Converting the Transcript into Caption Segments 
- **Category:** Deterministic Algorithm (Python)
- **Consumes:** Array of word-level timestamps - `project_state.json`
- **Operation:** Groups indivisual words into viewer-friendly caption blocks, splitting naturally at puncuation marks ike periods, commas, or pauses longer than 0.4 seconds
- **Produces:** A `captions` list in `project_state.json` where each item has a unique `id, start_time, end_time, text` and the nested `words` array.

## Stage 7 & 8: Topic Analysis and Semantic B-Roll Search 
- **Category:** Deterministic Chunking + AI Embeddings(`Sentence-transformers`)
- **Consumes:** The Transcript text grouped into 5-10 second spoken sentences, plus pre-indexed local B-Roll MetaData File.
- **Operation:** Converts each Spoken Word into a mathematical vector (a list of numbers representing it's meaning) and compares it against the vectors of B-roll descriptions using cosine similiarity... Identifies candidate moments where the similiarity score crosses a strict confidence threshold(eg: greater than 0.65) and ensure minimum cooldown gap, between B-roll cuts so the video dosen't cut too much lol
- **Produces:** A `broll_Placements` array in `project_state.json` listing suggsted `broll_clip_id, timeline_start, timeline_end and confidence_score`

## Stage 9 & 10: Human Review and Building the Edit Decision List (EDL) 
- **Category:**  Frontend UI + State Management
- **Consumes:** `project_state.json`
- **Operation:** The user edits, tweaks captions or wtv he wanna do. Saving Updates to `project_state.json. The JSON file will now be EDl as well, 

## Stage 11: Generating the Styled Subtitle Script 
- **Category:** Deterministic Formatting (Python)
- **Consumes:** The `captions` array and selected visual preset from `project_state.json`, plus video resolution from `metadata`.
- **Operation:** Translates the JSON Captions into an ***Advanced Substation Alpha (`.ass`)*** text file. This defines exact font families, font sizes scaled to videos resolution, border thickens and timing to the centisecond
- **Produces:** `captions.ass`

## Stage 12: Rendering and Compositing the Final Video
- **Category:**  Media tool (`ffmpeg` complex filtergraph)
- **Consumes:** `raw_input.mp4`. Selected B-roll video Files. `captions.ass` and `project_state.json`
- **Operation:** Constructs and executes an FFmpeg filter pipeline that:
    1. Scales and pads/crops every selected B-roll clip to match the exact resolution (1080p) and frame rate of the main talking-head video
    2. Overlays each B-roll clip on top of the talking head video track during it's designated `[timeline_start, timeline_end]` window (keeping the talking head audio tarack plying continously underneath)
    3. Burns the `captions.ass` subtitles onto the topmost visual layer so captions remain visible over both the speaker and the B-roll.
    4. Encodes the final video stream in H.264 (`libx264`) and copies/encodes the audio in AAC.
- **Produces:** `finaExport.mp4`

## Stage 3: Validating the Output and Serving Download
- **Category:**  Media Tool (`ffprobe`) + Backend API
- **Consumes:** `finalExport.mp4`
- **Operation:** Runs a quick `ffprobe` check on teeh output file to verify it exists and has a non zero file size, and it's duration matches the original input video. Marks the Render job as `Completed` and provides download link to the browser. 