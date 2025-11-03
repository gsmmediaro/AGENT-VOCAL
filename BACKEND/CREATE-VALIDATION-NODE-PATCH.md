# CREATE.json - Defensive Validation Node Patch

## Overview
This document describes how to add a defensive validation node to the CREATE.json workflow to catch AI date calculation mistakes before they reach Supabase.

## Node to Add

**Position:** After "Validate Required Fields" (true branch), before "Validate Business Rules"

**Node Configuration:**

```json
{
  "parameters": {
    "jsCode": "// DEFENSIVE VALIDATION: Catch AI date calculation mistakes before they reach Supabase\n// This node validates dates/times and rejects invalid requests with Romanian error messages\n\nconst data = $input.item.json;\nconst toolCallId = data.tool_call_id;\n\n// Get current date/time in Europe/Bucharest timezone\nconst now = new Date();\nconst bucharestTime = new Date(now.toLocaleString('en-US', { timeZone: 'Europe/Bucharest' }));\nconst today = new Date(bucharestTime.getFullYear(), bucharestTime.getMonth(), bucharestTime.getDate());\n\n// Parse requested date\nconst [year, month, day] = data.preferred_date.split('-').map(Number);\nconst requestedDate = new Date(year, month - 1, day); // month is 0-indexed\nconst requestedDateOnly = new Date(year, month - 1, day);\n\n// Validation errors array\nconst errors = [];\n\n// 1. Check if date is in the past\nif (requestedDateOnly < today) {\n  errors.push('Data solicitată este în trecut');\n}\n\n// 2. Check if date is weekend (Saturday = 6, Sunday = 0)\nconst dayOfWeek = requestedDate.getDay();\nif (dayOfWeek === 0 || dayOfWeek === 6) {\n  const dayName = dayOfWeek === 0 ? 'duminică' : 'sâmbătă';\n  errors.push(`Data solicitată este ${dayName} - clinica este închisă în weekend`);\n}\n\n// 3. Check if date is today but time has passed\nif (requestedDateOnly.getTime() === today.getTime()) {\n  const [hour, minute] = data.preferred_time.split(':').map(Number);\n  const requestedDateTime = new Date(bucharestTime);\n  requestedDateTime.setHours(hour, minute, 0, 0);\n  \n  if (requestedDateTime <= bucharestTime) {\n    const nextHour = bucharestTime.getHours() + 1;\n    errors.push(`Ora ${data.preferred_time} a trecut. Alegeți după ora ${nextHour.toString().padStart(2, '0')}:00`);\n  }\n}\n\n// 4. Check working hours (10:00-20:00 per knowledge base)\nconst [hour, minute] = data.preferred_time.split(':').map(Number);\nif (hour < 10 || hour >= 20) {\n  errors.push('Programul clinicii: luni-vineri între ora zece și douăzeci');\n}\n\n// 5. Check 30-minute slots\nif (minute !== 0 && minute !== 30) {\n  errors.push('Programări doar la ore întregi sau la jumătate (ex: 10:00, 10:30, 11:00)');\n}\n\n// 6. Known holidays (Romanian public holidays 2025)\nconst holidays = [\n  '2025-01-01', // Anul Nou\n  '2025-01-02', // Anul Nou\n  '2025-01-24', // Ziua Unirii\n  '2025-05-01', // Ziua Muncii\n  '2025-05-02', // Ziua Muncii\n  '2025-06-01', // Ziua Copilului\n  '2025-06-02', // Ziua Copilului\n  '2025-08-15', // Adormirea Maicii Domnului\n  '2025-11-30', // Sfântul Andrei\n  '2025-12-01', // Ziua Națională\n  '2025-12-25', // Crăciunul\n  '2025-12-26', // Crăciunul\n];\n\nif (holidays.includes(data.preferred_date)) {\n  errors.push('Data solicitată este sărbătoare legală - clinica este închisă');\n}\n\n// Return error if validation failed\nif (errors.length > 0) {\n  return {\n    json: {\n      valid: false,\n      error_message: 'Eroare: ' + errors.join('. ') + '. Vă rog să alegeți o altă dată între luni și vineri.',\n      tool_call_id: toolCallId\n    }\n  };\n}\n\n// Validation passed - pass through data\nreturn {\n  json: {\n    valid: true,\n    ...data\n  }\n};"
  },
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [-1200, 112],
  "id": "defensive-date-validation-node",
  "name": "Validate Date & Time (Defensive)",
  "notes": "Catches AI date calculation mistakes - validates past dates, weekends, holidays, working hours"
}
```

## Error Response Node to Add

**Position:** After validation node (false branch)

```json
{
  "parameters": {
    "assignments": {
      "assignments": [
        {
          "id": "date_validation_error",
          "name": "results",
          "value": "={{ [{\"toolCallId\": $json.tool_call_id, \"result\": $json.error_message}] }}",
          "type": "array"
        }
      ]
    },
    "options": {}
  },
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "position": [704, 304],
  "id": "date-validation-error-node",
  "name": "Error: Date Validation Failed"
}
```

## Updated Connections

**In CREATE.json connections section, update:**

```json
"Validate Required Fields": {
  "main": [
    [
      {
        "node": "Validate Date & Time (Defensive)",
        "type": "main",
        "index": 0
      }
    ],
    [
      {
        "node": "Error: Missing Fields",
        "type": "main",
        "index": 0
      }
    ]
  ]
},
"Validate Date & Time (Defensive)": {
  "main": [
    [
      {
        "node": "Validate Business Rules",
        "type": "main",
        "index": 0
      }
    ],
    [
      {
        "node": "Error: Date Validation Failed",
        "type": "main",
        "index": 0
      }
    ]
  ]
},
"Error: Date Validation Failed": {
  "main": [
    [
      {
        "node": "Merge All Outputs",
        "type": "main",
        "index": 0
      }
    ]
  ]
}
```

## Important Notes

1. **Working Hours Fix:** The existing "Validate Business Rules" node checks 09:00-18:00, but the knowledge base says 10:00-20:00. This defensive node uses the correct hours (10:00-20:00).

2. **Holiday Support:** This node includes Romanian public holidays for 2025. Update the holidays array annually.

3. **Error Messages:** All error messages are in Romanian to match user-facing language.

4. **Position Adjustments:** You may need to adjust node positions in the n8n UI to avoid overlaps.

## Testing

After adding this node, test with:
- Past date: `preferred_date: "2025-11-02"` → Should reject
- Weekend: `preferred_date: "2025-11-08"` (Saturday) → Should reject  
- Holiday: `preferred_date: "2025-12-25"` → Should reject
- Outside hours: `preferred_time: "09:00"` → Should reject (clinic opens at 10:00)
- Valid: `preferred_date: "2025-11-04"`, `preferred_time: "14:00"` → Should pass
