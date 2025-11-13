Here’s an implementation plan for **Direction 1 – Agentic AI Track** that you can hand directly to a coding agent working in your fork of:

`https://github.com/hafnium49/alex`

The plan assumes the **working directory is the repo root**, and that you will **store this plan as a markdown file** in:

`terraform/4_researcher_week_3_assignent/agentic_direction1_plan.md`

---

# Direction 1 – Agentic AI Track: Implementation Plan for Alex Researcher

## 0. Scope & Objectives

**Goal:** Extend the **Alex Researcher** service to be more “agentic” by:

1. Adding **one or more additional MCP servers** (e.g. a financial-data MCP such as Polygon, or another MCP you choose).
2. Improving **context engineering** for the researcher agent (richer system instructions that explicitly use new tools).
3. Adding **to-do-list tools** so the agent can explicitly plan, track, and execute research steps.

**Key code locations:**

* `backend/researcher/context.py`
* `backend/researcher/mcp_servers.py`
* `backend/researcher/tools.py`
* `backend/researcher/server.py`
* `terraform/4_researcher_week_3_assignent/main.tf`
* `terraform/4_researcher_week_3_assignent/variables.tf` (if present)
* `terraform/4_researcher_week_3_assignent/terraform.tfvars` and `terraform.tfvars.example`
* `.env` at repo root

**Non-goals:**

* No changes to the ingest Lambda, API Gateway, or S3 Vectors semantics.
* No breaking changes to existing **Guide 4** workflow (manual research → ingest → search).
* OSS vs Nova model choice is out of scope; reuse current Bedrock model wiring.

---

## 1. Git & Branch Setup

**Task 1.1 – Create feature branch**

1. From repo root:

   ```bash
   git checkout main
   git pull origin main
   git checkout -b feature/agentic-direction1
   ```

2. All changes in this plan go on this branch.

**Acceptance criteria**

* `git status` shows you on `feature/agentic-direction1`.
* No uncommitted changes before starting implementation.

---

## 2. New MCP Server(s) for Financial Data

The researcher currently only uses a **Playwright MCP** for generic web browsing. Extend it to also have a **financial-data MCP** (e.g. Polygon or equivalent) so that:

* The agent can fetch **structured market data** (prices, fundamentals, etc.).
* The MCP server runs as an **MCPServerStdio** child process inside the container.

### 2.1 Environment variables & Terraform wiring

**Task 2.1.1 – Add new secrets to `.env`**

In repo root `.env`, add placeholders for the new MCP server credentials (example for Polygon; rename if you use another MCP):

```dotenv
POLYGON_API_KEY=your-polygon-api-key-here
POLYGON_API_BASE_URL=https://api.polygon.io  # or as required by the MCP server
```

(If using a different MCP, adapt variable names accordingly.)

**Task 2.1.2 – Add terraform variables**

If not already present, create or update `terraform/4_researcher_week_3_assignent/variables.tf` to include variables for these secrets, following the existing pattern used for `openai_api_key` / `alex_api_key`. For example:

```hcl
variable "polygon_api_key" {
  description = "API key for Polygon MCP (financial data)"
  type        = string
  default     = ""
}

variable "polygon_api_base_url" {
  description = "Base URL for Polygon API (optional)"
  type        = string
  default     = ""
}
```

**Task 2.1.3 – Update `terraform.tfvars.example` and local `terraform.tfvars`**

In `terraform/4_researcher_week_3_assignent/terraform.tfvars.example`, add:

```hcl
polygon_api_key      = "your-polygon-api-key-here"
polygon_api_base_url = "https://api.polygon.io"
```

In your actual `terraform/4_researcher_week_3_assignent/terraform.tfvars`, set real values (or leave blank if you’ll inject via env later).

**Task 2.1.4 – Pass env vars into App Runner**

Update `terraform/4_researcher_week_3_assignent/main.tf`, in the `aws_apprunner_service.researcher` resource, `runtime_environment_variables` block:

```hcl
runtime_environment_variables = {
  OPENAI_API_KEY    = var.openai_api_key
  ALEX_API_ENDPOINT = var.alex_api_endpoint
  ALEX_API_KEY      = var.alex_api_key

  # NEW: MCP financial data server configuration
  POLYGON_API_KEY      = var.polygon_api_key
  POLYGON_API_BASE_URL = var.polygon_api_base_url
}
```

