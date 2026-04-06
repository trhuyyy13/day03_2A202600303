# Individual Report: Lab 3 - Chatbot vs ReAct Agent

- **Student Name**: Tran Quang Huy
- **Student ID**: 2A202600303
- **Date**: 2026-04-06
- **Role**: 🛠️ **Tool Design & API Integration**

---

## I. Technical Contribution (15 Points)

### Core Responsibility: Tool Design, API Integration & Prompt Testing

Our main project group consists of 6 members, divided into 3 sub-teams. My sub-team includes myself and Sơn TN (Student ID: 2A202600430). Together, we collaboratively designed and implemented all **3 external tools** that the ReAct agent uses to interact with the real world, along with the central tool registry. In addition to tool construction, we also conducted tests on several agent versions to ensure they properly integrate the **5 core Production-Grade components** (Identity, Capabilities, Instructions, Constraints, and Output Format) into the system prompt.

- **Modules Implemented**:
  - `src/tools/check_weather.py` — Weather forecast via OpenWeatherMap API
  - `src/tools/search_hotels.py` — Hotel search via SerpAPI Google Hotels
  - `src/tools/search_activities.py` — Activity recommendations via SerpAPI Local Results
  - `src/tools/tool_registry.py` — Central registry mapping tool names to functions
  - `src/tools/__init__.py` — Package initialization

### Code Highlights

#### 1. `check_weather.py` — Real-time Weather API with Fallback

```python
def check_weather(location: str, date: str) -> str:
    """
    Check weather forecast using OpenWeatherMap API.
    Returns condition (Rain/Clear/Clouds), temperature, humidity, wind speed.
    Falls back to simulated data if API key is unavailable.
    """
    params = {
        "q": location,
        "appid": OPENWEATHER_API_KEY,
        "units": "metric",
        "cnt": 40  # 5-day forecast, every 3 hours
    }
    response = requests.get(BASE_URL, params=params, timeout=10)
    
    # Find forecast closest to requested date (noon)
    target_noon = target_date.replace(hour=12)
    best_forecast = min(data["list"], key=lambda e: abs(
        datetime.fromtimestamp(e["dt"]) - target_noon
    ).total_seconds())
```

**Key design decisions:**
- **Fallback data**: If the API key is missing or invalid, the tool returns realistic simulated data for 9 Vietnamese cities. This ensures the agent still works during development/testing.
- **Rain detection**: The tool explicitly states whether rain is expected, providing a clear signal for the agent's branching logic.

#### 2. `search_hotels.py` — Budget-aware Hotel Search

```python
def search_hotels(location: str, max_price: float) -> str:
    """
    Search for hotels under max_price VND per night.
    Uses SerpAPI Google Hotels with Vietnamese locale (hl=vi, gl=vn).
    """
    params = {
        "engine": "google_hotels",
        "q": f"hotels in {location}",
        "check_in_date": _get_next_saturday(),
        "check_out_date": _get_next_sunday(),
        "currency": "VND",
        "api_key": SERPAPI_API_KEY,
    }
```

**Key design decisions:**
- Automatic date calculation (`_get_next_saturday()`) so "cuối tuần này" always maps correctly
- Price filtering: accepts VND amounts and filters server-side results
- Fallback: 4 cities × 3-8 hotels with realistic names and prices

#### 3. `search_activities.py` — Weather-conditional Recommendations

```python
def search_activities(location: str, weather_condition: str) -> str:
    is_rainy = weather_condition.lower() in ["rain", "drizzle", "thunderstorm"]
    if is_rainy:
        query_type = "quán cafe đẹp, bảo tàng"       # Indoor
    else:
        query_type = "địa điểm tham quan ngoài trời"  # Outdoor
```

This is where the **branching logic** manifests — the same tool returns completely different results based on weather input from a previous tool call.

#### 4. `tool_registry.py` — Tool Description Engineering

```python
{
    "name": "check_weather",
    "description": (
        "Check weather forecast for a specific location and date. "
        "Takes two arguments: location (city name, e.g. 'Da Lat') "
        "and date (YYYY-MM-DD format, within 5 days). "
        "IMPORTANT: The returned condition indicates if it will rain — "
        "use this to decide between outdoor vs indoor activities."
    ),
    "function": check_weather,
    "parameters": { ... }
}
```

I learned that **tool descriptions are the most critical prompt engineering** — the LLM only knows a tool through its string description. The v2 descriptions are 5x longer than v1 but resulted in 0 argument errors.

### Documentation

My tools interact with the ReAct loop as follows:
1. Agent's `_execute_tool()` method looks up tool name in the registry
2. It calls `tool["function"](**args)` with the parsed JSON arguments
3. The return string becomes the `Observation` appended to the conversation
4. The LLM reads this Observation to decide the next step (branching)

