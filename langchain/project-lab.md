# 🦜 LangChain Project Mastery

#### 📋 Project: SkillBot — AI HR Assistant at Zestora Technologies

**Zestora Technologies** is a growing Hyderabad-based software company with 800+ employees. Their HR team is drowning in repetitive questions: "How many leaves do I have?", "What's the appraisal policy?", "How do I apply for reimbursement?" — every single day, hundreds of such queries. The HR team wastes 4 hours daily answering the same questions from the company's internal policy PDF.

Your task: Build **SkillBot** — a LangChain-powered AI assistant that reads the HR policy documents and answers employee questions instantly, with memory of the conversation, the ability to search the web for current laws (like PF rates), and escalates unresolved queries by email.

> **🎯 What You'll Build**

> - LLM-powered Q&A over HR PDFs
> - Conversational memory across turns
> - Agent with web search + email tools
> - RAG pipeline with vector store
> - Custom prompt templates
> - LangChain + LangChain-Community
> - OpenAI GPT-4o (LLM)
> - FAISS (vector store)
> - Python 3.11+
> - Streamlit (UI)
> - Freshers learning AI/LLM development
> - Python developers exploring GenAI
> - Engineers moving into AI roles
> - Product teams building AI features

**Scene 0 — Day 1 at Zestora | "The Problem"**

> **Meghna** _Head of HR — Zestora Technologies_
> 
> "Priya, I've answered 'how many sick leaves do we get' seventeen times this week. Seventeen. From seventeen different people who could have read page 4 of the policy PDF. I need you to build something that handles this."

> **Priya** _Senior AI Engineer — Zestora Technologies_
> 
> "I'm going to build SkillBot — a LangChain agent that reads our HR PDF, answers employee questions, remembers what was said earlier in the conversation, and escalates complicated queries to you by email. The whole thing runs in Python. Give me this week."

### 1. Phase 1 — Understanding LangChain: The Building Blocks

**What is LangChain?** LangChain is a Python framework that makes it easy to build applications powered by Large Language Models (LLMs) like GPT-4o or Claude. Instead of writing raw API calls to OpenAI, LangChain gives you ready-made components — chains, agents, memory, tools, document loaders — that you connect together like LEGO blocks.

> **🌱 Fresher's Mental Model — The Restaurant Analogy**

> - **LLM** = the chef (GPT-4, Claude, Gemini). The actual intelligence. Does the thinking.
> - **Prompt Template** = the recipe. Tells the chef exactly what dish to make and how.
> - **Chain** = the kitchen workflow. Step 1: prep. Step 2: cook. Step 3: plate. Steps run in sequence.
> - **Memory** = the waiter's notepad. Remembers what you ordered earlier so you don't repeat yourself.
> - **Tool** = kitchen equipment (web search = the internet, calculator = the oven). The LLM calls tools when it needs them.
> - **Agent** = the head chef who decides WHICH tools to use and in what order, based on your request.
> - **Vector Store / RAG** = the cookbook shelf. Stores your documents so the LLM can "look up" answers from them.

#### 1.1 LangChain Architecture — How It All Fits

👤 User Input

"How many casual leaves do I get per year?"

▼

passes through

📝 Prompt Template

System: You are an HR assistant for Zestora. Context: {retrieved_docs} History: {chat_history} Question: {question}

▼

sent to

🧠 LLM (GPT-4o)

Reads prompt + context → generates answer

▼

optionally uses

🛠️ Tools / Agent

Web Search | Calculator | Email sender Runs tools if needed, loops back to LLM

▼

returns

💬 Final Response

"You get 12 casual leaves per year at Zestora, as per Section 3 of the HR policy."

#### 1.2 Install LangChain

```bash
# Create a virtual environment (always do this first)
python -m venv skillbot-env
source skillbot-env/bin/activate     # Mac/Linux
skillbot-env\Scripts\activate        # Windows
# Install LangChain and dependencies
pip install langchain langchain-openai langchain-community
pip install faiss-cpu pypdf streamlit python-dotenv
```

> **langchain** — the core framework: chains, agents, memory, prompt templates.
**langchain-openai** — connects LangChain to OpenAI models (GPT-4o, embeddings).
**langchain-community** — community integrations: document loaders, web search, FAISS.
**faiss-cpu** — Facebook's vector similarity search library. Used to store and search document embeddings. "cpu" means it runs without a GPU, fine for our project.
**pypdf** — reads PDF files. We'll use this to load the HR policy PDF.
**python-dotenv** — loads your API keys from a `.env` file so you never hardcode secrets.

