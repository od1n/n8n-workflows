# n8n workflows — self-hosted job finder and inbox evaluator

Two production workflows exported from my self-hosted n8n (v2.x, Docker on a Hetzner VPS behind EasyPanel).
They have been running daily since August 2026. Exported with the credentials removed; see
[How credentials stay out of the export](#how-credentials-stay-out-of-the-export).

| File | Trigger | What it does |
|---|---|---|
| `workflows/job-finder.json` | Schedule (every 15 min) + sub-workflow trigger | Pulls new freelance/job postings from Freelancer.com (REST API), Work Remoto (WordPress REST) and Upwork e-mail alerts, normalises them to one shape, converts budgets to USD, filters by keywords and bid count, de-duplicates against persisted state, scores each posting 1–10 with Claude and e-mails a ranked digest with a draft proposal. |
| `workflows/inbox-message-evaluator.json` | IMAP (Gmail) | Watches the inbox: Fiverr buyer messages are classified by Claude (real client / likely scam / irrelevant, with confidence and reasons) and the verdict is e-mailed and pushed to WhatsApp; Upwork job alerts are routed to the job-finder sub-workflow. |

## job-finder.json

```
Schedule ─┬─ Freelancer API ──► Normalizar Freelancer ─┐
          ├─ Work Remoto API ─► Normalizar Work Remoto ─┼─► Filtrar y deduplicar ─► Preparar scoring ─► Claude API ─► Parsear score ─► Correo
          └─ (RemoteOK, disabled)                       │
Sub-workflow trigger (Upwork alert from the evaluator) ─► Normalizar Upwork ─┘
```

* **Config** node (Code): the only place to edit — recipient e-mail, keyword lists, max bid count, score threshold, lines of business. Everything downstream reads `$('Config')`.
* **Normalizar \*** nodes (Code, Node.js): each source has a different payload; these map it to `{fuente, titulo, descripcion, presupuestoUSD, ofertas, url, publicado}`. Budgets in INR/GBP/EUR are converted with a cached exchange-rate call.
* **Filtrar y deduplicar** (Code): hard filters, then de-duplication using `$getWorkflowStaticData('global')` so a posting is never e-mailed twice across runs. Static data is only written on production runs, not on manual test runs.
* **Preparar scoring / Claude API / Parsear score**: builds one Anthropic Messages request per posting (HTTP Request node with a stored header credential), asks for strict JSON (`score`, `linea`, `razones`, `propuesta`), parses it defensively and drops anything under the threshold.
* **Correo**: Gmail/SMTP digest with `[score] [line] [N bids] title` subjects so the inbox is scannable.

## inbox-message-evaluator.json

```
IMAP trigger ─┬─ Filtrar remitente Fiverr ─► Evaluar mensaje con Claude ─► Parsear evaluacion ─┬─► Enviar veredicto por correo
              │                                                                              └─► (real client only) WhatsApp
              └─ ¿Alerta de Upwork? ─► Pasar alerta al buscador (Execute Workflow)
```

* The IMAP trigger only reacts to new, unread messages; the sender filters keep the two branches cheap.
* **Parsear evaluacion** (Code) is defensive: the model is told to return JSON only, but the node still strips code fences, extracts the first `{...}` block and falls back to a "could not parse" verdict instead of crashing the run.
* WhatsApp delivery uses an HTTP Request node against an Evolution API instance (host and instance name replaced with placeholders in the export).

## How credentials stay out of the export

* API keys, OAuth tokens and the IMAP password live only in n8n's credential store (encrypted with `N8N_ENCRYPTION_KEY`). Nodes reference them by credential id; nothing is typed into node parameters.
* The HTTP Request nodes use *Generic Credential Type → Header Auth / Query Auth* for Anthropic, Freelancer and Evolution API, so the key is injected at run time.
* Export = `GET /rest/workflows/:id` followed by removing the `credentials`, `id` and `webhookId` keys from every node, then a grep for key-like strings (`sk-ant-`, `AIza`, `apikey=`) and for personal e-mail/phone values before committing. That is the script-free equivalent of n8n's own "Download" minus the credential names.
* To import: *Workflows → Import from file*, then open each HTTP/IMAP/Gmail node and pick your own credentials. The evaluator's "Pasar alerta al buscador" node needs the id of your imported job-finder workflow.

## How it is tested

Honestly: manually, end to end. Each change is run from the editor with the *Execute workflow* button on captured inputs (pinned data for the API responses), then with one real test e-mail for the IMAP path, and the run is inspected in *Executions*. The Code nodes are small and pure where possible (normalisers take an item, return an item) so that a failing run points at one node. There is no automated test suite for these workflows yet; the Python tools in my other repositories are where I write tests first.

## Requirements

n8n ≥ 1.80 (tested on 2.21), an Anthropic API key, an IMAP mailbox and optionally a Freelancer.com OAuth token and an Evolution API instance. Nothing else.