**Acceptance criteria**

* `terraform validate` from `terraform/4_researcher_week_3_assignent` passes.
* After `terraform apply`, the App Runner service has `POLYGON_API_KEY` (and base URL) visible in `/health` debug output if you choose to log env there.

---

### 2.2 New MCP server factory in `mcp_servers.py`

**Task 2.2.1 – Add factory for financial-data MCP**

In `backend/researcher/mcp_servers.py`, add a new function similar to `create_playwright_mcp_server`:

* Name suggestion: `create_financial_data_mcp_server(timeout_seconds=60)`.
* Use `MCPServerStdio` with `command="npx"` and appropriate `args` for your financial MCP package (e.g. Polygon). Do **not** hardcode real keys; rely on env vars.

**Guidance for the coding agent (no hardcoding):**

* Look up the **official MCP server documentation** for your chosen financial MCP (e.g. Polygon MCP from the Agentic AI course).
* Use the recommended CLI invocation, e.g. something like:

  ```python
  args = ["@polygon-xyz/mcp@latest", "--some-flag", ...]
  ```
* Pass through environment variables (`POLYGON_API_KEY`, etc.) if `MCPServerStdio` supports an `env` parameter; otherwise rely on process env.

**Pseudo-code shape (coding agent should fill real command):**

```python
from agents.mcp import MCPServerStdio
import os

def create_financial_data_mcp_server(timeout_seconds: int = 60) -> MCPServerStdio:
    api_key = os.getenv("POLYGON_API_KEY", "")
    base_url = os.getenv("POLYGON_API_BASE_URL", "")

    params = {
        "command": "npx",
        "args": [
            "### FILL-IN: MCP PACKAGE NAME HERE ###",
            "--headless",
            # other flags as required by the MCP server docs
        ],
        # Optionally:
        # "env": {
        #     "POLYGON_API_KEY": api_key,
        #     "POLYGON_API_BASE_URL": base_url,
        # },
    }

    return MCPServerStdio(params=params, client_session_timeout_seconds=timeout_seconds)
```

**Acceptance criteria**

* `uv run pytest` (if any tests exist) still passes or at least imports `create_financial_data_mcp_server` without errors.
* No changes to existing `create_playwright_mcp_server` behavior.

---

### 2.3 Wire new MCP server into the agent in `server.py`

**Task 2.3.1 – Use both Playwright and financial MCP**

In `backend/researcher/server.py`, inside `run_research_agent`:

1. Import the new factory:

   ```python
   from mcp_servers import create_playwright_mcp_server, create_financial_data_mcp_server
   ```

2. In the `with trace("Researcher")` block, change the single MCP server context to include both:

   **Before (simplified):**

   ```python
   with trace("Researcher"):
       async with create_playwright_mcp_server(timeout_seconds=60) as playwright_mcp:
           agent = Agent(
               ...,
               mcp_servers=[playwright_mcp],
           )
           result = await Runner.run(agent, input=query, max_turns=15)
   ```

   **After (conceptual – coding agent must ensure valid async syntax):**

   ```python
   with trace("Researcher"):
       async with create_playwright_mcp_server(timeout_seconds=60) as playwright_mcp, \
                  create_financial_data_mcp_server(timeout_seconds=60) as financial_mcp:
           agent = Agent(
               name="Alex Investment Researcher",
               instructions=get_agent_instructions(),
               model=model,
               tools=[ingest_financial_document,  # to be extended later
                      # TODO tools.todo_* will be added in Section 4
               ],
               mcp_servers=[playwright_mcp, financial_mcp],
           )
           result = await Runner.run(agent, input=query, max_turns=15)
   ```

**Acceptance criteria**

* Service starts without runtime errors in App Runner.
* OpenAI Traces show both MCP servers in the tool list (`List MCP Tools` should include the new MCP server).

---

## 3. Context Engineering in `context.py`

Goal: Make the agent’s **system instructions** explicitly:

* Use financial MCP for **structured data**.
* Use Playwright MCP for **page-level detail**.
* Use a **to-do plan** (Section 4).
* Stay within strict resource limits (pages, steps, time).