#### 1.3 Your First LangChain Call

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
import os
# Load API key from .env file
os.environ["OPENAI_API_KEY"] = "sk-..."
# Create the LLM object
llm = ChatOpenAI(
    model="gpt-4o",
    temperature=0.3
)

# Send a message and get a response
response = llm.invoke([
    HumanMessage(content="What is LangChain?")
])

print(response.content)
```

**📖 Understanding This Code**

- **ChatOpenAI** — LangChain's wrapper around OpenAI's chat models. Handles all the API complexity.
- **model="gpt-4o"** — which OpenAI model to use. gpt-4o is fast and capable, good for production apps.
- **temperature=0.3** — controls creativity. 0 = deterministic/factual, 1 = creative/varied. For HR answers, use low temperature so the bot doesn't hallucinate.
- **HumanMessage** — wraps the user's text. LangChain uses typed messages (HumanMessage, AIMessage, SystemMessage) to structure conversations.
- **.invoke()** — sends the message to the LLM and returns a response object. `response.content` has the text.

```
LangChain is a framework for building applications powered by large language models.
It provides components like chains, agents, memory, and document loaders that
allow developers to connect LLMs with external data and tools.
```

### 2. Phase 2 — Prompt Templates: Talking to the LLM Correctly

**Business Problem:** Every time an employee asks SkillBot a question, we want to include: (a) instructions about who SkillBot is, (b) relevant HR policy text, (c) the employee's question. Hardcoding this every time is messy. **Prompt Templates** solve this — they're reusable message blueprints with variable slots you fill in at runtime.

**Scene 2 — SkillBot Design Meeting | "The Right Prompt Makes or Breaks the Bot"**

> **Priya** _Senior AI Engineer — Zestora Technologies_
> 
> "Rahul, the prompt template is like a letter format. You design it once with blanks — {employee_name}, {question}, {policy_context} — and at runtime you fill in the blanks. The LLM gets a perfectly structured message every time. The quality of your prompt directly determines the quality of the LLM's answer. This is the most important skill in AI engineering."

> **Rahul** _Junior AI Engineer (Fresher) — Zestora Technologies_
> 
> "So the prompt template is not just the question — it's the entire instruction set? Including who the bot is, what tone to use, what to do when it doesn't know?"

> **Priya** _Senior AI Engineer — Zestora Technologies_
> 
> "Exactly. Prompt engineering is 60% of LangChain work. Write it well and the LLM behaves perfectly. Write it carelessly and you'll spend weeks debugging hallucinations."

#### 2.1 Create a Prompt Template

```python
from langchain_core.prompts import ChatPromptTemplate
template = ChatPromptTemplate.from_messages([
  ("system", """You are SkillBot, the official HR
assistant at Zestora Technologies.
Answer only based on the HR policy below.
If you don't know, say 'Please contact HR.'
Do NOT make up policies.

HR Policy:
{policy_context}"""),
  ("human", "{question}")
])

# Fill in the template with actual values
filled_prompt = template.invoke({
  "policy_context": "Employees get 12 casual leaves per year.",
  "question": "How many casual leaves do I have?"
})
```

**📖 ChatPromptTemplate Explained**

- **from_messages()** — builds a multi-turn prompt with explicit roles: "system" (instructions to the AI), "human" (the user's message), "ai" (AI's previous reply).
- **"system" message** — the persona and rules for the LLM. This is where you say who it is, what it can/can't do, and how to behave. The LLM follows these very carefully.
- **{policy_context}** — a variable slot. At runtime, LangChain replaces this with the actual policy text retrieved from the HR PDF.
- **"Do NOT make up policies"** — a grounding instruction. Critical for HR bots — you never want the LLM to invent a policy that doesn't exist.
- **.invoke({})** — substitutes all the variables and returns the final prompt ready to send to the LLM.

#### 2.2 Chain the Prompt with the LLM (LCEL)

```python
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
llm = ChatOpenAI(model="gpt-4o", temperature=0.2)

# The pipe operator | chains components together
chain = template | llm | StrOutputParser()

