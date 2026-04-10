# gadogestor-audio-transcription

Microsserviço de transcrição de áudio para o GadoGestor. Recebe arquivos de áudio via upload multipart e transcreve usando o Google Gemini. Opcionalmente enriquece a transcrição com resumo, pontos-chave e sentimento.

## Índice

- [Papel no sistema](#papel-no-sistema)
- [Fluxo de integração](#fluxo-de-integração)
- [API](#api)
- [Stack](#stack)
- [Configuração](#configuração)
- [Execução local](#execução-local)
- [Docker](#docker)
- [Testes](#testes)
- [Formatos suportados](#formatos-suportados)

---

## Papel no sistema

Este serviço é chamado exclusivamente pelo [`gadogestor-whatsapp`](https://github.com/ginga-tech/gadogestor-whatsapp). Ele não se comunica diretamente com a API do WhatsApp (Meta) — jamais recebe nem armazena credenciais Meta.

**Responsabilidade única:** receber um binário de áudio e devolver o texto transcrito.

---

## Fluxo de integração

```
gadogestor-whatsapp
  │
  │  1. Recebe webhook Meta com AudioMediaId
  │
  │  2. GET https://graph.facebook.com/v21.0/{mediaId}
  │     Authorization: Bearer {AccessToken}          ← token Meta fica aqui
  │     → { "url": "https://cdn.whatsapp.net/...", "mime_type": "audio/ogg" }
  │
  │  3. GET {cdn_url}
  │     Authorization: Bearer {AccessToken}          ← token Meta fica aqui
  │     → Stream (binário do áudio)
  │
  │  4. POST http://localhost:8082/transcribe
  │     Content-Type: multipart/form-data
  │     field "audio" = binário
  │                                                  ← nenhum token Meta enviado
  ▼
gadogestor-audio-transcription
  │
  │  5. Recebe multipart, valida tamanho e tipo
  │  6. Envia binário ao Google Gemini para transcrição
  │  7. Opcionalmente gera resumo/pontos-chave/sentimento
  │
  └──► { "transcript": "...", "language": "pt-BR", "audioDuration": 12.3 }
```

O Bearer token da Meta **fica no `gadogestor-whatsapp`** e nunca é repassado.

---

## API

### POST /transcribe

Transcreve um arquivo de áudio.

**Request**

```
Content-Type: multipart/form-data
field: audio  (binário do arquivo de áudio)
```

**Response 201 — sucesso**

```json
{
  "audioFilename": "msg-abc123.ogg",
  "fileSizeBytes": 48320,
  "transcript": "a vaca 234 está com mastite, apliquei oxitetraciclina",
  "language": "pt-BR",
  "audioDuration": 12.3,
  "summary": "Relato de tratamento de mastite na vaca 234.",
  "keyPoints": ["mastite", "vaca 234", "oxitetraciclina"],
  "sentiment": "neutral",
  "createdAt": "2026-04-10T14:30:00Z"
}
```

Os campos `summary`, `keyPoints` e `sentiment` são opcionais e preenchidos somente quando `ENABLE_TRANSCRIPT_ANALYSIS=true`. O serviço retorna a transcrição mesmo que o enriquecimento falhe.

**Códigos de status**

| Status | Significado |
|--------|-------------|
| `201` | Transcrição concluída |
| `400` | Campo `audio` ausente no multipart |
| `413` | Arquivo excede o limite de tamanho (padrão: 25 MB) |
| `500` | Erro interno |
| `502` | Falha do Gemini na transcrição |
| `503` | Gemini não configurado — `GEMINI_API_KEY` ausente |

---

### GET /health

Verifica se o serviço está no ar.

**Response 200**
```json
{ "status": "ok" }
```

---

### GET /swagger/*

Interface Swagger UI para explorar e testar a API interativamente.

---

## Stack

| Camada | Tecnologia |
|---|---|
| HTTP Server | Go `net/http` padrão |
| Transcrição | Google Gemini API (`gemini-2.5-flash` por padrão) |
| Enriquecimento (opcional) | Google Gemini API |
| Documentação | Swagger via `swaggo/swag` |
| Configuração | Variáveis de ambiente |

---

## Configuração

| Variável | Descrição | Padrão |
|---|---|---|
| `GEMINI_API_KEY` | Chave da API do Google Gemini | **obrigatória** |
| `GEMINI_MODEL` | Modelo Gemini a usar | `gemini-2.5-flash` |
| `ENABLE_TRANSCRIPT_ANALYSIS` | Ativa resumo, pontos-chave e sentimento | `false` |
| `PORT` | Porta HTTP de escuta | `8080` |
| `RAILWAY_PUBLIC_DOMAIN` | Domínio público (Railway) — usado no Swagger | — |

Se `GEMINI_API_KEY` estiver ausente, o servidor sobe normalmente mas `POST /transcribe` retorna `503 Service Unavailable`. Isso evita restart loops em containers.

---

## Execução local

### Pré-requisitos

- Go 1.21+
- Google Gemini API key

### Rodar

```bash
export GEMINI_API_KEY="sua-chave-gemini"
go run ./cmd/server
```

O servidor sobe em `http://localhost:8080`.

### Testar manualmente

```bash
# Health check
curl http://localhost:8080/health

# Transcrever um arquivo de áudio
curl -X POST http://localhost:8080/transcribe \
  -F "audio=@caminho/para/audio.ogg"
```

---

## Docker

```bash
# Build
docker build -t gadogestor-audio-transcription .

# Run
docker run -p 8082:8080 \
  -e GEMINI_API_KEY=sua-chave-gemini \
  gadogestor-audio-transcription
```

> O `gadogestor-whatsapp` aponta por padrão para `http://localhost:8082` (`Transcription:BaseUrl`). Ajuste conforme o ambiente de deployment.

---

## Testes

```bash
go test ./...
go vet ./...
```

---

## Formatos suportados

`mp3`, `mp4`, `mpeg`, `mpga`, `m4a`, `wav`, `webm`, `ogg`, `flac` — até **25 MB**.
