# Support Ticket Router: Your First LLM Inside a Graph

## Requirements
TaskNest classifies each support ticket into exactly one category based on its main issue:
- billing: payments, invoices, refunds, subscriptions.
- technical: bugs, crashes, error messages, login problems.
- general: feature requests, general questions, thank-you messages, everything else.

Meaning takes precedence over isolated words. "The app crashes every time I try to pay" is technical because the main issue is a crash.

## Model and implementation
Use Python, LangGraph StateGraph, LangChain ChatOpenAI, and a Pydantic BaseModel with category: Literal["billing", "technical", "general"]. Use ChatOpenAI(model="gpt-6-luna", temperature=0, reasoning_effort="none").with_structured_output(...). GPT-6 Luna is low-cost and appropriate for simple classification; reasoning_effort="none" allows temperature=0. Set max_retries=0 to prevent extra API attempts, and method="json_schema", strict=True for native structured output without tools. Errors propagate without repair calls.

References: [model documentation](https://developers.openai.com/api/docs/models/gpt-6-luna) and [parameter compatibility](https://developers.openai.com/api/docs/guides/deployment-checklist).

## State and graph
The state has exactly four fields: ticket, category, team, reply. A total=False TypedDict permits invocation with only ticket. Nodes return partial updates, preserving existing state.

Exactly four nodes are registered:
1. classify: makes exactly one model invocation and returns only {"category": prediction.category}.
2. billing_team: returns team="Billing Team" and reply="Our Billing Team has received your ticket."
3. tech_team: returns team="Technical Support Team" and reply="Our Technical Support Team has received your ticket."
4. general_team: returns team="General Support Team" and reply="Our General Support Team has received your ticket."

START -> classify -> one selected team -> END.
Exactly one add_conditional_edges call follows classify. The router only returns state["category"]. Map billing to billing_team, technical to tech_team, and general to general_team. Each team connects directly to END and uses only Python. No extra nodes, graph loops, memory, tools, or additional LLM calls.

The classification node interprets language and may make a semantic error. The conditional edge only maps an already validated label to a node; its behavior is deterministic and costs no model request.

## Colab notebook
Install with !pip -q install langgraph langchain-openai. Configure OPENAI_API_KEY in Colab Secrets and allow notebook access. Load it using import os, from google.colab import userdata, and os.environ["OPENAI_API_KEY"] = userdata.get("OPENAI_API_KEY"). Never embed or print credentials or use getpass. The API project needs model access and credits.

Display the compiled graph using draw_mermaid_png(); if the external rendering service fails, print Mermaid source and continue. The service receives graph topology, not tickets or secrets. Support result = graph.invoke({"ticket": "I was charged twice this month"}) and print all four fields.

## Evaluation and reflection
Include 10 fixed-label tickets: three billing, four technical, three general, including crash-during-payment and other ambiguous cases. Each test prints ticket, expected category, actual category, team, reply, and match status. Report real match counts and branch coverage. Keep mismatches as findings and never change labels to manufacture success. Offline checks exercise all three deterministic routing functions; live graph coverage is measured separately and any missing branch is reported honestly.

A fresh top-to-bottom run makes 11 API requests: one example and ten tests. No paid API calls are made during artifact generation. Store no fabricated outputs. End with a four-sentence reflection explicitly marked DRAFT/TEMPLATE for completion after execution.
