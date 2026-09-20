# Email Support Agent with LangGraph

A LangGraph workflow that triages an incoming customer email, lets an LLM agent call tools to look up information, drafts a reply, and pauses for human approval when the message looks urgent. It is a compact demo of LangGraph state, tool calling, conditional routing, checkpointed memory, and human-in-the-loop interrupts.


## How it works

```
START → read_email → classify_intent → prepare_agent → support_agent ─┬→ (tool call) → tools ─┐
                                                            ▲          │                       │
                                                            └──────────┼───────────────────────┘
                                                                       └→ (done) → write_response ─┬→ send_reply → END
                                                                                                   └→ human_review ─┬→ send_reply → END
                                                                                                                    └→ END (rejected)
```

| Node | What it does |
| --- | --- |
| `read_email` | Placeholder for parsing the raw email. |
| `classify_intent` | Asks the LLM for a structured `EmailClassification`: intent (`question`, `bug`, `billing`, `feature`, `other`), urgency (`low`, `medium`, `high`, `critical`), and topic. |
| `prepare_agent` | Wraps the email text in a `HumanMessage` and adds it to the `messages` history. |
| `support_agent` | The tool-calling agent. A system prompt carries the classification as context, and the LLM either calls a tool or answers directly. |
| `tools` | A `ToolNode` that runs the requested tool and returns to `support_agent`, so the agent can chain several calls before it answers. |
| `write_response` | Takes the agent's final message as the draft, then routes with a `Command`. Urgency `high` or `critical`, or intent `other`, goes to `human_review`. Everything else goes straight to `send_reply`. |
| `human_review` | Calls `interrupt()` to pause the graph. A reviewer can approve, edit, or reject the draft. |
| `send_reply` | Stubbed send step that prints the reply. |

### Tools

The agent can call three mock tools. Replace them with real integrations for production.

| Tool | What it does |
| --- | --- |
| `get_order_status` | Looks up an order in a hard-coded table (`ORD-123`, `ORD-456`). |
| `search_knowledge_base` | Keyword search over a small hard-coded knowledge base (`subscription`, `refund`, `delivery`). |
| `create_support_ticket` | Returns a generated `TICKET-xxxxxxxx` ID for a given issue and priority. |

### State

`EmailAgentState` holds the email (`email_content`, `sender_email`, `email_id`), the `classification`, the `draft_response`, and a `messages` list that uses the `add_messages` reducer to accumulate the agent conversation and tool results.

The graph is compiled with an `InMemorySaver` checkpointer, so a paused run can be resumed later using its `thread_id`.

## Demos

The notebook runs two demos:

1. One urgent billing email. The graph pauses at `human_review`, then resumes with `Command(resume={"approved": True})`.
2. A batch of five emails. Those that need approval are collected and then reviewed one by one through `input()` prompts. A rejected draft is added to a manual follow-up list.

## Setup

**Requirements:** Python 3.11–3.13 and an OpenAI API key. The notebook uses `gpt-5-mini`.

```bash
pip install "langgraph>=1.0" "langchain>=1.0" "langchain-openai>=1.0" python-dotenv ipykernel jupyter
```

Create a `.env` file next to the notebook:

```
OPENAI_API_KEY=sk-...
```

Then open the notebook and run the cells in order:

```bash
jupyter notebook email_support_agent.ipynb
```

### Notes

- Run the cells top to bottom. Later cells depend on imports and definitions from earlier ones, so a kernel restart means starting again from the top.
- The graph diagram cell calls `draw_mermaid_png()`, which needs internet access.
- The last demo cell uses `input()`, so it needs an interactive notebook and will not work in a headless run.
- Keep `.env` out of version control.