# Run the full chain in one call
answer = chain.invoke({
  "policy_context": "12 casual leaves per year.",
  "question": "How many casual leaves do I get?"
})
print(answer)
```

**📖 LCEL — LangChain Expression Language**

- **The pipe operator |** — chains components together. Output of the left becomes input of the right. This is called LCEL (LangChain Expression Language). It's the modern LangChain way — clean and composable.
- **template | llm** — first format the prompt, then send it to the LLM. The LLM returns a message object.
- **| StrOutputParser()** — extracts just the text string from the LLM's response object. Without this, you'd get a complex `AIMessage` object and need to do `.content` yourself.
- **chain.invoke({})** — runs the entire pipeline: format prompt → send to LLM → parse to string. One call, everything handled.
- Think of LCEL like Unix pipes: `cat file | grep error | wc -l` — each step processes and passes along.

```
You are entitled to 12 casual leaves per year at Zestora Technologies,
as per the HR policy. These can be taken in full or half-day increments.
For any clarifications, please contact the HR team.
```

### 3. Phase 3 — RAG: Making the LLM Read Your PDF

**Business Problem:** The LLM (GPT-4o) was trained on internet data, not Zestora's internal HR policy PDF. We need to load the PDF, break it into chunks, convert them into numbers the LLM can search (embeddings), store them in a vector database (FAISS), and retrieve the relevant chunks when an employee asks a question. This is called **RAG — Retrieval-Augmented Generation**.

#### 3.1 How RAG Works — The Full Pipeline

📄

PDF

HR Policy doc

→

✂️

Chunker

Split into pieces

→

🔢

Embeddings

Convert to vectors

→

🗄️

FAISS

Vector DB

→

🔍

Retriever

Find top-k chunks

→

🧠

LLM

Generate answer

↑ Scroll horizontally on mobile to see the full pipeline

#### 3.2 Load and Split the PDF

```python
from langchain_community.document_loaders import PyPDFLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
# Step 1: Load the PDF
loader = PyPDFLoader("zestora_hr_policy.pdf")
pages = loader.load()
print(f"Loaded {len(pages)} pages")

# Step 2: Split into smaller chunks
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50
)
chunks = splitter.split_documents(pages)
print(f"Created {len(chunks)} chunks")
```

**📖 Loading & Splitting Documents**

- **PyPDFLoader** — reads a PDF file and converts each page into a LangChain `Document` object. A Document has `.page_content` (the text) and `.metadata` (page number, file source).
- **Why split?** — LLMs have a context limit (e.g., 128K tokens for GPT-4o). A 50-page HR PDF doesn't fit in one prompt. We split it into small pieces and only send the relevant ones.
- **chunk_size=500** — each chunk is at most 500 characters. Small enough to be precise, big enough to have context.
- **chunk_overlap=50** — consecutive chunks share 50 characters. This ensures a sentence split across chunk boundaries is still captured by at least one chunk.
- **RecursiveCharacterTextSplitter** — tries to split on paragraph breaks first, then sentences, then words — it's smart about not breaking sentences mid-way.

#### 3.3 Create Embeddings and Store in FAISS

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
# Create embedding model (converts text → numbers)
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small"
)

# Store all chunks in FAISS vector database
vectorstore = FAISS.from_documents(
    documents=chunks,
    embedding=embeddings
)

# Save to disk (no need to rebuild every time)
vectorstore.save_local("skillbot_db")
print("Vector store saved!")
```

**📖 Embeddings & Vector Store**

- **Embeddings** — a way to represent text as a list of numbers (a vector). Similar text has similar vectors. "casual leave" and "annual leave" will be numerically close. This is how semantic search works.
- **text-embedding-3-small** — OpenAI's fast, cheap embedding model. Each chunk of text becomes a 1536-dimension vector (a list of 1536 numbers).
- **FAISS** — Facebook AI Similarity Search. An efficient library to store millions of vectors and instantly find the ones most similar to a query vector. Think of it as a search engine for meaning, not keywords.
- **from_documents()** — embeds every chunk and stores all vectors in FAISS in one call. This runs once (takes a few seconds for a 50-page PDF).
- **save_local()** — saves the vector database to disk. Next time the app starts, just load it — no need to re-embed the whole PDF.

#### 3.4 Build the Full RAG Chain

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser
# Load vector store from disk
db = FAISS.load_local("skillbot_db", OpenAIEmbeddings(), allow_dangerous_deserialization=True)
retriever = db.as_retriever(search_kwargs={"k": 3})

# Format retrieved docs into a single string
def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

# Full RAG chain: retrieve → format → prompt → LLM → parse
rag_chain = (
    {"policy_context": retriever | format_docs,
     "question": RunnablePassthrough()}
    | template
    | ChatOpenAI(model="gpt-4o", temperature=0.2)
    | StrOutputParser()
)

