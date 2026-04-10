# gadogestor-audio-transcription

Audio transcription microservice for GadoGestor. Receives audio binary via multipart upload and transcribes it using Google Gemini.

## Architecture

This service is called exclusively by `gadogestor-whatsapp`. It never communicates directly with WhatsApp or Meta APIs.

```
gadogestor-whatsapp
  │
  │  1. Resolve media_id → Meta CDN URL  (Meta Graph API, Bearer token)
  │  2. Download audio binary            (Meta CDN, Bearer token)
  │  3. POST /transcribe multipart       (this service)
  ▼
gadogestor-audio-transcription
  │
  │  4. Transcribe with Google Gemini
  └──► { transcript, language, audioDuration }
```

The Bearer token for Meta CDN stays in `gadogestor-whatsapp` and is never forwarded here.

## API Contract

### POST /transcribe

**Request:** `multipart/form-data` with field `audio` (binary audio file)

Supported formats: `mp3`, `mp4`, `wav`, `m4a`, `ogg`, `webm`, `flac` — max 25 MB

**Response 201:**
```json
{
  "transcript": "a vaca 234 está com mastite",
  "language": "pt-BR",
  "audioDuration": 12.3
}
```

| Status | Meaning |
|--------|---------|
| `201` | Transcription successful |
| `400` | Missing `audio` field |
| `413` | File exceeds size limit |
| `502` | Gemini transcription failure |
| `503` | Gemini not configured (`GEMINI_API_KEY` missing) |

### GET /health

```json
{ "status": "ok" }
```

### GET /swagger/*

Swagger UI for interactive API documentation.

## Stack

| Layer | Technology |
|---|---|
| HTTP Server | Go standard `net/http` |
| Transcription | Google Gemini API |
| API Docs | Swagger via `swaggo/swag` |
| Config | Environment variables |

## Getting Started

### Prerequisites

- Go 1.21+
- Google Gemini API key

### Configuration

| Variable | Description | Default |
|---|---|---|
| `GEMINI_API_KEY` | Google Gemini API key | required |
| `GEMINI_MODEL` | Gemini model to use | `gemini-2.5-flash` |
| `ENABLE_TRANSCRIPT_ANALYSIS` | Enable summary/key points/sentiment | `false` |
| `PORT` | HTTP listen port | `8080` |

If `GEMINI_API_KEY` is missing, the server still starts but `POST /transcribe` returns `503 Service Unavailable`.

### Run

```bash
export GEMINI_API_KEY="your-gemini-key"
go run ./cmd/server
```

### Test the API

```bash
# Healthcheck
curl http://localhost:8080/health

# Transcribe an audio file
curl -X POST http://localhost:8080/transcribe \
  -F "audio=@path/to/audio.ogg"
```

## Development

```bash
go test ./...
go vet ./...
```

## Supported Audio Formats

`mp3`, `mp4`, `mpeg`, `mpga`, `m4a`, `wav`, `webm`, `ogg`, `flac`