---

## II. Debugging Case Study (10 Points)

### Case Study 1: OpenWeatherMap API Returns 401 → Agent Retries Redundantly

- **Problem Description**: During the first successful agent run, the `check_weather` tool returned an API error: `401 Client Error: Unauthorized`. The tool correctly fell back to simulated data, but the agent (v1) then **retried the same tool call**.
- **Log Source** (from `logs/2026-04-06.log`, lines 19-23):
```json
{"event": "TOOL_CALL", "data": { "step": 1, "tool": "check_weather", "observation_preview": "API Error: 401 ... Using fallback data." }}
```
- **Diagnosis**: The issue was twofold. First, the OpenWeatherMap API key was invalid. Second, the v1 system prompt lacked constraints, so the LLM instinctively retried the failed tool call.
- **Solution**: Fixed the API key in `.env` and added a strict guardrail in the v2 system prompt: *"If a tool returns an error, DO NOT retry the same call."*

### Case Study 2: Date Mapping Bug (Output Date Mismatch)

- **Problem Description**: When tracking agent failures, we noticed a query requesting the weather for `2024-05-20` resulted in `check_weather` returning data for `"2026-04-06 09:00:00"`. The output date simply did not match the requested input date.
- **Diagnosis**: This is a mapping bug or an improper fallback behavior. The OpenWeatherMap API (or our fallback logic) focuses strictly on current and near-future dates (e.g., 5-day forecast). When the agent requested an invalid or past date, the system defaulted to returning the current/nearest date's data instead of throwing a validation error, misleading the LLM.
- **Solution / Learnings**: The `check_weather` tool needs input validation middleware. It should verify if the requested date is within the allowable range (present to +5 days). If not, it should throw a descriptive error (e.g., "Error: Date out of range") instead of returning mismatched realistic data.

### Case Study 3: Empty Search Results (No-Result vs. Technical Failure)

- **Problem Description**: During some tool executions for `search_activities`, `search_hotels`, or restaurant queries, the tool returned an empty array `[]`.
- **Diagnosis**: While this initially looked like a technical failure, checking the traces revealed that the API call actually succeeded (HTTP 200). The real issue was that the search query generated by the LLM was either too specific or yielded no local results (bad search quality). Returning `[]` is a valid "no-result" scenario rather than a tool crash.
- **Solution / Learnings**: To handle this gracefully, the tool should intercept empty arrays and return a clear observation to the agent (e.g., "No results found. Try a broader search term."), prompting the ReAct loop to refine its search query rather than terminating or hallucinating.

---

## III. Personal Insights: Chatbot vs ReAct (10 Points)

1. **Reasoning**: The `Thought` block is where the agent demonstrates **understanding of the user's conditional requirements**. For example, in the multi-step query, the Thought after receiving weather data was: *"Since the weather is cloudy with no rain, I will proceed to find a hotel AND outdoor activities."* A chatbot can't do this — it has no intermediate reasoning step and no way to branch based on real data.

2. **Reliability**: The Agent performed **worse than the Chatbot on simple Q&A** ("Đà Lạt nổi tiếng với gì?"). The chatbot answered in 4,095ms / 383 tokens, while the agent took 5,825ms / 2,736 tokens for the same quality answer. The overhead comes from the larger system prompt (tool descriptions, few-shot examples) that's included even when no tools are needed. For trivial queries, a chatbot is simply more efficient.

3. **Observation**: The Observation is the **ground truth** that prevents hallucination. When `check_weather` returned "Clouds, 29°C, No rain", this concrete data point forced the agent to take the "outdoor" branch. Without observations, the LLM might have guessed the weather (and guessed wrong). I saw this happen in v1 where the agent **hallucinated** a fake Observation instead of waiting for the real tool result.

---

## IV. Future Improvements (5 Points)

- **Scalability**: Implement **parallel tool calls** — when the agent needs both hotels and activities (after weather check), call both APIs simultaneously using `asyncio.gather()` instead of sequentially. This would cut the multi-step latency from ~5.5s to ~3.5s.

- **Safety**: Add **input validation middleware** — sanitize all tool arguments before execution. For example, validate that `date` is in YYYY-MM-DD format and within 5 days, validate that `max_price` is a positive number under 100 million VND. Currently the tools trust whatever the LLM provides.

- **Performance**: Implement **response caching** — weather forecasts for the same city+date don't change within 3 hours, and hotel prices don't change within 1 hour. Cache tool results with TTL to avoid redundant API calls and reduce latency/cost. This is especially important given SerpAPI's 100 searches/month free tier limit.

---
