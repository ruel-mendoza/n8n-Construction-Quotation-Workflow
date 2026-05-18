# Production-Ready n8n Construction Quotation Workflow

## Part 1: Architectural Logic Breakdown

### Flow Overview

```
Webhook/Form Trigger
       ↓
Google Sheets Fetch (full price list)
       ↓
Code Node: Cross-Reference Materials
       ↓
    [Branch]
    /      \
Found    Missing/Unavailable
    \      /
     ↓    ↓
AI Fallback Node (missing items only)
       ↓
Code Node: Merge & Calculate Total
       ↓
    [Split]
    /      \
Customer   Internal
 Email     Notification
(Gmail)  (Slack/Teams)
```

---

### Stage-by-Stage Logic

**Stage 1 — Trigger**
A `Webhook` node receives a POST payload with `customer_email` and `requested_materials` (array of `{ name, quantity }`). A Form Trigger variant is also supported.

**Stage 2 — Google Sheets Fetch**
A single `Google Sheets → Get Many Rows` call pulls the entire price list sheet into memory. This avoids per-item API calls and is far more efficient.

**Stage 3 — Code Node: Material Cross-Reference**
A JavaScript `Code` node iterates over every requested material and checks against the fetched sheet rows. It produces two arrays:
- `matched` — items found in the sheet AND marked `Availability: Yes`, with unit price × quantity subtotal.
- `missing` — items not found OR where `Availability: No`.

**Stage 4 — IF Branch**
An `IF` node checks `{{ $json.missing.length > 0 }}`. If true, the flow routes to the AI Agent; if false (all items available), it skips directly to the merge/calculation step.

**Stage 5 — AI Agent (Fallback)**
An `OpenAI` or `Anthropic` node receives a structured prompt containing the missing items list. It is instructed to return a JSON array of `{ name, estimated_unit_price, source_note, alternative_suggestion }`. An optional HTTP Request node can be chained to perform a live web search (SerpAPI or similar) to ground the estimates.

**Stage 6 — Merge & Calculate**
A second `Code` node combines matched items + AI-estimated items, calculates line totals, grand total, and flags which items are estimated vs. confirmed.

**Stage 7 — Customer Email (Gmail)**
An HTML email is built inline with a professional quotation table. Estimated items are highlighted in amber with an "⚠ Estimated Market Price" badge. A disclaimer footer is appended if any fallback pricing was used.

**Stage 8 — Internal Notification**
A parallel branch sends the full payload (including missing materials and AI suggestions) to the internal marketing/ops team via Slack, Teams, or a second Gmail node.

---

### AI Fallback Prompt Design

The prompt is dynamically constructed:

```
You are a construction materials pricing expert in the Philippines.
The following materials were NOT found in our internal price list or are currently unavailable:

{{missingItemsJSON}}

For each item:
1. Estimate a reasonable current market unit price (in PHP).
2. Suggest one widely available alternative if the exact item is hard to source.
3. Briefly note your pricing source assumption.

Return ONLY a valid JSON array with this exact shape:
[{ "name": "", "quantity": 0, "estimated_unit_price": 0, "subtotal": 0, "source_note": "", "alternative": "" }]
```

---

## Part 2: Complete n8n JSON Workflow