**Task 3.1 – Redesign `get_agent_instructions`**

Update `backend/researcher/context.py`:

1. Keep the function signature, but restructure the returned prompt into clear sections:

   * Role and date.
   * High-level objective.
   * Tool usage policy (financial MCP vs Playwright).
   * To-do list workflow.
   * Constraints (pages, turns, latency).

2. Explicitly describe the **three/four-step flow**. Example structure (coding agent should implement as text, not code):

* Step 0: Plan with to-do list tools (will be implemented in Section 4).
* Step 1: Fetch structured market data via financial MCP (prices, fundamentals).
* Step 2: Use Playwright MCP for 1–2 pages of contextual news if needed.
* Step 3: Write a short investment note and call `ingest_financial_document`.

3. Keep or refine the emphasis on **brevity** and **max 2 pages** from the original prompt.

4. Update the `Topic: "[Asset] Analysis {…}"` part if necessary to reflect the more structured workflow (optional, but keep semantics similar).

**Task 3.2 – Update `DEFAULT_RESEARCH_PROMPT`**

Optionally refine `DEFAULT_RESEARCH_PROMPT` to:

* Encourage selection of a **topic that benefits from structured market data** (e.g. large cap stock or macro theme with clear instruments).
* Mention that the agent should **use to-do list tools** to plan steps (once they exist).

**Acceptance criteria**

* Instructions clearly describe:

  * Use of financial MCP for numeric/fundamental data.
  * Use of Playwright MCP for limited web browsing.
  * Use of to-do list tools to plan and track steps.
* The prompt remains concise enough to fit within typical context limits.

---

## 4. To-Do-List Tools in `tools.py` and Agent Wiring

Goal: Provide **explicit planning tools** that the agent can call, so its behavior is more structured and inspectable in traces.

### 4.1 Design requirements

* Tools should be **idempotent and side-effect-light**. It’s acceptable for this assignment if they:

  * Keep the to-do list in memory for the duration of a single run **only**, and/or
  * Return the updated plan as a text string that the agent can keep in its context.
* Tools should **log actions** so you can see them in CloudWatch logs and in agent traces.
* No persistent storage is required (but harmless if you add it).

### 4.2 Implement tools in `backend/researcher/tools.py`

**Task 4.2.1 – Add tool functions**

Extend `backend/researcher/tools.py` with three new tools (names can be adjusted but should be descriptive):

1. `create_research_todo_list(goal: str) -> str`

   * Purpose: initialize a to-do list for the current research run.
   * Behavior:

     * Log `goal` and initialize an in-memory list (even if just in a module-level variable).
     * Return a brief textual plan skeleton, e.g.:

       * `"TODO List: 1) Fetch market data, 2) Check latest news, 3) Write 3-bullet summary"`

2. `add_todo_item(description: str, priority: str = "medium") -> str`

   * Purpose: append an item to the current to-do list.
   * Behavior:

     * Log the new item.
     * Return the updated list in a concise text format.

3. `complete_todo_item(index: int) -> str`

   * Purpose: mark a to-do item as done.
   * Behavior:

     * Log which item was completed.
     * Return an updated list with status flags (e.g. `[x]` vs `[ ]`).

**Implementation hints for the coding agent:**

* Use a simple module-level data structure such as:

  ```python
  _CURRENT_TODO: list[dict[str, Any]] = []
  ```
* Since App Runner instances are short-lived and stateless across runs, it’s fine if the state is **per request** only.
* Make sure each tool has a clear docstring; they will appear in traces.

**Task 4.2.2 – Export tools for import**

Ensure `ingest_financial_document` remains intact, and add these functions to the module’s `__all__` (if one exists) or at least make them importable.

### 4.3 Wire tools into Agent in `server.py`

**Task 4.3.1 – Import new tools**

In `backend/researcher/server.py`, update imports:

```python
from tools import (
    ingest_financial_document,
    create_research_todo_list,
    add_todo_item,
    complete_todo_item,
)
```

**Task 4.3.2 – Add tools to the Agent**

Inside `run_research_agent`, in the `Agent(...)` construction, update the `tools` list:

```python
agent = Agent(
    name="Alex Investment Researcher",
    instructions=get_agent_instructions(),
    model=model,
    tools=[
        create_research_todo_list,
        add_todo_item,
        complete_todo_item,
        ingest_financial_document,
    ],
    mcp_servers=[playwright_mcp, financial_mcp],
)
```

**Task 4.3.3 – Ensure instructions mention tool names**

Verify that the updated `get_agent_instructions`:

* Explicitly instructs the agent to:

  * Call `create_research_todo_list` at the beginning with its overall goal.
  * Use `add_todo_item` to refine the plan if needed.
  * Call `complete_todo_item` as it finishes logical steps.
* Uses the **exact function names** so the agent can discover them via tool listing.

**Acceptance criteria**

* OpenAI Traces show tool calls in order:

  * `create_research_todo_list` → one or more `add_todo_item` → `browser_*` & financial MCP calls → `complete_todo_item` → `ingest_financial_document`.
* Research output still gets stored in S3 Vectors via ingest Lambda.

---

## 5. Tests & Validation

### 5.1 Local smoke tests (backend)

**Task 5.1.1 – Run researcher locally (optional)**

From `backend/researcher`:

```bash
uv run test_research.py "NVIDIA AI chip market share"
uv run test_research.py "Apple services revenue growth"
```

Check that:

* Requests complete (even if sometimes slow).
* To-do tools appear in terminal logs (if you log there).

### 5.2 Deployed tests (App Runner)

**Task 5.2.1 – Redeploy researcher**

From repo root:

1. Rebuild and push Docker image:

   ```bash
   cd backend/researcher
   uv run deploy.py
   ```

2. Re-run Terraform (if needed):

   ```bash
   cd ../../terraform/4_researcher_week_3_assignent
   terraform apply
   ```

**Task 5.2.2 – Health check**

Once deployment is finished, from any terminal:

```bash
curl https://YOUR_SERVICE_URL/health
```

Expect JSON with `"status": "healthy"`.

### 5.3 End-to-end pipeline test

**Task 5.3.1 – Research → Ingest → Search**

1. Clean S3 Vectors:

   ```bash
   cd backend/ingest
   uv run cleanup_s3vectors.py
   ```

2. Generate research:

   ```bash
   cd ../researcher
   uv run test_research.py "Tesla competitive advantages"
   ```

3. Search:

   ```bash
   cd ../ingest
   uv run test_search_s3vectors.py "electric vehicles"
   ```

Verify:

* New research content is stored.
* Semantic search finds it.

### 5.4 Agentic behavior verification in OpenAI Traces

**Task 5.4.1 – Inspect traces**

Use the OpenAI Platform Logs → Traces UI:

* Confirm that:

  * To-do tools are called early and mid-run.
  * The financial MCP server is being invoked (e.g. `financial_data_*` tool calls).
  * Playwright MCP calls remain but are limited in number.

**Acceptance criteria (overall)**

* Existing **Guide 4** flows still work end-to-end.
* New behavior:

  * Researcher uses **two MCP servers** (web + financial).
  * Agent uses **to-do list tools** in traces.
  * Context instructions align with this behavior.

---

## 6. Documentation & Cleanup

**Task 6.1 – Add implementation notes**

Create a markdown file:

* Path: `terraform/4_researcher_week_3_assignent/agentic_direction1_plan.md`
* Content: this plan (or the final implemented description) plus:

  * Short summary of what you actually implemented.
  * Any deviations from this plan.

**Task 6.2 – Optional updates to guides**

If you want, add a short subsection to `guides/4_researcher.md` describing:

* The new MCP server.
* The to-do tools.
* How to see them in traces.

---

## 7. Ready-for-PR Checklist

Before opening a PR to the course repo:

* [ ] All code changes committed on `feature/agentic-direction1`.
* [ ] Terraform validates and applies successfully.
* [ ] App Runner service is `RUNNING`.
* [ ] End-to-end test (`test_research.py` → `test_search_s3vectors.py`) works.
* [ ] OpenAI Traces show:

  * To-do tools.
  * Two MCP servers.
* [ ] `agentic_direction1_plan.md` created in `terraform/4_researcher_week_3_assignent/`.
* [ ] Optional: notebook under `production/community_contributions` summarizing the enhancement and linking to your fork.

This plan should be directly usable as instructions for a coding agent operating on your `hafnium49/alex` fork.