answer = rag_chain.invoke("How many sick leaves per year?")
print(answer)
```

**📖 The RAG Chain Explained**

- **as_retriever(k=3)** — converts the vector store into a retriever. When you pass a question, it searches FAISS and returns the 3 most relevant chunks. "k" controls how many chunks to fetch.
- **format_docs()** — joins the 3 retrieved chunks into a single string with blank lines between them. This becomes the {policy_context} in the prompt.
- **RunnablePassthrough()** — a passthrough that forwards the original question unchanged. It's needed here because the chain input (the question) must reach two places: the retriever (to search) and the prompt template (to insert into the human message).
- **The dict at the top** — creates the two variables the template needs: fetches {policy_context} via retrieval, passes {question} through unchanged.
- Result: the LLM receives the question plus only the relevant 3 policy chunks — not the whole 50-page PDF.

```
According to Zestora's HR Policy (Section 4.2), employees are entitled to
7 sick leaves per calendar year. These cannot be carried forward to the next year.
A medical certificate is required for sick leave exceeding 3 consecutive days.
For further queries, please contact hr@zestora.in.
```

### 4. Phase 4 — Memory: SkillBot Remembers the Conversation

**Business Problem:** Without memory, every question the employee asks is treated as brand new. If they ask "how many leaves do I have?" and then follow up with "can I take them in half days?" — the bot has no idea what "them" refers to. Memory fixes this by keeping track of the conversation history.

**Scene 4 — Testing SkillBot | "It Forgot Everything!"**

> **Rahul** _Junior AI Engineer — Zestora Technologies_
> 
> "Priya, I tested the bot. First I asked about casual leaves. Then I asked 'can I split them into half days?' — and it said 'I don't know what you're referring to.' It's like talking to someone with amnesia."

> **Priya** _Senior AI Engineer — Zestora Technologies_
> 
> "That's because we haven't added memory yet. We need to attach a ConversationBufferMemory to the chain. It stores the full chat history and automatically injects it into every prompt — so the bot knows what was said earlier in the same session."

#### 4.1 Add Conversation Memory

```python
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationalRetrievalChain
# Create memory object
memory = ConversationBufferMemory(
    memory_key="chat_history",
    return_messages=True
)

# Chain that combines retriever + memory + LLM
chat_chain = ConversationalRetrievalChain.from_llm(
    llm=ChatOpenAI(model="gpt-4o", temperature=0.2),
    retriever=retriever,
    memory=memory
)

# Turn 1: employee asks about leaves
r1 = chat_chain.invoke({"question": "How many casual leaves do I get?"})
print(r1["answer"])

# Turn 2: follow-up — "them" refers to casual leaves
r2 = chat_chain.invoke({"question": "Can I take them in half days?"})
print(r2["answer"])
```

**📖 Memory Types in LangChain**

- **ConversationBufferMemory** — stores the complete conversation history (every message). Simple but grows large for long conversations.
- **memory_key="chat_history"** — the variable name injected into the prompt. The prompt template must have a `{chat_history}` slot.
- **return_messages=True** — stores history as structured message objects (HumanMessage, AIMessage) rather than plain text. Required for chat models.
- **ConversationalRetrievalChain** — a ready-made chain that: (1) rephrases follow-up questions using history, (2) retrieves relevant chunks, (3) generates an answer. All automatically.
- Other memory types: **ConversationSummaryMemory** (summarizes old history to save tokens), **ConversationBufferWindowMemory** (only keeps last N turns).

```
Turn 1 → You are entitled to 12 casual leaves per year at Zestora.

Turn 2 → Yes, casual leaves can be availed in half-day increments (minimum 0.5 days).
          You can take them on any working day with prior manager approval.
          (Bot remembered "them" = casual leaves from the previous question)
```

### 5. Phase 5 — Agents & Tools: SkillBot Takes Action

**Business Problem:** Some employee questions go beyond what's in the HR PDF — like "What is the current EPF contribution rate in India?" (changes based on government rules). SkillBot needs to search the web. Other times, an employee escalates a complaint and SkillBot must send an email to HR. This requires an **Agent** that can decide which **Tool** to use based on the question.

#### 5.1 How Agents Work — The Think → Act Loop

👤 Employee Question

"What is the current EPF rate in India?"

▼

Agent THINKS

🧠 LLM Reasoning (ReAct)

Thought: This is about current Indian law, not in HR PDF. Action: Use web_search tool. Action Input: "EPF contribution rate India 2025"

▼

Agent ACTS

🛠️ Tool Execution

web_search("EPF contribution rate India 2025") → returns: "Employee: 12%, Employer: 12% as of 2024"

▼

Agent OBSERVES → THINKS again → Final Answer

💬 Final Response

"The current EPF contribution rate is 12% from both employee and employer, as per EPFO guidelines 2024."

#### 5.2 Define Tools for the Agent

```python
from langchain_core.tools import tool
from langchain_community.tools import DuckDuckGoSearchRun
import smtplib
# Tool 1: Web search
search = DuckDuckGoSearchRun()

# Tool 2: Custom email escalation tool
@tool
def escalate_to_hr(issue: str) -> str:
    """Escalate an unresolved employee issue
    to the HR team by sending an email."""