```json
{
  "name": "Construction Quotation System",
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "construction-quote",
        "responseMode": "responseNode",
        "options": {}
      },
      "id": "node-webhook-trigger",
      "name": "Webhook Trigger",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [240, 300],
      "webhookId": "REPLACE_WITH_YOUR_WEBHOOK_ID"
    },
    {
      "parameters": {
        "documentId": {
          "__rl": true,
          "value": "REPLACE_WITH_YOUR_GOOGLE_SHEET_ID",
          "mode": "id"
        },
        "sheetName": {
          "__rl": true,
          "value": "Sheet1",
          "mode": "name"
        },
        "filtersUI": {},
        "options": {
          "firstRowAsHeaders": true
        }
      },
      "id": "node-gsheets-fetch",
      "name": "Fetch Price List (Google Sheets)",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 4,
      "position": [460, 300],
      "credentials": {
        "googleSheetsOAuth2Api": {
          "id": "REPLACE_WITH_GSHEETS_CREDENTIAL_ID",
          "name": "Google Sheets OAuth2"
        }
      }
    },
    {
      "parameters": {
        "mode": "combine",
        "combinationMode": "multiplex",
        "options": {}
      },
      "id": "node-merge-inputs",
      "name": "Merge Webhook + Sheet Data",
      "type": "n8n-nodes-base.merge",
      "typeVersion": 3,
      "position": [680, 300]
    },
    {
      "parameters": {
        "jsCode": "// ============================================================\n// STAGE 3: Cross-Reference requested materials vs. price list\n// ============================================================\n\n// Input 0 = Webhook payload, Input 1 = Google Sheets rows\nconst webhookData = $input.all()[0].json;\nconst sheetRows   = $input.all().slice(1).map(i => i.json);\n\nconst requestedMaterials = webhookData.requested_materials || [];\nconst customerEmail      = webhookData.customer_email || '';\n\nconst matched = [];\nconst missing = [];\n\nfor (const item of requestedMaterials) {\n  const itemNameLower = (item.name || '').toLowerCase().trim();\n  const qty           = parseFloat(item.quantity) || 1;\n\n  // Find row in Google Sheet (case-insensitive match on 'Material Name')\n  const sheetRow = sheetRows.find(\n    r => (r['Material Name'] || '').toLowerCase().trim() === itemNameLower\n  );\n\n  if (sheetRow && (sheetRow['Availability'] || '').toLowerCase() === 'yes') {\n    const unitPrice = parseFloat(sheetRow['Unit Price']) || 0;\n    matched.push({\n      name:        sheetRow['Material Name'],\n      quantity:    qty,\n      unit_price:  unitPrice,\n      subtotal:    unitPrice * qty,\n      source:      'price_list',\n      availability:'confirmed'\n    });\n  } else {\n    missing.push({\n      name:     item.name,\n      quantity: qty,\n      reason:   sheetRow ? 'unavailable' : 'not_in_catalog'\n    });\n  }\n}\n\nreturn [{\n  json: {\n    customer_email:  customerEmail,\n    matched_items:   matched,\n    missing_items:   missing,\n    has_missing:     missing.length > 0,\n    matched_subtotal: matched.reduce((s, i) => s + i.subtotal, 0)\n  }\n}];"
      },
      "id": "node-crossref-code",
      "name": "Cross-Reference Materials",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [900, 300]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": false,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "cond-has-missing",
              "leftValue": "={{ $json.has_missing }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "true"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "node-if-missing",
      "name": "Has Missing Items?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [1120, 300]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "={{ \n  'You are a construction materials pricing expert in the Philippines.\\n' +\n  'The following materials were NOT found in our internal price list or are currently unavailable:\\n\\n' +\n  JSON.stringify($json.missing_items, null, 2) +\n  '\\n\\nFor each item:\\n' +\n  '1. Estimate a reasonable current market unit price in PHP.\\n' +\n  '2. Suggest one widely available alternative if the item is hard to source.\\n' +\n  '3. Note your pricing source assumption briefly.\\n\\n' +\n  'IMPORTANT: Return ONLY a valid raw JSON array. No markdown, no explanation.\\n' +\n  'Shape: [{\"name\":\"\",\"quantity\":0,\"estimated_unit_price\":0,\"subtotal\":0,\"source_note\":\"\",\"alternative\":\"\"}]'\n}}",
        "options": {
          "systemMessage": "You are a helpful assistant that ONLY returns valid JSON arrays. Never include markdown code blocks or any text outside the JSON array.",
          "temperature": 0.2,
          "maxTokens": 1500
        }
      },
      "id": "node-ai-fallback",
      "name": "AI Fallback Pricing Agent",
      "type": "@n8n/n8n-nodes-langchain.openAi",
      "typeVersion": 1,
      "position": [1340, 200],
      "credentials": {
        "openAiApi": {
          "id": "REPLACE_WITH_OPENAI_CREDENTIAL_ID",
          "name": "OpenAI API"
        }
      }
    },
    {
      "parameters": {
        "jsCode": "// ============================================================\n// STAGE 5b: Parse AI response and attach original context\n// ============================================================\n\nconst aiText = $input.first().json?.text ||\n               $input.first().json?.message?.content ||\n               $input.first().json?.choices?.[0]?.message?.content || '[]';\n\nlet aiItems = [];\ntry {\n  // Strip any accidental markdown fences\n  const cleaned = aiText.replace(/```json|```/g, '').trim();\n  aiItems = JSON.parse(cleaned);\n} catch(e) {\n  aiItems = [{ name: 'Parse Error', quantity: 0, estimated_unit_price: 0, subtotal: 0, source_note: 'AI response could not be parsed', alternative: 'N/A' }];\n}\n\n// Recalculate subtotals defensively\naiItems = aiItems.map(item => ({\n  ...item,\n  subtotal: (parseFloat(item.estimated_unit_price) || 0) * (parseFloat(item.quantity) || 1),\n  source: 'ai_estimate'\n}));\n\nreturn [{ json: { ai_items: aiItems } }];"
      },
      "id": "node-parse-ai",
      "name": "Parse AI Response",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1560, 200]
    },
    {
      "parameters": {
        "mode": "combine",
        "combinationMode": "mergeByPosition",
        "options": {}
      },
      "id": "node-merge-ai-with-context",
      "name": "Merge AI Items with Context",
      "type": "n8n-nodes-base.merge",
      "typeVersion": 3,
      "position": [1560, 380]
    },
    {
      "parameters": {
        "jsCode": "// ============================================================\n// STAGE 6: Final merge, total calculation, build quote object\n// ============================================================\n\n// This node receives either:\n//   A) merged AI path (has both matched_items + ai_items)\n//   B) direct path (only matched_items, no missing)\n\nconst data    = $input.first().json;\nconst matched = data.matched_items  || [];\nconst aiItems = data.ai_items       || [];\n\nconst matchedTotal  = matched.reduce((s, i) => s + (i.subtotal || 0), 0);\nconst aiTotal       = aiItems.reduce((s, i) => s + (i.subtotal || 0), 0);\nconst grandTotal    = matchedTotal + aiTotal;\nconst hasEstimates  = aiItems.length > 0;\n\n// Build a reference number\nconst refNum = 'QTE-' + Date.now().toString(36).toUpperCase();\nconst today  = new Date().toLocaleDateString('en-PH', { year:'numeric', month:'long', day:'numeric' });\n\n// ---- Build HTML email body ----------------------------------------\nconst confirmedRows = matched.map(i =>\n  `<tr style=\"background:#f9f9f9\">\n     <td style=\"padding:8px 12px;border-bottom:1px solid #e2e8f0\">${i.name}</td>\n     <td style=\"padding:8px 12px;border-bottom:1px solid #e2e8f0;text-align:center\">${i.quantity}</td>\n     <td style=\"padding:8px 12px;border-bottom:1px solid #e2e8f0;text-align:right\">₱${i.unit_price.toLocaleString('en-PH',{minimumFractionDigits:2})}</td>\n     <td style=\"padding:8px 12px;border-bottom:1px solid #e2e8f0;text-align:right\">₱${i.subtotal.toLocaleString('en-PH',{minimumFractionDigits:2})}</td>\n     <td style=\"padding:8px 12px;border-bottom:1px solid #e2e8f0;text-align:center\"><span style=\"color:#16a34a;font-weight:600\">✓ Confirmed</span></td>\n   </tr>`\n).join('');\n\nconst estimatedRows = aiItems.map(i =>\n  `<tr style=\"background:#fffbeb\">\n     <td style=\"padding:8px 12px;border-bottom:1px solid #fde68a\">${i.name}${ i.alternative && i.alternative !== 'N/A' ? `<br><small style=\"color:#92400e\">Alt: ${i.alternative}</small>` : '' }</td>\n     <td style=\"padding:8px 12px;border-bottom:1px solid #fde68a;text-align:center\">${i.quantity}</td>\n     <td style=\"padding:8px 12px;border-bottom:1px solid #fde68a;text-align:right\">₱${(parseFloat(i.estimated_unit_price)||0).toLocaleString('en-PH',{minimumFractionDigits:2})}</td>\n     <td style=\"padding:8px 12px;border-bottom:1px solid #fde68a;text-align:right\">₱${(parseFloat(i.subtotal)||0).toLocaleString('en-PH',{minimumFractionDigits:2})}</td>\n     <td style=\"padding:8px 12px;border-bottom:1px solid #fde68a;text-align:center\"><span style=\"color:#d97706;font-weight:600\">⚠ Estimated</span></td>\n   </tr>`\n).join('');\n\nconst estimateDisclaimer = hasEstimates\n  ? `<div style=\"margin:20px 0;padding:14px 18px;background:#fffbeb;border-left:4px solid #f59e0b;border-radius:4px\">\n       <strong style=\"color:#92400e\">⚠ Notice on Estimated Prices</strong><br>\n       <span style=\"color:#78350f;font-size:13px\">Items marked \"Estimated\" are based on current market research and may vary. \n       Our team will confirm exact pricing within 24 hours.</span>\n     </div>` : '';\n\nconst htmlEmail = `\n<!DOCTYPE html><html><head><meta charset=\"UTF-8\"></head>\n<body style=\"font-family:Arial,sans-serif;color:#1e293b;max-width:700px;margin:0 auto;padding:20px\">\n  <div style=\"background:#1e3a5f;padding:24px 30px;border-radius:8px 8px 0 0\">\n    <h1 style=\"color:#fff;margin:0;font-size:22px\">Construction Materials Quotation</h1>\n    <p style=\"color:#94a3b8;margin:6px 0 0\">Reference: <strong style=\"color:#e2e8f0\">${refNum}</strong> &nbsp;|&nbsp; Date: ${today}</p>\n  </div>\n  <div style=\"border:1px solid #e2e8f0;border-top:none;padding:24px 30px;border-radius:0 0 8px 8px\">\n    <p>Dear Customer,</p>\n    <p>Thank you for your quotation request. Please find below the detailed pricing for your requested materials.</p>\n    ${estimateDisclaimer}\n    <table style=\"width:100%;border-collapse:collapse;font-size:14px;margin:20px 0\">\n      <thead>\n        <tr style=\"background:#1e3a5f;color:#fff\">\n          <th style=\"padding:10px 12px;text-align:left\">Material</th>\n          <th style=\"padding:10px 12px;text-align:center\">Qty</th>\n          <th style=\"padding:10px 12px;text-align:right\">Unit Price</th>\n          <th style=\"padding:10px 12px;text-align:right\">Subtotal</th>\n          <th style=\"padding:10px 12px;text-align:center\">Status</th>\n        </tr>\n      </thead>\n      <tbody>\n        ${confirmedRows}\n        ${estimatedRows}\n      </tbody>\n      <tfoot>\n        <tr style=\"background:#f1f5f9\">\n          <td colspan=\"3\" style=\"padding:12px;text-align:right;font-weight:700\">GRAND TOTAL</td>\n          <td style=\"padding:12px;text-align:right;font-weight:700;font-size:16px\">₱${grandTotal.toLocaleString('en-PH',{minimumFractionDigits:2})}</td>\n          <td></td>\n        </tr>\n      </tfoot>\n    </table>\n    <p style=\"font-size:13px;color:#64748b\">This quotation is valid for 7 days from the date issued. \n    For questions, please reply to this email or contact our team directly.</p>\n    <p>Best regards,<br><strong>Your Company Construction Team</strong></p>\n  </div>\n</body></html>`;\n\nreturn [{\n  json: {\n    customer_email:   data.customer_email,\n    reference_number: refNum,\n    quote_date:       today,\n    matched_items:    matched,\n    ai_items:         aiItems,\n    matched_subtotal: matchedTotal,\n    ai_subtotal:      aiTotal,\n    grand_total:      grandTotal,\n    has_estimates:    hasEstimates,\n    html_email:       htmlEmail,\n    missing_items:    data.missing_items || []\n  }\n}];"
      },
      "id": "node-build-quote",
      "name": "Build Final Quote",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1780, 300]
    },
    {
      "parameters": {
        "sendTo": "={{ $json.customer_email }}",
        "subject": "={{ 'Your Construction Quotation – ' + $json.reference_number }}",
        "emailType": "html",
        "message": "={{ $json.html_email }}",
        "options": {
          "replyTo": "quotes@yourcompany.com"
        }
      },
      "id": "node-email-customer",
      "name": "Send Quote to Customer (Gmail)",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [2000, 200],
      "credentials": {
        "gmailOAuth2": {
          "id": "REPLACE_WITH_GMAIL_CREDENTIAL_ID",
          "name": "Gmail OAuth2"
        }
      }
    },
    {
      "parameters": {
        "sendTo": "marketing@yourcompany.com",
        "subject": "={{ '[Internal] New Quotation – ' + $json.reference_number + ($json.has_estimates ? ' ⚠ Contains Estimates' : '') }}",
        "emailType": "html",
        "message": "=<h2>New Quotation Submitted</h2>\n<p><strong>Ref:</strong> {{ $json.reference_number }}<br>\n<strong>Customer:</strong> {{ $json.customer_email }}<br>\n<strong>Date:</strong> {{ $json.quote_date }}<br>\n<strong>Grand Total:</strong> ₱{{ $json.grand_total.toLocaleString('en-PH') }}</p>\n<h3>Confirmed Items ({{ $json.matched_items.length }})</h3>\n<pre style=\"background:#f1f5f9;padding:12px;border-radius:4px\">{{ JSON.stringify($json.matched_items, null, 2) }}</pre>\n{% if $json.has_estimates %}\n<h3 style=\"color:#d97706\">⚠ Missing / AI-Estimated Items ({{ $json.ai_items.length }})</h3>\n<pre style=\"background:#fffbeb;padding:12px;border-radius:4px\">{{ JSON.stringify($json.ai_items, null, 2) }}</pre>\n<h3>Original Missing Items Requested</h3>\n<pre style=\"background:#fff1f2;padding:12px;border-radius:4px\">{{ JSON.stringify($json.missing_items, null, 2) }}</pre>\n{% endif %}",
        "options": {}
      },
      "id": "node-email-internal",
      "name": "Internal Notification (Marketing)",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [2000, 420],
      "credentials": {
        "gmailOAuth2": {
          "id": "REPLACE_WITH_GMAIL_CREDENTIAL_ID",
          "name": "Gmail OAuth2"
        }
      }
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "={{ JSON.stringify({ success: true, reference: $json.reference_number, message: 'Your quotation has been sent to ' + $json.customer_email }) }}",
        "options": {
          "responseCode": 200
        }
      },
      "id": "node-webhook-response",
      "name": "Webhook Response",
      "type": "n8n-nodes-base.respondToWebhook",
      "typeVersion": 1.1,
      "position": [2220, 300]
    }
  ],
  "connections": {
    "Webhook Trigger": {
      "main": [
        [{ "node": "Fetch Price List (Google Sheets)", "type": "main", "index": 0 }]
      ]
    },
    "Fetch Price List (Google Sheets)": {
      "main": [
        [{ "node": "Merge Webhook + Sheet Data", "type": "main", "index": 1 }]
      ]
    },
    "Webhook Trigger": {
      "main": [
        [{ "node": "Merge Webhook + Sheet Data", "type": "main", "index": 0 }]
      ]
    },
    "Merge Webhook + Sheet Data": {
      "main": [
        [{ "node": "Cross-Reference Materials", "type": "main", "index": 0 }]
      ]
    },
    "Cross-Reference Materials": {
      "main": [
        [{ "node": "Has Missing Items?", "type": "main", "index": 0 }]
      ]
    },
    "Has Missing Items?": {
      "main": [
        [{ "node": "AI Fallback Pricing Agent", "type": "main", "index": 0 }],
        [{ "node": "Build Final Quote", "type": "main", "index": 0 }]
      ]
    },
    "AI Fallback Pricing Agent": {
      "main": [
        [{ "node": "Parse AI Response", "type": "main", "index": 0 }]
      ]
    },
    "Parse AI Response": {
      "main": [
        [{ "node": "Merge AI Items with Context", "type": "main", "index": 0 }]
      ]
    },
    "Has Missing Items?": {
      "main": [
        [{ "node": "Merge AI Items with Context", "type": "main", "index": 1 }]
      ]
    },
    "Merge AI Items with Context": {
      "main": [
        [{ "node": "Build Final Quote", "type": "main", "index": 0 }]
      ]
    },
    "Build Final Quote": {
      "main": [
        [
          { "node": "Send Quote to Customer (Gmail)", "type": "main", "index": 0 },
          { "node": "Internal Notification (Marketing)", "type": "main", "index": 0 }
        ]
      ]
    },
    "Send Quote to Customer (Gmail)": {
      "main": [
        [{ "node": "Webhook Response", "type": "main", "index": 0 }]
      ]
    }
  },
  "settings": {
    "executionOrder": "v1",
    "saveManualExecutions": true,
    "callerPolicy": "workflowsFromSameOwner",
    "errorWorkflow": "REPLACE_WITH_ERROR_WORKFLOW_ID"
  },
  "staticData": null,
  "tags": ["construction", "quotation", "automation"],
  "triggerCount": 1,
  "updatedAt": "2026-01-01T00:00:00.000Z",
  "versionId": "1"
}
```

---

## Post-Import Checklist

### Credentials to Replace
| Placeholder | What to Set |
|---|---|
| `REPLACE_WITH_GSHEETS_CREDENTIAL_ID` | Your Google Sheets OAuth2 credential |
| `REPLACE_WITH_OPENAI_CREDENTIAL_ID` | Your OpenAI API key credential |
| `REPLACE_WITH_GMAIL_CREDENTIAL_ID` | Your Gmail OAuth2 credential (used twice) |
| `REPLACE_WITH_YOUR_GOOGLE_SHEET_ID` | The Sheet ID from your Google Sheets URL |
| `REPLACE_WITH_ERROR_WORKFLOW_ID` | Optional: a separate n8n error-handler workflow |

### Google Sheet Expected Format
| Material Name | Unit Price | Availability |
|---|---|---|
| Portland Cement (40kg) | 285 | Yes |
| Steel Bar 10mm | 420 | No |
| Hollow Blocks | 18 | Yes |

### Sample Webhook Test Payload
```json
{
  "customer_email": "client@example.com",
  "requested_materials": [
    { "name": "Portland Cement (40kg)", "quantity": 50 },
    { "name": "Tempered Glass 6mm", "quantity": 10 },
    { "name": "Steel Bar 10mm", "quantity": 100 }
  ]
}