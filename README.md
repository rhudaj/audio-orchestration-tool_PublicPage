# Streaming Platform

**NOTE**

This is the public version of the private repo titled "audio-orchestration-tool" – to act as a landing page for the public.

It contains only the README of the original repo. If you wish to see the actual code, contact me. 


## Overview

A real-time audio streaming platform with transcription, analysis, and processing capabilities.

## 🚀 Quick Start

### Development
```bash
# Start both backend and frontend
./scripts/start.sh

# Or start individually:
./scripts/start_backend.sh  # Backend only (port 8000)
./scripts/start_frontend.sh # Frontend only (port 3000)
```

## 📁 Project Structure

```
streaming-platform/
├── 📂 backend/              # Python FastAPI backend
│   ├── 🐍 main.py          # Main FastAPI application
│   ├── 📋 requirements.txt # Python dependencies
│   ├── 📂 db/local/        # Local file-based database
│   ├── 📂 pipeline/        # Audio processing pipeline
│   └── 📂 util/            # Utility functions
├── 📂 frontend/            # React/Next.js frontend
│   ├── 📦 package.json    # Node.js dependencies
│   └── 📂 src/            # Source code
├── 📂 deployment/         # Docker & deployment config
│   ├── 🐳 Dockerfile     # Container definition
│   ├── 📄 render.yaml    # Render deployment config
│   └── 📖 DEPLOYMENT.md  # Deployment guide
└── 📂 scripts/           # Automation scripts
    ├── 🚀 start.sh       # Start both servers
    ├── 🔧 start_backend.sh
    ├── 🎨 start_frontend.sh
    └── 📦 deploy.sh      # Deploy to production
```

## 🎯 Features

- **Real-time Audio Streaming**: Live audio capture and processing
- **Transcription**: Convert audio to text using custom AI services
- **Analysis**: Process transcripts for insights and topics
- **File Upload**: Support for various audio formats
- **Local Database**: File-based storage (no external DB needed)
- **Container Ready**: Docker support for easy deployment

## 🔄 Audio Processing Pipeline

The platform processes audio through a sophisticated multi-stage pipeline that handles transcription, analysis, and structured content extraction.

### Pipeline Architecture

```
Audio Input
    ↓
Audio Processing (chunking & formatting)
    ↓
Transcription (Deepgram) → Words with timing & speaker info
    ↓
Speaker Grouping → Organized by speaker turns
    ↓
Transcript Analysis
    ├─ Topic Segmentation (detect topic boundaries)
    └─ Topic Summarization (extract insights & summaries)
    ↓
Structured Sections (with summaries, key points, timestamps)
```

### Pipeline Steps

#### 1. **Audio Processing**
Incoming audio is chunked into small segments (0.25 seconds by default) and formatted to standard specifications for consistent processing.

#### 2. **Transcription (Deepgram)**
- **Model**: Nova-2 (high-accuracy, real-time speech recognition)
- **Features**:
  - Punctuation support
  - Speaker diarization (identifies different speakers)
  - Language detection or language-specific processing
  - Utterance endpoint detection (recognizes natural pauses)
- **Output**: `Transcript` objects containing:
  - Words with character offsets
  - Start/end timestamps for each word
  - Speaker identification
  - Confidence levels

#### 3. **Speaker Transcript Handling**
Groups incoming transcripts by speaker turns to handle speaker changes and flush buffers when needed.

#### 4. **Transcript Analysis** 🔎

This is where the interesting insights are extracted. The analysis has two key components:

##### **Step A: Topic Segmentation**

Topic segmentation identifies natural boundaries in the audio where the topic or discussion subject changes. This is the foundation for breaking continuous speech into meaningful sections.

**Methods & Algorithm:**

The segmentation uses **semantic similarity analysis** with BERT embeddings:

1. **Embedding Generation**:
   - Each word chunk (configurable size, default: 40 words) is converted to a semantic embedding using a sentence transformer model
   - Models available: `all-MiniLM-L12-v2` (default, balanced), `paraphrase-multilingual-MiniLM-L12-v2` (multilingual)