# In production: use smtplib or SendGrid
print(f"EMAIL SENT TO HR: {issue}")
    return "Your issue has been escalated to HR. They will respond within 2 business days."
tools = [search, escalate_to_hr]
```

**📖 Defining Tools**

- **@tool decorator** — the simplest way to turn any Python function into a LangChain tool. The function's name becomes the tool name. The docstring tells the agent WHEN to use this tool — write it clearly.
- **DuckDuckGoSearchRun** — a ready-made tool that searches DuckDuckGo (no API key required). Returns a text summary of the top search results.
- **The docstring is critical** — the agent decides which tool to use based on the docstring. "Escalate an unresolved employee issue to the HR team" tells the agent: use this when the employee needs human HR help, not just a policy lookup.
- **Type hints** — `issue: str` and `-> str` help LangChain validate the tool's inputs and outputs automatically.
- Tools can do anything: query a database, call an API, read a file, send an SMS. If it's a Python function, it can be a tool.

#### 5.3 Create and Run the Agent

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate
agent_prompt = ChatPromptTemplate.from_messages([
    ("system", "You are SkillBot, HR assistant at Zestora. Use tools when needed."),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}")
])

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# Create the agent
agent = create_tool_calling_agent(llm, tools, agent_prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# Run the agent
result = executor.invoke({"input": "What is the current EPF rate in India?"})
print(result["output"])
```

**📖 Agents & AgentExecutor**

- **create_tool_calling_agent** — creates a modern OpenAI-style agent that calls tools through the LLM's native function-calling API. More reliable than the older ReAct text-parsing approach.
- **{agent_scratchpad}** — a special placeholder that the agent uses to record its intermediate thoughts and tool results. The agent writes its reasoning here as it goes through the Think → Act → Observe loop.
- **AgentExecutor** — the loop runner. It keeps running the agent until it reaches a final answer (or hits max_iterations). It handles tool execution, error catching, and feeding results back to the LLM.
- **verbose=True** — prints every step the agent takes: "Invoking tool web_search with input...", "Tool returned...", etc. Very useful for debugging. Set to False in production.
- **temperature=0** — for agents, always use 0. Agents make decisions that call external tools — you want deterministic, reliable choices, not creative randomness.

```
Invoking: `duckduckgo_search` with `EPF contribution rate India 2025`

The current EPF (Employee Provident Fund) contribution rate in India is:
- Employee contribution: 12% of Basic + DA
- Employer contribution: 12% of Basic + DA (3.67% to EPF, 8.33% to EPS)

This is mandated by the EPFO for all companies with 20+ employees.
Source: EPFO official guidelines.
```

### 6. Phase 6 — Putting It All Together: Complete SkillBot

Now we combine all phases — RAG + Memory + Agent — into a single, clean SkillBot class that Zestora can deploy.

**Scene 6 — Final Review | "Ship It"**

> **Meghna** _Head of HR — Zestora Technologies_
> 
> "Priya, I just tested it for 20 minutes. It answered leave queries correctly. It fetched the EPF rate from the web. When I typed 'I have a harassment complaint,' it immediately escalated to HR email. This is exactly what I wanted. When can we go live?"

> **Priya** _Senior AI Engineer — Zestora Technologies_
> 
> "Monday. We wrap it in Streamlit this weekend. 30 lines of UI code. Employees access it from the intranet. No logins needed — just type and get answers."

#### 6.1 Complete SkillBot Class

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationalRetrievalChain
from langchain_community.tools import DuckDuckGoSearchRun
class SkillBot:
    def __init__(self):
        # Load the HR vector database
self.db = FAISS.load_local(
            "skillbot_db", OpenAIEmbeddings(),
            allow_dangerous_deserialization=True
        )
        # Set up memory and conversational chain
self.memory = ConversationBufferMemory(
            memory_key="chat_history", return_messages=True
        )
        self.chain = ConversationalRetrievalChain.from_llm(
            llm=ChatOpenAI(model="gpt-4o", temperature=0.2),
            retriever=self.db.as_retriever(search_kwargs={"k": 4}),
            memory=self.memory
        )
        self.search = DuckDuckGoSearchRun()

    def chat(self, question: str) -> str:
        # Web search for regulatory questions
if any(w in question.lower() for w in ["epf", "tds", "government", "law"]):
            return self.search.run(question)
        # Policy questions → RAG chain with memory
result = self.chain.invoke({"question": question})
        return result["answer"]
