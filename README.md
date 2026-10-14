# ✈️ AI-TripPlanner — A Multi-Agent Travel Planner with LangGraph

AI-TripPlanner turns a plain-English trip request like *"Plan a 7 days Japan trip from Bangladesh under 2 lakhs"* into a complete travel plan: live flight information, hotel suggestions, a day-by-day itinerary and an estimated budget. Four agents built with LangGraph each handle one step, and a FastAPI web app puts them behind a simple chat-style page.

## Why this project?

Planning a trip usually means jumping between flight sites, hotel searches and a notes app. AI-TripPlanner puts all of that into one request. Four agents work in sequence:

- a **flight agent** that works out the route and fetches live flights,
- a **hotel agent** that searches the web for places to stay,
- an **itinerary agent** that builds a day-by-day plan, and
- a **final agent** that turns everything into one clear response.

## Features

- ✈️ **Live flight data** from AviationStack, with route detection from natural language ("Japan trip from Bangladesh" → `DAC → NRT`)
- 🏨 **Hotel suggestions** from Tavily web search
- 🧠 **Multi-agent workflow** coordinated by LangGraph
- 📝 **Structured itineraries**: trip summary, flights, hotels, day-by-day plan, budget and recommendations
- 💾 **Conversation memory** stored in PostgreSQL through LangGraph checkpoints
- 🌐 **Web interface** with quick-prompt buttons, Markdown rendering, copy-to-clipboard and PDF download
- 🔍 **Optional tracing** with LangSmith
- ⚡ **Fast LLM responses** from Groq

## Tech Stack

| Layer | Tools |
|---|---|
| Backend | Python 3.10+, FastAPI, Uvicorn |
| Agents | LangGraph, LangChain, Groq (`openai/gpt-oss-120b` by default) |
| Data sources | AviationStack (flights), Tavily (hotels) |
| Storage | PostgreSQL 17 (Docker), pgAdmin |
| Frontend | Jinja2, HTML, CSS, JavaScript, marked.js, html2pdf.js |

## How It Works

```text
START → flight_agent → hotel_agent → itinerary_agent → final_agent → END
```

1. **Flight agent:** works out the origin and destination from your message and fetches live flights from AviationStack.
2. **Hotel agent:** searches Tavily for hotels at the destination.
3. **Itinerary agent:** sends the flight and hotel results to the LLM to build a practical, budget-aware plan.
4. **Final agent:** formats everything into the final response.

After each step, the full state is saved to PostgreSQL under a `thread_id`. Reusing the same `thread_id` keeps the conversation history, and it survives server restarts.

## Project Structure

```text
.
├── app.py                # FastAPI app: web page + API routes
├── backend.py            # LangGraph workflow, agents and Postgres checkpointer
├── docker-compose.yml    # PostgreSQL + pgAdmin containers
├── requirements.txt      # Python dependencies
├── static/
│   ├── script.js         # Frontend logic (requests, rendering, PDF export)
│   └── style.css         # Styles
├── templates/
│   └── index.html        # Main page
└── tools/
    ├── flight_tool.py    # AviationStack search + route parsing
    └── tavily_tool.py    # Tavily hotel search
```

## Prerequisites

