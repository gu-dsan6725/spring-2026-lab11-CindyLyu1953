# A2A discovery and invocation — observations (from agent terminal logs)

## Sequence of interactions

1. **Client → Travel Assistant (A2A)** — The discovery test posts JSON-RPC `message/send` to `http://127.0.0.1:10001/`. The assistant runs local tools first (e.g. `search_flights` for NYC–LAX).
2. **Travel Assistant → Registry (HTTP, not A2A)** — The model calls `discover_remote_agents`. The Travel Assistant POSTs to the registry semantic endpoint with a natural-language query.
3. **Registry → Travel Assistant** — The stub logs the query and returns a canned match for the Flight Booking Agent.

**Evidence (Registry Stub):**

```text
Semantic search request: query='flight booking reservation payment processing confirm seats', max_results=5
Returning canned response: Flight Booking Agent
127.0.0.1:57296 - "POST /api/agents/discover/semantic?query=...&max_results=5 HTTP/1.1" 200 OK
```

4. **Travel Assistant** — Parses the discovery result, creates `RemoteAgentClient` for `Flight Booking Agent` (id `/flight-booking-agent`), and caches it.

**Evidence (Travel Assistant):**

```text
Tool called: discover_remote_agents(query='flight booking reservation payment processing confirm seats', max_results=5)
Semantic search: 'flight booking reservation payment processing confirm seats' (max_results=5)
Found 1 agents
Created RemoteAgentClient for: Flight Booking Agent (ID: /flight-booking-agent, Skills: 5)
Cached agent: Flight Booking Agent (ID: /flight-booking-agent)
Discovery successful: found 1 agents, cached 1 new
```

5. **Travel Assistant → Flight Booking (well-known URI)** — Before the first A2A call, the client fetches the peer’s agent card over HTTP.

**Evidence (Flight Booking):**

```text
127.0.0.1:57299 - "GET /.well-known/agent-card.json HTTP/1.1" 200 OK
```

**Evidence (Travel Assistant — card resolver log):**

```text
Successfully fetched agent card data from http://127.0.0.1:10002/.well-known/agent-card.json: {'capabilities': {'streaming': True}, ... 'skills': [{'id': 'check_availability', ...}, ...], 'url': 'http://127.0.0.1:10002/', ...}
A2A client initialized for Flight Booking Agent
```

6. **Travel Assistant → Flight Booking (A2A / JSON-RPC)** — `invoke_remote_agent` sends `message/send` to the booking agent root URL; the booking server accepts `POST /` and runs the agent (LiteLLM + tools).

**Evidence (Travel Assistant — outbound JSON-RPC):**

```text
Tool called: invoke_remote_agent(agent_id='/flight-booking-agent', message='I need to book flight ID 1 for 2 seats...')
A2A REQUEST to Flight Booking Agent:
{
  "jsonrpc": "2.0",
  "method": "message/send",
  "params": {
    "message": {
      "role": "user",
      "parts": [
        {
          "kind": "text",
          "text": "I need to book flight ID 1 for 2 seats. Please reserve the seats, confirm the reservation, and process the payment."
        }
      ],
      "messageId": "f510454c8c5049e5a2a5df35ef428f43"
    }
  }
}
Successfully invoked Flight Booking Agent
```

**Evidence (Flight Booking — same connection / time window):**

```text
127.0.0.1:57299 - "POST / HTTP/1.1" 200 OK
```

*(Neither service terminal prints the full JSON-RPC **response** body for this hop; completion is implied by HTTP 200 on `POST /` and the Travel Assistant’s “Successfully invoked” line.)*

---

## JSON-RPC request and response format (from logs + correlation)

**Request (verbatim structure from Travel Assistant logs):** `jsonrpc` `2.0`, `method` `message/send`, `params.message` with `role`, `parts` (`kind` + `text`), and `messageId`. No top-level `id` appears in the logged outbound peer payload (the Strands/A2A client may supply it on the wire).

**Response:** The three agent terminals do not log the JSON response document. By A2A/Strands behavior in this lab, a successful round-trip returns HTTP 200 on `POST /` and a JSON body whose `result` includes a **task** (`kind`, `status.state`, `history`) and **artifacts** with consolidated `text` — the same pattern used when clients call either agent directly.

---

## Agent card: what appears and how it is used

From the Travel Assistant log line after `GET /.well-known/agent-card.json`, the card includes:

- **Identity:** `name`, `description`, `url`, `version`
- **Protocol:** `protocolVersion` (e.g. `0.3.0`), `preferredTransport` (`JSONRPC`)
- **Capabilities / modes:** `capabilities.streaming`, `defaultInputModes` / `defaultOutputModes` (`text`)
- **Skills:** list of `{ id, name, description, tags }` for `check_availability`, `reserve_flight`, `confirm_booking`, `process_payment`, `manage_reservation`

**During discovery:** The registry returns which agent to use and connection metadata; the **card** at the peer’s URL is what the A2A client uses to build a proper JSON-RPC client (`ClientFactory` / transport). The registry response also supplies **skill names** used when caching (`Skills: 5` on `RemoteAgentClient`), so the orchestrator can match intent before invocation.

---

## Brief takeaway

- **Discovery** is registry HTTP + semantic query; **invocation** is A2A JSON-RPC after **well-known** card fetch.
- **Logs** line up across processes: registry POST → cache + card GET → `A2A REQUEST` → booking `POST /` 200.
- **Gap:** To audit full request/response pairs for peer calls, rely on client debug logging or HTTP capture; default INFO logs on these servers show requests and access lines, not always full JSON-RPC responses.