```

> **Class structure** — wrapping SkillBot in a class means all state (memory, chain, vector store) is contained. Creating a new instance = a new conversation session. Clean, reusable design.
**__init__** — runs once when you create the bot. Loads the pre-built FAISS database from disk. No need to re-process the PDF on every chat.
**keyword routing** — a simple but effective pattern: if the question contains "EPF", "TDS", "law" etc., route to web search (live data). Otherwise, use RAG from the HR PDF. You can make this smarter with an LLM classifier later.
**self.chain.invoke()** — handles retrieval + memory + LLM automatically. Memory is stored inside the chain and updated after every turn.

#### 6.2 Streamlit UI — 30 Lines

```python
import streamlit as st
from skillbot import SkillBot

st.set_page_config(page_title="SkillBot — Zestora HR", page_icon="🤖")
st.title("🤖 SkillBot — Your HR Assistant")

# Create bot once per session (not on every message)
if "bot" not in st.session_state:
    st.session_state.bot = SkillBot()
    st.session_state.messages = []

# Show chat history
for msg in st.session_state.messages:
    with st.chat_message(msg["role"]):
        st.markdown(msg["content"])

# Chat input box at the bottom
if prompt := st.chat_input("Ask about leaves, policies, EPF..."):
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"): st.markdown(prompt)
    with st.chat_message("assistant"):
        with st.spinner("Thinking..."):
            answer = st.session_state.bot.chat(prompt)
        st.markdown(answer)
    st.session_state.messages.append({"role": "assistant", "content": answer})