- Python 3.10 or newer
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) for PostgreSQL
- API keys for:
  - [Groq](https://console.groq.com/keys)
  - [Tavily](https://app.tavily.com/)
  - [AviationStack](https://aviationstack.com/) (the free plan allows about 100 requests a month)
  - [LangSmith](https://smith.langchain.com/) *(optional, for tracing)*

## Setup

### 1. Clone the repository

```bash
git clone https://github.com/prateek-developer/Ai-Travel-Planner.git
cd Ai-Travel-Planner
```

### 2. Create a `.env` file

Create a `.env` file in the project root:

```env
# --- Database (used by docker-compose and the app) ---
POSTGRES_PASSWORD=choose_a_db_password
PGADMIN_PASSWORD=choose_a_pgadmin_password

# --- API keys ---
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
AVIATIONSTACK_API_KEY=your_aviationstack_api_key

# --- Optional ---
GROQ_MODEL=openai/gpt-oss-120b      # any chat model your Groq account offers
DEFAULT_ORIGIN_IATA=DEL             # used when a request has no origin (airport code or city)

# --- Optional: LangSmith tracing ---
LANGSMITH_TRACING=true              # must be lowercase "true"
LANGSMITH_ENDPOINT=https://api.smith.langchain.com
LANGSMITH_API_KEY=your_langsmith_api_key
LANGSMITH_PROJECT=AI-TripPlanner
```

The app connects to `travel_user@localhost:5432/travel_planner` by default, which matches `docker-compose.yml`. To use a different database, set `DATABASE_URL=postgresql://user:password@host:5432/dbname`.

### 3. Start PostgreSQL

```bash
docker compose up -d
docker ps          # wait until travel-planner-db shows "(healthy)"
```

| Service | Address |
|---|---|
| PostgreSQL | `localhost:5432` |
| pgAdmin | http://localhost:5050 (log in with `admin@example.com` and your `PGADMIN_PASSWORD`) |

To browse the database in pgAdmin, add a server with host **`postgres`** (the Docker service name, not `localhost`), port `5432`, user `travel_user` and your `POSTGRES_PASSWORD`.

### 4. Install Python dependencies

```bash
python -m venv .venv
source .venv/bin/activate        # On Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

## Running the App

```bash
python app.py
```

Open **http://127.0.0.1:8000** in your browser, type a trip request or click one of the quick prompts, and press **Generate Plan** (or **Ctrl + Enter**). A full plan usually takes about a minute, because the app calls three external services and the LLM twice.

On first start, the app creates the LangGraph checkpoint tables in PostgreSQL automatically.

## API

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Web interface |
| `GET` | `/health` | Health check |
| `POST` | `/api/travel` | Generate a travel plan |

**Request**

```bash
curl -X POST http://127.0.0.1:8000/api/travel \
  -H "Content-Type: application/json" \
  -d '{"message": "Plan a 5 days Dubai trip from Delhi with flights and hotels"}'
```

Send back the `thread_id` from a previous response to continue the same conversation:

```json
{ "message": "Make it cheaper", "thread_id": "user_3f2a..." }
```

**Response**

```json
{
  "success": true,
  "thread_id": "user_3f2a...",
  "answer": "## Trip Summary ...",
  "flight_results": "Live flights from DEL to DXB ...",
  "hotel_results": "1. **Best hotels in Dubai** ...",
  "itinerary": "Day 1 ...",
  "llm_calls": 4
}
```

## Example Prompts

- `Plan a complete 7 days Japan trip from Bangladesh including flights, hotels and sightseeing under 2 lakhs.`
- `Plan a 5 days Dubai trip from Dhaka with flights, hotels and sightseeing.`
- `Plan a 3 days trip from Mumbai to Singapore.`
- `Show flights from DEL to DXB.`
- `Give me all country flight info.` *(global live flights)*

## Limitations

- **No ticket prices.** AviationStack provides live flight status and schedules, not fares. For prices, add a pricing API such as Amadeus.
- **Free-tier limits.** The AviationStack free plan allows about 100 requests a month, and every trip request uses one.
- **Follow-up questions.** Conversation history is saved, but the itinerary and final agents currently use only the latest message, so a follow-up like "make it cheaper" is planned from scratch.

## Troubleshooting

| Problem | Fix |
|---|---|
| `connection timeout expired` on startup | Docker Desktop or the database container isn't running. Run `docker compose up -d`. |
| `model_not_found` from Groq | The model was retired. Set `GROQ_MODEL` in `.env` to a model listed in your Groq console. |
| No traces in LangSmith | `LANGSMITH_TRACING` must be lowercase `true`. Restart the app after changing `.env`. |
| `Flight API error: AVIATIONSTACK_API_KEY is missing` | Add the key to `.env` and restart. |

## Contributing

Contributions are welcome:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-feature`)
3. Commit your changes
4. Open a pull request

## License

This project is licensed under the [MIT License](LICENSE).
