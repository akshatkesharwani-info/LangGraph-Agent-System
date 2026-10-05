# LangGraph Agent System

A business analyst agent built with LangGraph. It plans a task, calls tools to get real numbers, drafts an answer, and passes it through a reviewer that can send it back for one more try. It remembers earlier questions within a conversation.

Built in Google Colab with Groq (`openai/gpt-oss-120b`) through LangChain's OpenAI-compatible client.

## How it works

```
planner -> agent <-> tools
              |
           reviewer -> APPROVE -> end
              |
           REVISE (max 2 review passes) -> feedback -> agent
```

- **Planner:** writes a short plan (at most 4 steps) and is told to do only what was asked.
- **Agent:** calls tools and never guesses numbers. If a tool cannot give what it needs, it must say so.
- **Tools:**
  - `get_sales(region, month, product)`: units and revenue in rupees
  - `top_products(n)`: top products by revenue
  - `check_inventory(product)`: stock, several products at once
  - `calculate(expression)`: arithmetic with a restricted character set
- **Reviewer:** sees the question, the answer **and the raw tool results**. It checks that every part is answered, that every number comes from the tool results or a correct calculation of them, and that the calculation matches the question (right product, region, period).
- **Memory:** `MemorySaver` with a thread id, so "And what about the South region in the same month?" works inside a conversation.
- **Safety limits:** a recursion limit of 60 steps, a maximum of 2 review passes, and the notebook catches the step-limit error instead of crashing.

## Results from the run

| Task | Result |
|---|---|
| Total North revenue in March | ₹6,711,000, matches the data |
| Top 3 products, and the #1 product's share of total revenue | Laptop, Tablet, Phone; Laptop is 40.0% (₹55.55M of ₹138.88M), matches the data |
| Lower stock of Monitor and Tablet, then months of Tablet cover in the West | Monitor is lower (18 vs 45); cover is 0.97 months (45 units / 46.33 average monthly sales), matches the data |
| Follow-up "South region in the same month", same thread | ₹3,004,000 for March (it remembered the month) |
| Same follow-up in a brand-new thread | Asked which month was meant, because it had no memory |

**Checks: 8 of 8 passed.** The checks verify both which tools were used and that the final numbers match values calculated directly from the data. A nicely written answer with a wrong number would fail.

## What I learned building it

- **The first version gave a wrong answer and the reviewer approved it.** For "how many months of Tablet stock", the tool had no product filter, so the agent divided 45 tablets by the sales of all products and answered 0.22 months. The correct answer is 0.97. Two fixes solved it: a product filter on the sales tool, and showing the reviewer the tool results instead of only the final answer.
- **Checking tool use is not enough.** The first check list passed 5 of 5 while the answer was wrong. Comparing final numbers with the data is what catches it.
- **Loops need limits.** The reviewer and the agent can bounce back and forth, so the review passes and the total steps are capped.

## Limitations

- **Four test tasks only.** This shows the design works, it is not a benchmark.
- **The agent is not always efficient.** For the North March revenue question it made 7 tool calls (one per product) where a single `get_sales` call would do. The answer was right, but the plan was wasteful.
- **The reviewer is the same model as the agent**, so it can share the agent's blind spots.
- **`calculate` uses a restricted `eval`.** That is fine for a demo, but a real service should use a proper expression parser.
- The sales data is made up.

## Tech stack

LangGraph, LangChain (`langchain-openai`, `langchain-core`), Groq API, pandas.

## How to run

1. Open the notebook in Google Colab.
2. Run the cells from top to bottom.
3. When asked, paste your own Groq API key (free at console.groq.com). It is hidden while typing and is never saved in the notebook.

It makes about 30 API calls.

## Files the notebook creates

- `langgraph_agent_runs.csv`: the tools used and the answer for each task

---

Built by **Akshat Kesharwani** | [GitHub](https://github.com/akshatkesharwani-info) | [Portfolio](https://akshatkesharwani-info.github.io/)