2. **Semantic Drift Detection**:
   - Continuously calculates cosine similarity between adjacent chunks
   - When similarity drops below a threshold, a topic boundary is detected
   - **Granularity control**: A setting from 0-1 controls topic generality
     - Higher values (e.g., 0.9) = more general topics (fewer segments)
     - Lower values (e.g., 0.5) = more granular topics (more segments)

3. **Buffering & Context**:
   - Maintains a sliding window of past words (buffer_size: default 800 words)
   - Uses context for more accurate embeddings
   - Removes stop words if enabled for cleaner semantic signals

4. **Confirmation Mechanism** (Anti-flickering):
   - **Immediate confirmation** (delay=0): Segments are confirmed as soon as detected
   - **Delayed confirmation** (delay>0): Segments must be detected in X consecutive recomputations before confirmation
   - Recomputation interval: Controlled by `recompute_interval` (default: 5 chunks)
   - **Purpose**: Prevents false positives from minor semantic fluctuations

5. **Segment Management**:
   - Tracks segment positions and timestamps
   - Maintains ordered list of segment boundaries
   - Outputs segments as `(start_time, end_time)` tuples with millisecond precision

**Configuration Parameters:**
- `bert_model_name`: Semantic embedding model
- `word_chunk_size`: Words per chunk (30-50 typical)
- `buffer_size`: Context window size in words
- `granularity`: Topic specificity (0.5-0.9 range)
- `recompute_interval`: Recompute frequency (1-10 chunks)
- `segment_confirmation_delay`: Confirmation threshold (0-3 typical)
- `remove_stop_words`: Clean embeddings for better signals

##### **Step B: Topic Summarization**

Once topics are segmented, each section is analyzed to extract meaningful information:

1. **Speaker Grouping**: Transcripts within the segment are organized by speaker
2. **LLM Analysis** (using Azure OpenAI GPT-4):
   - Generates section name (title)
   - Creates concise summary (1-3 sentences)
   - Extracts key points (3-5 bullet points)
3. **Fine-tuned Prompts**: Uses prompt engineering for consistent output format

**Configuration:**
- `azure_deployment_name`: GPT model version (e.g., "gpt-4.1")
- `temperature`: Creativity level (0.2 = precise/factual, recommended)
- `max_tokens`: Maximum output length

### Real-Time Processing

Both segmentation and summarization work in **streaming mode**:
- Analyzed transcripts are immediately processed
- New segments trigger summarization callbacks
- Frontend receives updates in real-time via WebSocket
- No need to wait for entire audio to complete

## Project Structure

```
.
├── backend/                  # Backend FastAPI application
│   ├── app/                  # Main application code
│   │   ├── main.py           # FastAPI application and API endpoints
│   │   ├── transcribe.py     # Audio transcription using Deepgram
│   │   ├── process_json.py   # JSON processing and AI analysis
│   │   ├── stream_record.py  # Stream recording functionality
│   │   └── ...               # Other backend modules
│   ├── processed_texts/      # Directory for processed transcripts
│   ├── transcriptions/       # Directory for transcription results
│   ├── recordings/           # Directory for recorded audio files
│   └── requirements.txt      # Python dependencies
├── frontend/                 # Frontend React application
│   ├── src/                  # Source code
│   │   ├── components/       # React components
│   │   │   ├── AudioProcessing.tsx # Main audio processing component
│   │   │   ├── AIAnalysisPanel.tsx # AI analysis panel
│   │   │   ├── TranscriptionPanel.tsx # Transcription panel
│   │   │   └── ...           # Other components
│   │   └── ...               # Other frontend code
│   └── package.json          # Node.js dependencies
├── .env                      # Environment variables
└── start.sh                  # Script to start both backend and frontend
```

## Features

### Recording
- Record radio streams from URLs
- Save recordings with timestamps
- View recording history

### Transcription
- Transcribe recordings using Deepgram
- View transcription results
- Download transcripts

### AI Analysis
- Analyze transcripts using Azure OpenAI
- Break down content into logical sections
- Identify content types (news, weather, commercials, etc.)
- Generate summaries and key points

## Getting Started

### Prerequisites
- Python 3.8+
- Node.js 16+
- npm or yarn

### Installation

