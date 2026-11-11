# Agent overload: what this branch adds

## The problem
OpenPoke's interaction agent saw only the **names** of its execution agents. As one user's agents piled up, it lost
track of them:
- It created duplicates instead of reusing the right agent: 0 of 48 cases at 10 agents, 13 of 48 at 300.
- Its pass rate fell from 62% to 40%. (pass rate means if the IA routed correctly to either of reuse, create or clarify)

Each duplicate starts a fresh thread and loses the history of the old one.

## What was added
- **Agent store** (`server/services/agents/store.py`): a SQLite store of agents, each with a description, its
  contacts and search indexes. It replaces `roster.json`; importing an existing `roster.json` is not built yet.
- **Descriptions** (`descriptions.py`): after each run, one cheap LLM call rewrites the agent's description, so it
  shows what the agent actually does.
- **Contacts** (`contacts.py`): records who each agent emails, so people can be matched by email address as well as
  name.
- **Retrieval** (`retrieval.py`, `indexer.py`, `embeddings.py`): each turn, the 20 agents most relevant to the
  message are found and shown to the interaction agent as cards.
- **Interaction agent** (`server/agents/interaction_agent/`):
  - It reuses agents by `agent_id` instead of by name.
  - It gets a `search_agents` tool for agents the retrieval missed.
  - Its prompt has routing rules.
- **Evals** (`evals/`): a synthetic user with up to 300 agents, labelled test cases, a cost-capped runner, and
  reports. Every run is logged in `evals/results/EXPERIMENTS.md` and every paid run in `SPEND.md`.

## Result
Final test run on Claude Haiku 5.5: 33 test cases, 3 runs each, so 99 attempts per cell. Pass rates by number of
agents:

| Mode | 10 | 30 | 100 | 300 |
|---|---|---|---|---|
| `legacy_rules` | 80% | 76% | 74% | 63% |
| `full` | 94% | 93% | 93% | 93% |
| `hybrid` (production) | 96% | 98% | 96% | 97% |

`hybrid` holds steady as the number of agents grows; names-only does not. `hybrid` also costs far less than `full`,
because it shows 20 cards instead of all of them.

A later rerun replaced agent names that appeared in the prompt's examples and reran the 11 test cases that shared
them. With those 11 cases swapped in, the table becomes:
- `legacy_rules`: 75 / 71 / 69 / 66%
- `full`: 95 / 94 / 92 / 92%
- `hybrid`: 92 / 98 / 97 / 95%

`hybrid` and `full` stay close, and both stay well ahead of names-only.

## Terms
- **Interaction agent (IA):** the agent the user chats with. It decides which execution agent handles a request.
- **Execution agent:** a worker agent that owns one ongoing task, such as an email thread with one person.
- **Roster:** the list of execution agents the IA is shown on a turn.
- **Card:** one agent's entry in the roster: its id, name, description and contacts.
- **Modes:**
  - **`legacy`:** names only, which was the original behaviour.
  - **`legacy_rules`:** names plus routing rules.
  - **`full`:** every card.
  - **`hybrid`:** the top 20 retrieved cards plus the `search_agents` tool. This is the production default, set by
    `OPENPOKE_ROSTER_MODE`.
- **N:** the number of agents the synthetic user has: 10, 30, 100 or 300.
- **A3 / message weighting:** retrieval runs on the latest message alone and on the last exchange plus the message.
  Recent history can then pull in the right agent without drowning out the message.

See `README.md` for how to run the tests and the evals.