```

> **st.session_state** — Streamlit's in-memory store. Because Streamlit re-runs the entire script on every user interaction, you store the bot object and message history here so they persist across messages without rebuilding the LLM chain.
**"bot" not in st.session_state** — creates the bot only on the first load. Without this, a new SkillBot (and new memory) would be created on every message, wiping the conversation.
**st.chat_input()** — renders a standard chat input box at the bottom of the page. The walrus operator `:=` checks if the user typed something and assigns it to `prompt` in one line.
**st.spinner()** — shows a loading animation while the LLM processes the request. Important for UX — LLM calls can take 1–5 seconds.
Run with: `streamlit run app.py`. Opens in your browser at `localhost:8501`.

### 7. Core LangChain Concepts — Quick Reference

Concept

What It Does

When to Use

LLM / ChatModel

The AI brain (GPT-4o, Claude, Gemini)

Every app needs one — it's the core.

PromptTemplate

Reusable message blueprint with variable slots

Whenever you talk to an LLM consistently.

Chain (LCEL)

Connect components with | pipe operator

Multi-step pipelines: retrieve → prompt → LLM.

DocumentLoader

Reads PDFs, CSVs, URLs, Notion, etc.

When your data isn't in the LLM's training.

TextSplitter

Breaks documents into manageable chunks

Always needed before embedding documents.

Embeddings

Converts text to numeric vectors

For semantic search and RAG.

VectorStore (FAISS)

Stores and searches embedding vectors

The "database" for your RAG application.

Retriever

Fetches relevant chunks for a given query

In every RAG pipeline.

Memory

Stores conversation history

Chatbots, multi-turn assistants.

Tool

A function the LLM can call (search, email, API)

When the LLM needs to take actions.

Agent

LLM that decides which tools to use and when

Complex tasks requiring reasoning + actions.

AgentExecutor

Runs the agent's Think→Act loop

Needed to actually execute agents.

OutputParser

Extracts/structures the LLM's raw response

When you need JSON, lists, or plain text out.

### 8. Common Mistakes — Fresher vs Experienced

**⚠️ How Freshers vs Senior Engineers Approach LangChain**

- **❌ Fresher Mistake #1** — Putting the API key in code

- **❌ Fresher Mistake #2** — Re-embedding on every run

- **❌ Fresher Mistake #3** — Using temperature=1 for fact bots

- **❌ Fresher Mistake #4** — Not testing retrieval quality

##### ❓ Fresher Questions — Interview & Understanding

**Q: Q1: What is the difference between a Chain and an Agent?**

A: A **Chain** is a fixed pipeline — steps always run in the same order: retrieve → prompt → LLM → parse. You define the flow. An **Agent** is dynamic — the LLM itself decides which tools to call, in what order, and whether to call them at all. Chains are predictable and fast. Agents are flexible but can be slower and less predictable. Rule of thumb: use a Chain when you know exactly what steps to take. Use an Agent when the LLM needs to reason and decide.

**Q: Q2: Why do we need RAG? Can't we just put the whole PDF in the prompt?**

A: You could — GPT-4o supports up to 128K tokens (about 100 pages). But it's expensive (you pay per token) and slower. More importantly, for very long documents, LLMs tend to "lose focus" in the middle — research shows retrieval accuracy drops for content buried in a long context. RAG solves this by only sending the 3–5 most relevant chunks, keeping the prompt short, cheap, and focused. For 50+ page documents or production systems, RAG is always the right choice.

**Q: Q3: What is the difference between ConversationBufferMemory and ConversationSummaryMemory?**

A: **BufferMemory** stores every single message verbatim. Simple, accurate, but grows large — a 20-turn conversation can hit the context limit. **SummaryMemory** uses the LLM to summarize older messages into a paragraph, keeping only the recent turns in full. Slower (costs an extra LLM call) but handles very long conversations. Use BufferMemory for short sessions (customer support, HR queries). Use SummaryMemory for long-running assistants or research tools.

**Q: Q4: What does "embedding" actually mean in simple terms?**

A: An embedding is a list of numbers that represents the meaning of a sentence. Example: "How many leaves do I get?" becomes a list like [0.12, -0.34, 0.87, ...] — 1536 numbers in total. The key property: sentences with similar meaning have mathematically similar number lists. So "leave entitlement per year" is numerically close to "how many days off do I get?" — even though the words are different. FAISS uses these numbers to find the most semantically relevant chunks.

**Q: Q5: When should I use LangChain vs raw OpenAI API?**

A: Use the raw OpenAI API for simple, one-shot completions — a single LLM call with no retrieval, no memory, no tools. LangChain adds value when you need: document loading + splitting + embedding, conversational memory, agents that call tools, structured output parsing, or multi-step chains. If you're building a simple text summarizer, raw API is fine. If you're building a chatbot that reads PDFs and takes actions, LangChain saves you hundreds of lines of boilerplate.

**Quiz: 🧠 Quick Check: Which LangChain component would you use to store and search employee performance review PDFs for a Q&A bot?**

- A) ConversationBufferMemory
- B) PromptTemplate
- C) FAISS VectorStore + Retriever
- D) AgentExecutor

> **Answer/explanation:** **✅ Answer: C — FAISS VectorStore + Retriever**

The PDFs need to be loaded, split into chunks, embedded, stored in FAISS, and retrieved at query time — this is the RAG pipeline.
A (Memory) — stores conversation history, not documents. B (PromptTemplate) — formats messages, doesn't store documents. D (AgentExecutor) — runs agents, not storage.
Complete flow: PyPDFLoader → RecursiveCharacterTextSplitter → OpenAIEmbeddings → FAISS.from_documents() → db.as_retriever() → ConversationalRetrievalChain.

> **LangChain Project — Core Takeaways for Freshers**

> - **Prompt Engineering is 60% of the work.** The LLM's answer is only as good as your prompt. Invest time writing a clear system message — include who the bot is, what it can/can't do, and what to say when it doesn't know. Bad prompts produce hallucinations. Good prompts produce reliable answers.
> - **Build and save the vector store once.** Embedding a 50-page PDF costs a few rupees and takes 30 seconds. Save it to disk. Load it on startup. Never re-embed unless the document changes. Freshers who embed on every run waste money and time.
> - **RAG keeps LLMs grounded in facts.** Without RAG, LLMs answer from training data (cutoff 2024, no company-specific info). With RAG, they answer from your actual documents. For any enterprise application — HR bots, legal bots, support bots — RAG is non-negotiable.
> - **Memory has a cost.** Every message you keep in memory takes tokens. A 100-turn conversation with BufferMemory can hit the context limit. In production, use ConversationSummaryMemory or limit history to the last 10 turns.
> - **Agents are powerful but use them carefully.** An agent with an email tool can send emails. An agent with a database tool can modify data. Always add guardrails — approval steps, dry-run modes, rate limits — before giving agents real-world powers.
> - **Never put API keys in code.** Use .env files and python-dotenv. Add .env to .gitignore. A leaked OpenAI key can cost lakhs of rupees if someone runs it overnight with automated requests.
> - **Test your retriever before you test the full chain.** If the wrong chunks are retrieved, even a perfect LLM won't give the right answer. Always test: does `retriever.invoke("question")` return the chunks you expect?
> - **temperature=0 for factual bots.** HR, legal, finance, and support bots must be deterministic. Creative outputs ("temperature=1") are for writing assistants, story generators, and brainstorming tools — never for fact-based systems.

##### LangChain Production Standards — Zestora Engineering Rules

- Always load API keys from environment variables — never from code. Use a secrets manager (AWS Secrets Manager, GCP Secret Manager) in production, not .env files.
- Log every LLM call with the prompt, response, and latency using LangSmith (LangChain's own tracing platform) — essential for debugging production issues and measuring quality.
- Set `max_tokens` on every LLM call. Without a limit, a runaway chain can generate thousands of tokens and cost thousands of rupees in API fees.
- Add fallback handling — if the LLM raises an exception or returns an empty response, return a graceful error ("SkillBot is unavailable, please contact hr@zestora.in") instead of a crash.
- Version your FAISS index alongside your document source. When the HR policy PDF is updated, rebuild the index and bump the version. This prevents answers from stale data.
- Evaluate your RAG pipeline regularly — create a test set of 20–30 HR questions with known correct answers, run them through SkillBot monthly, and measure accuracy. LLM quality can drift with model updates.

##### 🏋️ Hands-On Exercises — Extend SkillBot

1. **Add a follow-up classifier:** Before routing to web search or RAG, add an LLM classifier that detects if the question is a follow-up (e.g., "what about for part-timers?" after a leave question). If follow-up is detected, rephrase the question using chat history before passing to the retriever. Use a ChatPromptTemplate with a one-shot example to guide the classifier.
2. **Add structured JSON output:** Modify the RAG chain to return answers as JSON with fields `answer`, `policy_section`, and `confidence`. Use LangChain's `JsonOutputParser` with a Pydantic schema. Display the source section in the Streamlit UI so employees know exactly which policy clause answered their question.
3. **Build a leave tracker integration:** Create a custom LangChain tool called `check_leave_balance` that accepts an employee ID and queries a mock SQLite database of leave balances. Integrate it into the agent so employees can ask "How many leaves do I have remaining?" and get their actual balance, not just the policy maximum.
4. **Switch to a local LLM:** Replace `ChatOpenAI` with `ChatOllama` (from langchain-community) using the Llama 3 model running locally via Ollama. This makes SkillBot run entirely offline — no data leaves Zestora's network. Test if answer quality is acceptable for HR queries versus GPT-4o.
5. **Add LangSmith tracing:** Sign up for LangSmith (free tier available), set `LANGSMITH_API_KEY` in your .env, and run 10 SkillBot conversations. Open the LangSmith dashboard and find: (a) which question had the highest latency, (b) which retriever call returned irrelevant chunks, (c) the average tokens per conversation. This is how AI engineers debug production issues.

### LangChain Project Complete 🎉

You have built SkillBot — Zestora's production AI HR assistant. You built a complete RAG pipeline that reads PDFs and retrieves relevant policy chunks, added conversational memory so the bot remembers what was discussed, created an agent with web search and email escalation tools, wrapped it in a Streamlit chat UI, and deployed it for 800+ employees to use. This is the exact architecture used by real enterprise AI assistants.

> **Meghna**
> 
> "First week live: SkillBot handled 847 employee queries. HR team handled 12 — only the genuinely complex ones it escalated. Before SkillBot, we spent 4 hours a day on repetitive questions. Now it's 20 minutes. And the employees love it — they get instant answers at 11pm on a Sunday when they're planning their leaves. This is what AI engineering looks like when it actually solves a real problem."

> **Priya**
> 
> "Rahul, you built the vector store, wrote the tool docstrings, and caught the temperature=1 bug that was causing hallucinations. Those aren't small contributions. That's the difference between a bot that works in a demo and one that works in production. You're not a fresher anymore."

> **Next: Advanced LangChain — LangGraph, Multi-Agent Systems & Evaluation**

> - **LangGraph** — build stateful, multi-step AI workflows as graphs (nodes + edges). Enables complex reasoning: plan → research → draft → review → revise. The modern replacement for sequential chains.
> - **Multi-Agent Systems** — orchestrate multiple specialist agents (research agent, writing agent, critic agent) working together. One agent delegates to another. Used by AutoGPT, CrewAI, and enterprise AI systems.
> - **LangSmith** — LangChain's production monitoring and evaluation platform. Track every LLM call, measure accuracy, run A/B tests between prompts, catch regressions when you change the LLM model.
> - **Structured Outputs with Pydantic** — force LLMs to return valid JSON matching a strict schema. Essential for connecting LLM output to downstream code (APIs, databases, UI components).
> - **Local LLMs with Ollama** — run Llama 3, Mistral, Phi-3 locally with zero API cost and zero data leaving your machine. Critical for enterprise apps where data privacy is mandatory.
> - **RAG Evaluation** — measure your RAG pipeline's accuracy with RAGAS metrics: faithfulness (does the answer match the retrieved chunks?), answer relevancy, context precision. Know if your bot is reliable before your users discover it isn't.
> - **Fine-tuning vs RAG** — when is fine-tuning an LLM better than RAG? Answer: fine-tuning teaches the model a new style or task. RAG gives it new knowledge. Most enterprise use cases need RAG, not fine-tuning — and understanding the difference is a senior AI engineer skill.
