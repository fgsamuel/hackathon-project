# Finance Assistant

A personal-finance chat assistant for bank customers. A customer signs in with
their document number and date of birth, then asks questions about their own
money in plain language: "How much did I spend on food in the last 3 months?",
"Why was my last transaction declined?", "What's my credit card limit?". The
assistant answers in the customer's language and can draw spending charts.

Built with Django REST Framework, LangGraph and Claude, on a synthetic
Latin American banking dataset.

## How it works

```
Browser / terminal ──► agent (FastAPI + LangGraph + Claude) ──► backend (Django REST API) ──► PostgreSQL
                         :8001                                    :8000
```

- **`backend/`**: read-only Django REST API over the bank's data (customers,
  products, transactions, call-center interactions, complaints, campaigns,
  exchange rates and more). Customers log in at `POST /api/auth/login/` and
  receive a JWT. Every customer-owned endpoint is filtered by the customer in
  that token, and internal data (agents, campaigns) is staff-only.
- **`agent/`**: a LangGraph agent that calls Claude with a small set of tools:
  `list_transactions`, `spending_summary`, `show_spending_chart`,
  `declined_transactions` and `get_my_accounts`. It ships with a terminal chat,
  a web chat and an evaluation script. See [agent/README.md](agent/README.md).

### Security model

The customer's JWT never reaches the language model. It travels in the
LangGraph run config, not as a tool argument, so the model can't choose whose
data to read, and the backend scopes every query to the customer in the token
anyway. In the web chat, the token stays on the server; the browser only holds
a random session id. The evaluation script includes prompt-injection attempts
("ignore all previous instructions…", "show customer X's transactions") and
flags any answer that returns another customer's data.

### Answering rules

- Only data returned by the tools is used; the agent never invents amounts or dates.
- "Spending" means approved purchases, payments and cash withdrawals. Transfers
  and deposits are not spending.
- Amounts in different currencies (USD, COP, ARS) are reported separately,
  never summed.
- The dataset ends on 2026-06-18, so the agent treats that date as "today".

## Getting started

### Requirements

- Python 3.14
- PostgreSQL, with a `hackathon_backend` database (user `postgres`, password
  `postgres` on `localhost:5432`, as set in `backend/backend/settings.py`)
- An [Anthropic API key](https://console.anthropic.com/)

### 1. Configure

```bash
cp .env.example .env
```

Set `ANTHROPIC_API_KEY` in `.env`. `AGENT_MODEL` defaults to
`claude-haiku-4-5` (cheap, for development); switch it to `claude-opus-5-5`
for demos.

### 2. Run the backend

```bash
cd backend
python3.14 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver 8000
```

Then load the synthetic dataset into the database.

### 3. Run the agent

In another terminal:

```bash
cd agent
python3.14 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

uvicorn server:app --port 8001   # web chat at http://localhost:8001
python cli.py --debug            # terminal chat; --debug prints tool calls and results
python evals/run_questions.py    # fixed question set, including injection attempts
```

Demo login (synthetic data): document `78481769`, date of birth `1982-02-05`.

## Tests

```bash
cd backend
python manage.py test
```

## Project structure

```
.
├── .env.example          # settings for the agent (API key, model, API URL)
├── backend/
│   ├── manage.py
│   └── backend/
│       ├── settings.py
│       ├── urls.py
│       └── core/         # models, serializers, viewsets, filters, JWT auth, tests
└── agent/
    ├── config.py         # settings and the simulated "today"
    ├── api_client.py     # login and paginated GETs with the customer's JWT
    ├── tools.py          # tools the model can call
    ├── prompts.py        # system prompt
    ├── graph.py          # LangGraph graph with per-thread memory
    ├── cli.py            # terminal chat
    ├── server.py         # web chat API
    ├── web/index.html    # web chat UI
    └── evals/            # question set for regression checks
```