1. Clone the repository
2. Install backend dependencies:
   ```
   cd backend
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```

3. Install frontend dependencies:
   ```
   cd frontend
   npm install
   ```

4. Set up environment variables:
   - Create a `.env` file in the backend directory with:
     ```
     DEEPGRAM_API_KEY=your_deepgram_api_key
     AZURE_API_KEY=your_azure_api_key
     AZURE_API_BASE=your_azure_api_base
     AZURE_API_VERSION=2024-02-15-preview
     AZURE_DEPLOYMENT_NAME=your_deployment_name
     ```

### Running the Application

Use the provided start script to run both the backend and frontend:

```
./start.sh
```

Or run them separately:

Backend:
```
cd backend
source venv/bin/activate  # On Windows: venv\Scripts\activate
cd app
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

Frontend:
```
cd frontend
npm run dev
```

## API Endpoints

### Backend API

- `GET /api/recordings` - List all recordings
- `GET /api/recordings/{filename}` - Get a specific recording
- `POST /api/record` - Record a stream
- `POST /api/transcribe` - Transcribe a recording
- `GET /api/transcripts` - List all transcripts
- `POST /api/analyze-transcript` - Process a transcript with AI analysis

## 🛠️ Technology Stack

### Backend
- **Python 3.11+**
- **FastAPI** - Modern web framework
- **FFmpeg** - Audio processing
- **Deepgram** - Speech-to-text
- **OpenAI** - AI analysis
- **Local File DB** - No external database required

### Frontend
- **React/Next.js** - UI framework
- **TypeScript** - Type safety
- **Tailwind CSS** - Styling
- **Bun/npm** - Package management

### Deployment
- **Docker** - Containerization
- **Render.com** - Cloud deployment
- **GitHub Actions** - CI/CD (optional)

## 🔧 Development Setup

### Prerequisites
- Python 3.11+
- Node.js 18+
- Bun (recommended) or npm
- FFmpeg (for local development)

### Backend Setup
```bash
cd backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python main.py
```

### Frontend Setup
```bash
cd frontend
bun install  # or npm install
bun run dev  # or npm run dev
```

## 🐳 Docker Development

```bash
# Build and run with Docker Compose
cd deployment
docker-compose up --build

# Or build container manually
docker build -f deployment/Dockerfile -t streaming-platform .
docker run -p 8000:8000 streaming-platform
```

## 🌐 API Endpoints

- `GET /health` - Health check
- `POST /upload` - Upload audio files
- `WS /ws` - WebSocket for real-time streaming
- `GET /docs` - API documentation (Swagger)

## 📊 Database Structure

Local file-based database in `backend/db/local/`:
```
db/local/
├── recordings/           # Audio recordings
├── transcripts/         # Transcription results
├── analyzed_transcripts/ # Analysis results
└── audio_clips/         # Processed clips
```

**Note**: Uploaded files are now stored temporarily in `backend/tmp/uploads/` and are automatically cleaned up after pipeline processing.

## 🚀 Deployment

### Render.com (Recommended)
1. Push code to GitHub
2. Connect repository to Render
3. Use `deployment/render.yaml` configuration
4. Deploy automatically

See `deployment/DEPLOYMENT.md` for detailed instructions.

## 🔐 Environment Variables

### Development
```bash
ENVIRONMENT=development
DB_TYPE=local
```

### Production
```bash
ENVIRONMENT=production
DB_TYPE=local
LOG_LEVEL=INFO
```

## 🧪 Testing

```bash
# Backend tests
cd backend
python -m pytest

# Test pipeline cancellation
python test_pipeline_cancellation.py
```

## 📖 Documentation

- `deployment/DEPLOYMENT.md` - Complete deployment guide
- `deployment/README.md` - Deployment files overview
- `scripts/README.md` - Scripts documentation
- `/docs` endpoint - API documentation

## 🤝 Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open Pull Request

## 📄 License

This project is licensed under the MIT License.

## 🆘 Support

- Check the documentation in `/deployment/DEPLOYMENT.md`
- Review health endpoint: `http://localhost:8000/health`
- Check logs for error details
- Open an issue for bugs or feature requests
