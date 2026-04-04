# Segnog: Value Proposition and User Stories

At its core, interacting with LLMs is akin to working with a brilliant colleague who suffers from severe short-term memory loss. Every time you start a new conversation or spin up a new agent, you have to re-explain the rules, the background, and everything you've done so far. 

**Segnog solves this problem by providing structured, self-organizing agentic memory behind a single `/observe` endpoint.** 

Here are three core user stories that illustrate why you would use Segnog over a standard database or basic RAG (Retrieval-Augmented Generation) pipeline.

---

## 1. The Autonomous Coding Assistant (Long-Running Tasks)

**The Problem:** 
An AI agent is tasked with building a complex, multi-day web application. On Day 1, the agent discovers that a specific library version crashes due to a bug, and it makes an architectural decision to use an alternative library. On Day 3, a fresh session of the agent is started to build a new feature. Without memory, the agent might try to use the broken library again, completely forgetting the lesson from Day 1.

**The Segnog Solution (Background Knowledge Extraction):**
- **How it works:** Instead of manually telling the agent to "remember this," the agent simply pipes its daily operations to Segnog\'s `/observe` endpoint. 
- **The Magic:** Segnog immediately returns context to the agent, but *in the background*, it spins up a slower, cheaper process to extract structured facts and rules from the day\'s work (e.g., *"Rule: Do not use Library X due to memory leak"*).
- **The Value:** On Day 3, when the agent queries `/observe` while planning the new feature, Segnog searches its long-term knowledge graph and instantly injects the extracted rule into the context. **The developer gets a self-improving agent without writing a single line of summarization or vector database code.**

## 2. The Multi-Agent Swarm (Hierarchical Sessions)

**The Problem:** 
You are building an AI company orchestrator. You have a "CEO Agent" that breaks a massive research goal into 5 sub-tasks, and spins up 5 "Researcher Agents" to handle them. The researcher agents need to understand the CEO\'s overarching goal, but the CEO shouldn\'t have its own context polluted by the thousands of lines of raw data the researchers are reading.

**The Segnog Solution (Session Hierarchies):**
- **How it works:** Segnog supports **nested sessions**. The CEO agent operates in `session_id: "project-x"`. When it launches a researcher, it assigns `session_id: "task-1"` with a `parent_session_id: "project-x"`.
- **The Magic:** When Researcher 1 calls `/observe`, Segnog automatically traverses the hierarchy and injects the CEO\'s high-level goals into the researcher\'s context.
- **The Value:** Complete context isolation with inheritance. Sub-agents instantly know what the parent is doing, but the parent\'s memory trace remains clean and uncluttered.

## 3. The Personal AI Companion (Ontology & Graph Storage)

**The Problem:** 
A user talks to a personal AI companion every day. In January, the user mentions, "My sister, Sarah, is allergic to peanuts." In July, the user says, "I'm baking a cake for Sarah's birthday." A standard vector search (RAG) might fail to connect these thoughts because "peanut allergy" and "birthday cake" are not semantically similar in an embedding space.

**The Segnog Solution (Knowledge Graphs & Schema.org):**
- **How it works:** While the user is chatting, Segnog\'s background workers are silently building a graph database (FalkorDB). 
- **The Magic:** It parses the January conversation and creates a persistent Ontology node: `[Entity: Sarah] -> [Trait: Peanut Allergy]`. In July, when the user mentions "Sarah," Segnog sweeps its graph, finds the linked trait, and injects *"Memory: Sarah has a peanut allergy"* into the AI\'s prompt.
- **The Value:** The AI demonstrates profound, human-like recall of relationships and entities across months of disjointed conversations, making it feel truly intelligent rather than just a keyword-matcher.

---

## Technical Summary: Why not build this yourself?

To replicate Segnog, you would need to build:
1. A fast cache (Redis) for real-time conversation history.
2. A Vector Database to store semantic embeddings.
3. A Graph Database (FalkorDB) to store entity relationships.
4. An async task queue to summarize past conversations so they don't blow up the LLM token limit.
5. Complex prompt engineering to weave all three databases together into a readable paragraph.

**Segnog bundles all of this complexity into a single Docker container and a single API call:**

```
POST /observe {"content": "User said hello"}
```

It handles the routing, database management, embeddings, background extraction, graph mapping, and context chunking for you. You just plug it into your agent and it instantly gains a human-like memory system.
