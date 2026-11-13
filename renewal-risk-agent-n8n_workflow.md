# AI Renewal & Revenue Risk Management (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `renewal-risk-aiagent-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "AI Renewal & Revenue Risk Management (AI Agent, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## Setup (n8n Cloud)\n1. Credentials (free tiers): Google Gemini, Slack, Postgres (Supabase)\n2. Create table renewal_risks in Supabase SQL Editor\n3. Slack channel: revenue-risk (change in the Slack node)\n4. Click Execute workflow: 4 mock accounts scored by rules, then AI writes the retention plan\nReal client use: replace Mock Accounts with usage, support and payment data from your systems\nAI Agent node (Gemini) (no tools, JSON answer)",
        "height": 260,
        "width": 560
      },
      "id": "256147fe-8fbe-4595-9e53-d2bd44297acf",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -220
      ]
    },
    {
      "parameters": {},
      "id": "a60c1125-3d78-436d-9c87-1882b9cc6135",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return [\n  { json: { account: 'Contoso Retail', arr: 48000, days_to_renewal: 21, usage_change_pct: -45, tickets_30d: 7, payment_status: 'failed' } },\n  { json: { account: 'Adventure Works', arr: 120000, days_to_renewal: 60, usage_change_pct: 12, tickets_30d: 1, payment_status: 'on_time' } },\n  { json: { account: 'Litware Inc', arr: 25000, days_to_renewal: 35, usage_change_pct: -20, tickets_30d: 3, payment_status: 'late' } },\n  { json: { account: 'Proseware', arr: 9000, days_to_renewal: 14, usage_change_pct: 0, tickets_30d: 0, payment_status: 'on_time' } }\n];"
      },
      "id": "ed4d0cce-fb98-49bd-a85a-b2c748d92287",
      "name": "Mock Accounts",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return $input.all().map((it) => {\n  const a = it.json;\n  let score = 0;\n  const why = [];\n  if (a.usage_change_pct <= -30) { score += 40; why.push('usage down ' + Math.abs(a.usage_change_pct) + '%'); }\n  else if (a.usage_change_pct <= -10) { score += 20; why.push('usage down ' + Math.abs(a.usage_change_pct) + '%'); }\n  if (a.tickets_30d >= 5) { score += 25; why.push(a.tickets_30d + ' support tickets in 30 days'); }\n  else if (a.tickets_30d >= 3) { score += 12; why.push(a.tickets_30d + ' support tickets in 30 days'); }\n  if (a.payment_status === 'failed') { score += 30; why.push('payment failed'); }\n  else if (a.payment_status === 'late') { score += 15; why.push('late payment'); }\n  if (a.days_to_renewal <= 30) { score += 5; why.push('renews in ' + a.days_to_renewal + ' days'); }\n  const level = score >= 60 ? 'high' : score >= 30 ? 'medium' : 'low';\n  return { json: Object.assign({}, a, { risk_score: score, risk_level: level, risk_factors: why.join('; ') || 'no warning signs' }) };\n});"
      },
      "id": "75be0427-6c55-4082-802f-542d505871eb",
      "name": "Score Risk",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        440,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=You are a customer retention strategist. For the account below write a one-sentence recommendation for the account manager and a short, friendly message draft to the customer. Do not mention internal risk scores in the message.\n\nAccount: {{ $json.account }}\nAnnual recurring revenue: {{ $json.arr }}\nRenews in: {{ $json.days_to_renewal }} days\nRisk level: {{ $json.risk_level }}\nWarning signs: {{ $json.risk_factors }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a customer retention strategist. Respond with ONLY a JSON object, no markdown, no extra text: {\"recommendation\":\"one sentence for the account manager\",\"message_draft\":\"short friendly message to the customer\"}"
        }
      },
      "id": "dba89283-d3ae-49c8-9983-8c58862833f7",
      "name": "AI Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        680,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "7d1ec1f9-f673-4ff5-a0a0-41a5e013f381",
      "name": "Google Gemini Chat Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        660,
        420
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Score Risk').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nreturn { json: {\n  account: ctx.account,\n  arr: ctx.arr,\n  risk_score: ctx.risk_score,\n  risk_level: ctx.risk_level,\n  days_to_renewal: ctx.days_to_renewal,\n  risk_factors: ctx.risk_factors,\n  recommendation: String(r.recommendation || '').slice(0, 400),\n  message_draft: String(r.message_draft || '').slice(0, 800)\n}};"
      },
      "id": "9271ad17-74fe-48e0-b32e-891595ca3183",
      "name": "Parse Advice",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1040,
        200
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "3ff14162-2939-40d4-9889-bc4931e1c3fc",
              "leftValue": "={{ $json.risk_level }}",
              "rightValue": "high",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "a247136b-6748-4a5a-911a-9afdd4501007",
      "name": "High Risk?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        1260,
        200
      ]
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "revenue-risk",
          "mode": "name"
        },
        "text": "=:rotating_light: *High renewal risk: {{ $json.account }}*\nARR: {{ $json.arr }} | Renews in {{ $json.days_to_renewal }} days | Score: {{ $json.risk_score }}\nSigns: {{ $json.risk_factors }}\nNext step: {{ $json.recommendation }}",
        "otherOptions": {}
      },
      "id": "4dfa4ad9-094c-4246-a4a9-2d827e927ab4",
      "name": "Slack Alert",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        1520,
        120
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "renewal_risks",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "account": "={{ $json.account }}",
            "arr": "={{ $json.arr }}",
            "risk_score": "={{ $json.risk_score }}",
            "risk_level": "={{ $json.risk_level }}",
            "days_to_renewal": "={{ $json.days_to_renewal }}",
            "risk_factors": "={{ $json.risk_factors }}",
            "recommendation": "={{ $json.recommendation }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "account",
              "displayName": "account",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "arr",
              "displayName": "arr",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "risk_score",
              "displayName": "risk_score",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "risk_level",
              "displayName": "risk_level",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "days_to_renewal",
              "displayName": "days_to_renewal",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "risk_factors",
              "displayName": "risk_factors",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "recommendation",
              "displayName": "recommendation",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "31e13466-0a92-4023-8265-ecfd6f1e8555",
      "name": "Save Risk",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        1520,
        320
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    }
  ],
  "connections": {
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Mock Accounts",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Mock Accounts": {
      "main": [
        [
          {
            "node": "Score Risk",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Score Risk": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Parse Advice": {
      "main": [
        [
          {
            "node": "High Risk?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "High Risk?": {
      "main": [
        [
          {
            "node": "Slack Alert",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Risk",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Save Risk",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Gemini Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "AI Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "AI Agent": {
      "main": [
        [
          {
            "node": "Parse Advice",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
