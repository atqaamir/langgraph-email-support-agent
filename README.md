# Email Support Agent with LangGraph

A LangGraph workflow that triages an incoming customer email, drafts a reply with an LLM, and pauses for human approval when the message looks urgent. It is a compact demo of LangGraph state, nodes, conditional routing, checkpointed memory, and human-in-the-loop interrupts.


## How it works

```
START → read_email → classify_intent ─┬→ search_documentation ─┐
                                      └→ bug_tracking ─────────┴→ write_response ─┬→ send_reply → END
                                                                                  └→ human_review ─┬→ send_reply → END
                                                                                                   └→ END (rejected)
```

| Node | What it does |
| --- | --- |
| `read_email` | Placeholder for parsing the raw email. |
| `classify_intent` | Asks the LLM for a structured `EmailClassification`: intent (`question`, `bug`, `billing`, `feature`, `other`), urgency, and topic. |
| `search_documentation` | Stubbed knowledge-base lookup. Runs in parallel with `bug_tracking`. |
| `bug_tracking` | Creates a fake ticket ID. Runs in parallel with `search_documentation`. |
| `write_response` | Drafts a reply from the email, classification, and search results, then routes with a `Command`. High or critical urgency goes to `human_review`. Everything else goes straight to `send_reply`. |
| `human_review` | Calls `interrupt()` to pause the graph. A reviewer can approve, edit, or reject the draft. |
| `send_reply` | Stubbed send step that prints the reply. |

The graph is compiled with an `InMemorySaver` checkpointer, so a paused run can be resumed later using its `thread_id`.

The notebook runs two demos:

1. One urgent billing email. The graph pauses at `human_review`, then resumes with `Command(resume={"approved": True})`.
2. A batch of five emails. Those that need approval are collected and then reviewed one by one through `input()` prompts.

Search, ticketing, and sending are placeholders. Swap in real integrations where the comments say so.

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

- Cell 3 imports `doublecheck_env` from `env_utils.py`, a small helper in the course repo that prints masked API-key values. It is only a sanity check. Copy `env_utils.py` from the course repo next to the notebook, or delete that cell.
- The graph diagram cell calls `draw_mermaid_png()`, which needs internet access.
- Keep `.env` out of version control.
