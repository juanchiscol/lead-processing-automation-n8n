# 🚀 Guía de Setup — Automatización de Leads con n8n + Groq

## Lo que vas a tener al final

Un pipeline que recibe leads por webhook, los clasifica con IA (Groq LLaMA 3.3 70B) y guarda todo en Google Sheets automáticamente.

![Vista del Workflow en n8n](Captura%20de%20pantalla%202026-02-23%20174818.png)

---

## PASO 1 — Instalar n8n (elige una opción)

### Opción A: n8n Cloud (más fácil, sin servidor)
1. Ve a **https://n8n.io** → "Get started for free"
2. Crea tu cuenta
3. Listo, ya tienes n8n corriendo ✅

### Opción B: Docker en tu servidor (gratis, self-hosted)
```bash
# Crea el archivo docker-compose.yml
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
      - N8N_BASIC_AUTH_PASSWORD=TuPasswordSeguro123!
      - GROQ_API_KEY=${GROQ_API_KEY}
      - GOOGLE_SHEET_ID=${GOOGLE_SHEET_ID}
      - NOTIFY_EMAIL=${NOTIFY_EMAIL}
    volumes:
      - n8n_data:/home/node/.n8n
volumes:
  n8n_data:
EOF

# Copia tu .env.example como .env y llena los valores
cp .env.example .env
nano .env   # Edita con tus valores reales

# Levanta n8n
docker compose up -d

# Abre en tu navegador:
# http://localhost:5678
```

---

## PASO 2 — Obtener tu API Key de Groq

1. Ve a **https://console.groq.com**
2. Inicia sesión o crea cuenta gratuita (con GitHub o Gmail)
3. Menú izquierdo → **"API Keys"** o **"Developers"**
4. Clic en **"Create API Key"** → dale un nombre: "n8n-leads"
5. **Copia la key** (solo se muestra una vez) — empieza con `gsk_...`
6. Pégala en tu `.env` como `GROQ_API_KEY=gsk_...`

> ⚠️ Groq ofrece un tier gratuito muy generoso. Con el modelo **llama-3.3-70b-versatile**, puedes procesar miles de leads sin costo. Límite: ~14,400 requests/día en tier gratuito.

---

## PASO 3 — Configurar Google Sheets

### 3a. Crear la hoja de cálculo
1. Ve a **https://sheets.google.com**
2. Crea una hoja nueva → nómbrala **"Leads Sistema"**
3. Crea **dos pestañas** (tabs):
   - **`Leads_DB`** — aquí van los leads válidos
   - **`Errores`** — aquí van los que fallan validación

### 3b. Encabezados de Leads_DB (fila 1)
| A | B | C | D | E | F | G | H | I | J |
|---|---|---|---|---|---|---|---|---|---|
| Fecha | Nombre | Email | Telefono | Categoria | Prioridad | Resumen | Confianza IA | Fuente | Mensaje Original |

### 3c. Encabezados de Errores (fila 1)
| A | B | C | D | E |
|---|---|---|---|---|
| Fecha | Nombre | Email Recibido | Error | Payload Completo |

### 3d. Obtener el ID de la hoja
La URL de tu hoja se ve así:
```
https://docs.google.com/spreadsheets/d/1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms/edit
                                       ↑ ESTE ES TU SHEET ID
```
Cópialo y ponlo en `.env` como `GOOGLE_SHEET_ID=...`

---

## PASO 4 — Conectar Google Sheets en n8n

1. En n8n, ve a **Settings → Credentials**
2. Clic en **"Add Credential"**
3. Busca **"Google Sheets OAuth2 API"**
4. Sigue el flujo de autorización con tu cuenta de Google
5. Dale permisos de lectura y escritura a Sheets
6. Guarda con el nombre **"Google Sheets Account"**

---

## PASO 5 — Conectar Gmail en n8n

1. En n8n → **Settings → Credentials**
2. Clic en **"Add Credential"**
3. Busca **"Gmail OAuth2"**
4. Autoriza con tu cuenta de Gmail
5. Guarda con el nombre **"Gmail Account"**

> 💡 Alternativa: Si prefieres usar SMTP (Outlook, otro proveedor), cambia los nodos de Gmail por nodos "Send Email" con SMTP.

---

