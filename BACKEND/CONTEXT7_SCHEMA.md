# Context7 Client Configuration Schema

## Overview
Acest document definește schema de configurare pentru fiecare client în platforma multi-tenant. Fiecare client are un document JSON stocat în Context7 care conține toate setările și credențialele necesare.

## Schema Structură

### Root Level Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `clientId` | String | Yes | Identificator unic al clientului (ex: "dr_serban") |
| `clientName` | String | Yes | Numele complet al clientului (ex: "Clinica Dr. Șerban") |
| `vapiAssistantId` | String | Yes | ID-ul asistentului Vapi.ai asociat acestui client |
| `googleCalendarId` | String | Yes | ID-ul Google Calendar pentru programări |
| `n8nCredentialId_Google` | String | Yes | ID-ul credențialelor Google din n8n |
| `enabledTools` | Array<String> | Yes | Lista uneltelor activate pentru acest client |
| `billingPlan` | Object | Yes | Detalii plan de facturare |
| `workingHours` | Object | Yes | Configurare program de lucru |
| `appointmentSettings` | Object | Yes | Setări pentru programări |
| `notifications` | Object | Yes | Configurare notificări |
| `knowledgeBase` | String | Yes | Baza de cunoștințe în format Markdown |
| `metadata` | Object | Yes | Metadate despre configurație |

### billingPlan Object

```json
{
  "taxaLunara": 800,
  "costPerMinut": 0.56
}
```

| Field | Type | Description |
|-------|------|-------------|
| `taxaLunara` | Number | Taxa fixă lunară (RON) |
| `costPerMinut` | Number | Cost per minut de convorbire (RON) |

### workingHours Object

```json
{
  "start": 10,
  "end": 20,
  "timezone": "Europe/Bucharest",
  "workingDays": [1, 2, 3, 4, 5]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `start` | Number | Ora de început (format 24h) |
| `end` | Number | Ora de sfârșit (format 24h) |
| `timezone` | String | Timezone IANA |
| `workingDays` | Array<Number> | Zilele lucrătoare (0=Duminică, 6=Sâmbătă) |

### appointmentSettings Object

```json
{
  "defaultDuration": 60,
  "slotInterval": 30,
  "advanceBookingDays": 90
}
```

| Field | Type | Description |
|-------|------|-------------|
| `defaultDuration` | Number | Durata implicită programare (minute) |
| `slotInterval` | Number | Interval între slot-uri (minute) |
| `advanceBookingDays` | Number | Câte zile în avans se pot face programări |

### notifications Object

```json
{
  "sms": {
    "enabled": true,
    "twilioFrom": "+40373811265"
  },
  "email": {
    "enabled": false
  }
}
```

### enabledTools Array

Valori posibile:
- `"googleCalendar"` - Google Calendar API pentru programări
- `"braveSearch"` - Brave Search API pentru căutări web
- `"mcpPlaywright"` - MCP Playwright pentru web scraping

## Context7 API Integration

### Endpoint pentru preluare configurație

```http
GET https://api.context7.com/v1/search
Headers:
  Authorization: Bearer YOUR_CONTEXT7_API_KEY
  Content-Type: application/json

Body:
{
  "query": "clientId:dr_serban",
  "limit": 1
}
```

### Response Expected

```json
{
  "results": [
    {
      "document": {
        // ... full client config JSON
      }
    }
  ]
}
```

## Usage în n8n

În workflow-ul MAIN.json, după extragerea `clientId` din query parameter:

1. **Extract ClientID** - extrage `clientId` din `$request.query.clientId`
2. **Get Client Config** - face HTTP Request la Context7
3. **Parse Config** - extrage documentul din răspuns și îl trimite către sub-workflows

Toate sub-workflow-urile (CREATE, CHECK, CANCEL, TOOL_*) primesc `clientConfig` ca input și îl folosesc pentru:
- Credential IDs dinamice
- Calendar IDs dinamice
- Working hours validation
- Enabled tools verification

## Security Notes

- **Nu stoca niciodată API keys în Context7** - folosește doar IDs de credențiale din n8n
- **Validează întotdeauna clientId** - asigură-te că provine dintr-o sursă sigură (Vapi webhook signature)
- **Rate limiting** - implementează rate limiting per clientId
- **Audit logs** - logăm toate accesările de configurații

## Example Query în n8n

```javascript
// În nodul "Get Client Config (Context7)"
{
  "method": "POST",
  "url": "https://api.context7.com/v1/search",
  "headers": {
    "Authorization": "Bearer {{ $credentials.context7Api.apiKey }}",
    "Content-Type": "application/json"
  },
  "body": {
    "query": "clientId:{{ $json.clientId }}",
    "limit": 1
  }
}
```

## Versioning

Schema version: **1.0**

Pentru modificări majore, incrementează `metadata.version` și documentează migration steps.
