# 🚀 Lead Processing Automation with n8n + Groq AI

## Project Overview

I built an automated lead processing system that receives leads through webhooks, classifies them using AI (Groq LLaMA 3.3 70B), and stores everything in Google Sheets automatically.

![n8n Workflow View](Captura%20de%20pantalla%202026-02-23%20174818.png)

---

## Tech Stack

- **n8n**: Workflow automation platform
- **Groq API**: AI classification using LLaMA 3.3 70B (70B params model)
- **Google Sheets**: Data storage and management
- **Gmail**: Automated email responses

---

## How It Works

### Architecture

```
Webhook Endpoint
       ↓
  Email Validation
       ↓
  Duplicate Check (Google Sheets)
       ↓
  AI Classification (Groq LLaMA 3.3 70B)
       ↓
  Priority Assignment
       ↓
  Save to Google Sheets
       ↓
  Send Categorized Email Response
       ↓
  Return Success/Error JSON
```

### Features

✅ **Email validation** with regex pattern matching  
✅ **Duplicate detection** to prevent repeat entries  
✅ **AI-powered classification** into categories: Sales, Support, Information, Spam  
✅ **Automatic priority assignment** (High/Medium/Low) based on category  
✅ **Error logging** in separate sheet for failed validations  
✅ **Automated email responses** customized by category  
✅ **RESTful API** with proper status codes (200, 400, 409)

---

## Setup Instructions

### 1. n8n Installation

I used **n8n Cloud** for this project, but it can also be self-hosted with Docker:

```bash
# Docker setup (optional)
mkdir n8n-leads && cd n8n-leads

cat > docker-compose.yml << 'EOF'
version: '3.8'
services:
  n8n:
    image: n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=YourSecurePassword123!
      - GROQ_API_KEY=${GROQ_API_KEY}
      - GOOGLE_SHEET_ID=${GOOGLE_SHEET_ID}
      - NOTIFY_EMAIL=${NOTIFY_EMAIL}
    volumes:
      - n8n_data:/home/node/.n8n
volumes:
  n8n_data:
EOF

docker compose up -d
```

### 2. Groq API Configuration

I chose Groq for AI classification because of its generous free tier and fast inference:

1. Created account at **https://console.groq.com**
2. Generated API key (starts with `gsk_...`)
3. Configured as environment variable in n8n: `GROQ_API_KEY`

**Cost**: Free tier allows ~14,400 requests/day with **llama-3.3-70b-versatile** model

### 3. Google Sheets Structure

Created a spreadsheet with two tabs:

**Leads_DB** (valid leads):
| Fecha | Nombre | Email | Telefono | Categoria | Prioridad | Resumen | Confianza IA | Fuente | Mensaje Original |

**Errores** (validation failures):
| Fecha | Nombre | Email Recibido | Error | Payload Completo |

### 4. OAuth2 Credentials

Connected the following services in n8n Settings → Credentials:
- Google Sheets OAuth2 API
- Gmail OAuth2

---

## API Usage

### Endpoint

```
POST https://your-n8n-instance.com/webhook/leads-entrantes
Content-Type: application/json
```

### Request Example

```json
{
  "nombre": "Ana García",
  "email": "ana.garcia@empresa.com",
  "telefono": "+52 55 1234 5678",
  "mensaje": "Hello, I'm interested in a demo for my team of 50 people. What are the enterprise plans?",
  "fuente": "landing-page"
}
```

### Response Examples

**Success (200)**
```json
{
  "success": true,
  "message": "Lead procesado correctamente",
  "data": {
    "nombre": "Ana García",
    "categoria": "Ventas",
    "prioridad": "Media",
    "resumen": "Solicita demo enterprise para equipo de 50 personas"
  }
}
```

**Validation Error (400)**
```json
{
  "success": false,
  "error": "El campo email es obligatorio y debe tener formato valido"
}
```

**Duplicate (409)**
```json
{
  "success": false,
  "error": "Este email ya fue registrado anteriormente"
}
```

---

## AI Classification Logic

I configured the Groq API with a custom system prompt:

```
You are a B2B lead classifier. Analyze the message and respond ONLY with valid JSON without markdown.
Exact structure: {"categoria": "Ventas | Soporte | Informacion | Spam", "resumen": "max 15 words", "confianza": 0.95}

- Sales: buy/contract/pay/invoice
- Support: error/not working/technical problem
- Information: general info/how it works
- Spam: bot/promotional/nonsense
```

Priority mapping:
- **Sales** → Medium
- **Support** → High
- **Information** → Medium
- **Spam** → Low

---

## Testing

I tested the workflow with Postman:

```bash
# Valid lead
curl -X POST https://your-instance.com/webhook/leads-entrantes \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Test","email":"test@example.com","mensaje":"Need a quote"}'

# Invalid email
curl -X POST https://your-instance.com/webhook/leads-entrantes \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Test","mensaje":"No email field"}'

# Duplicate
# Send same request twice
```

---

## Workflow Nodes

The n8n workflow consists of:

1. **Webhook Entrada** - Receives POST requests
2. **Validar Email** - Regex validation for email format
3. **Registrar Error en Sheets** - Logs validation failures
4. **Buscar Email Duplicado** - Checks existing emails in sheet
5. **Clasificar con Groq** - HTTP request to Groq API
6. **Procesar y Asignar Prioridad** - Parses AI response and assigns priority
7. **Guardar en Sheets** - Appends to Leads_DB
8. **Router por Categoría** - Routes to appropriate email template
9. **Enviar Email [Ventas/Soporte/etc]** - Sends categorized responses
10. **Responder Success** - Returns 200 with JSON

---

## Challenges & Solutions

**Challenge**: GitHub blocked my push due to exposed API key in workflow  
**Solution**: Replaced hardcoded key with environment variable `{{ $env.GROQ_API_KEY }}`, cleaned Git history with orphan branch

**Challenge**: Groq sometimes returns markdown-wrapped JSON  
**Solution**: Added `.replace(/```json|```/g, '').trim()` before parsing

**Challenge**: Email duplicate detection was case-sensitive  
**Solution**: Normalized emails with `.toLowerCase().trim()` in comparison

---

## Environment Variables

```bash
GROQ_API_KEY=gsk_...           # Groq API key
GOOGLE_SHEET_ID=1BxiMVs...     # Google Sheets document ID
NOTIFY_EMAIL=your@email.com    # Notification email address
```

---

## Future Improvements

- [ ] Add webhook signature validation for security
- [ ] Implement rate limiting
- [ ] Add Slack/Discord notifications for high-priority leads
- [ ] Create dashboard for lead analytics
- [ ] Add multi-language support for AI classification
- [ ] Implement lead scoring system

---

## License

MIT

---

**Built with n8n v1.x+ and Groq llama-3.3-70b-versatile**