## PASO 6 — Importar el Workflow

1. Abre n8n en tu navegador
2. Menú izquierdo → **"Workflows"**
3. Clic en **"Import from file"**
4. Sube el archivo **`leads_workflow.json`**
5. El workflow se abre con todos los nodos ya conectados

---

## PASO 7 — Configurar Variables de Entorno en n8n

1. En n8n → **Settings → n8n Settings**
2. Busca **"Variables"**
3. Agrega estas variables:
   - `GROQ_API_KEY` = tu key de Groq (empieza con gsk_...)
   - `GOOGLE_SHEET_ID` = ID de tu hoja
   - `NOTIFY_EMAIL` = tu email de notificaciones

---

## PASO 8 — Activar el Workflow y Obtener URL del Webhook

1. Abre el workflow importado
2. Clic en el nodo **"📥 Webhook Entrada"**
3. Copia la **"Production URL"** — se ve así:
   ```
   https://tudominio.com/webhook/leads-entrantes
   ```
4. Clic en el toggle de arriba a la derecha para **activar** el workflow

---

## PASO 9 — Probar con Postman

### Instalar Postman
Descarga en: https://www.postman.com/downloads/

### Request de prueba
```
Método:  POST
URL:     https://tudominio.com/webhook/leads-entrantes
Headers: Content-Type: application/json
```

**Body (raw → JSON):**
```json
{
  "nombre":   "Ana García",
  "email":    "ana.garcia@empresa.com",
  "telefono": "+52 55 1234 5678",
  "mensaje":  "Hola, me interesa una demostración de su producto para mi equipo de 50 personas. ¿Cuáles son los planes enterprise?",
  "fuente":   "landing-page"
}
```

**Respuesta esperada (200):**
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

### Prueba de error (email vacío)
```json
{
  "nombre":  "Sin Email",
  "mensaje": "Prueba sin email"
}
```
**Respuesta esperada (400):**
```json
{
  "success": false,
  "error": "El campo email es obligatorio y debe tener formato valido"
}
```

### Prueba de duplicado (mismo email dos veces)
Envía el mismo request dos veces → segunda vez responde 409.

---

## PASO 10 — Conectar tu Formulario Web (opcional)

Si tienes un formulario HTML, solo agrega esto al `submit`:

```javascript
// En tu formulario HTML
async function enviarLead(event) {
  event.preventDefault();
  
  const data = {
    nombre:   document.getElementById('nombre').value,
    email:    document.getElementById('email').value,
    telefono: document.getElementById('telefono').value,
    mensaje:  document.getElementById('mensaje').value,
    fuente:   'formulario-web'
  };

  const response = await fetch('https://tudominio.com/webhook/leads-entrantes', {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify(data)
  });

  const result = await response.json();
  
  if (result.success) {
    alert('¡Gracias! Te contactaremos pronto.');
  } else {
    alert('Error: ' + result.error);
  }
}
```

---

## Flujo completo resumido

```
Formulario/Postman
       ↓
  [Webhook n8n]
       ↓
  ¿Tiene email válido? ──NO──→ Registrar error en Sheets → Responder 400
       ↓ SÍ
  ¿Email ya existe en Sheets? ──SÍ──→ Responder 409 (duplicado)
       ↓ NO
  [Groq AI clasifica el mensaje con LLaMA 3.3 70B]
       ↓
  Asignar prioridad (Alta/Media/Baja)
       ↓
  Guardar en Google Sheets
       ↓
  Router por categoría
  ├── Ventas      → Email personalizado de ventas
  ├── Soporte     → Email con ticket de soporte
  ├── Información → Email con recursos y docs
  └── Spam        → Solo registrar, no enviar email
       ↓
  Responder 200 OK
```

---

## Troubleshooting común

| Problema | Solución |
|----------|----------|
| Webhook no responde | Verifica que el workflow esté **activado** (toggle ON) |
| Error de Google Sheets | Reconecta las credenciales OAuth2 en Settings |
| Groq no clasifica bien | Ajusta el system prompt en el nodo "Clasificar con Groq" |
| Email no llega | Revisa carpeta de spam / reconecta Gmail OAuth2 |
| Duplicados no se detectan | Verifica que la columna se llame exactamente **"Email"** |

---
