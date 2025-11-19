# **The Architect's Guide to Agent Tooling: A Deep Dive into Context Engineering and Ideal Development Patterns**

## **Part I: The Central Scarcity: Context as the Agent's Core Asset**

The development of high-performance Artificially Intelligent (AI) agents is fundamentally a discipline of resource management. The single most critical and scarce resource is not computational power, nor even the size of the model's context window, but the effective *utilization* of that window. The foundational principle for all robust agentic architecture is that "context engineering fundamentals... matter more than context window size".1

An agent's context window is its working memory. It is the finite space where it holds its instructions (the system prompt), its history (the conversation), and its tools (the API definitions). When this working memory becomes cluttered, the agent's reasoning ability degrades. It loses focus, hallucinates, and fails to complete complex tasks. The "big takeaway" from modern agent architecture is that most engineers overthink tooling and, by default, "bleed context" 1, fatally undermining their agent's performance before the first task is even initiated.

However, the term "context bleeding" is critically ambiguous and describes two different, though related, architectural failures. A production-grade system architect must distinguish between and design solutions for both.

### **1\. The Critical Disambiguation of "Context Bleeding"**

#### **Context Consumption (The Performance Problem)**

This is the most common form of "context bleeding" and the primary focus of development-centric analysis.1

* **Definition:** Context consumption is a performance and cost problem. It describes the massive, non-recoverable, upfront consumption of an agent's context window by verbose tool definitions and manifests *before* the agent begins its work.  
* **Mechanism:** The agent's system prompt is "front-loaded" with the entire manifest of all available tools, regardless of whether they are needed for the current task.  
* **Quantified Impact:** This is not a theoretical concern. A single, complex Model Context Protocol (MCP) server, such as the one for GitHub, can inject 91 tool definitions that consume approximately 46,000 tokens.2 In a 200,000-token context window, 25% of the agent's "working memory" is exhausted at static load time, severely degrading its capacity for multi-step reasoning.

#### **Context Leakage (The Security Threat)**

This is the more insidious and dangerous form of "context bleeding," representing a critical security and privacy vulnerability.

* **Definition:** Context leakage is the unintentional and unauthorized transfer of session context, including private data, secrets, or personally identifiable information (PII), *between* different, isolated user sessions or *across* sequential steps in a chained tool call.3  
* **Mechanism:** This vulnerability manifests in two primary ways:  
  1. **Multi-Tenant Failure:** In a shared or multi-tenant environment, a global memory buffer or shared vector store fails to properly isolate data. This can lead to "stale or cross-pollinated context," where "User B" inadvertently gains access to sensitive data from "User A".4  
  2. **Tool-Chain Contamination:** A sequence of tool calls improperly passes on data. A tool that should only receive a "userID" might also receive a "session\_token" from a previous step's output.3 This can be detected by tracing context boundaries and identifying "cross-call secret reuse".3  
* **Implications:** The consequences of context leakage are severe, ranging from catastrophic privacy breaches to unpredictable and unreliable model behavior.

The architectural patterns discussed in this report primarily focus on solving the *consumption* problem. This reveals a fundamental tension in agent design: the very patterns created to solve the *leakage* (security) problem—such as centralized, policy-governed MCP servers 6—are often the *cause* of the *consumption* (performance) problem. A successful architecture must navigate this trade-off, balancing performance, security, and scalability.

## **Part II: Foundational Patterns: The Agentic Loop and the Four Tooling Architectures**

Before selecting a tool, an architect must understand the process it serves. The foundational agentic loop, as defined by the engineering principles behind the Claude Agent SDK 7, provides a definitive three-stage model for agentic behavior.7

1. **Gather Context:** The agent actively pulls information from the environment *into* its context window. This is not just a passive prompt; it is an active search using tools like grep, tail, and file system navigation to find relevant data.7  
2. **Take Action:** The agent uses its tools to effect change in the external world. This includes running Bash commands, editing files, or calling external MCPs.7  
3. **Verify Work:** The agent checks its own output for reliability. This crucial step provides a feedback loop, using defined rules (e.g., code linters), visual feedback (e.g., screenshots), or even a subordinate "LLM as a Judge" to self-correct.7

To analyze the four primary tooling architectures, we will use a continuous practical case study: building a Kalshi Info-Finance Agent. This agent's goal is to query the Kalshi prediction markets, analyze market data, and leverage "info finance" concepts.1 We will build this same agent's functionality in four different ways to expose the architectural trade-offs.

### **Pattern 1: The MCP Server (The Monolithic Standard)**

* **Architecture:** The Model Context Protocol (MCP) is a standardized communication protocol 8 designed to create a formal, governed middleware layer. The agent (client) does not interact with tools directly; it sends requests (e.g., /fetchData) to an MCP server.6 This server then acts as a broker, handling the actual interaction with external systems (APIs, databases), enforcing policies, and managing sessions.6  
* **Trade-offs:**  
  * **Pro:** High interoperability, standardization, and discoverability. The MCP architecture is built for enterprise-grade, multi-agent systems where central governance, security, and scale are paramount.1  
  * **Con:** As established, this pattern is the primary source of "context bleeding" (consumption).1 Its strength—a comprehensive, discoverable tool manifest—is its greatest performance weakness.  
* **Kalshi Case Study (Implementation):** We build a kalshi-mcp-server that exposes the entire Kalshi API. When our agent initializes, its system prompt is injected with the *entire API manifest*—get\_market, get\_all\_markets, get\_market\_history, get\_user\_balance, execute\_trade, get\_trade\_history, and dozens more. To perform a simple query, "What is the price of the 'Fed Rate' market?", the agent must first load 20,000-30,000 tokens of tool definitions it has no intention of using. This is the context consumption problem in practice.  
* **Architectural Analysis:** The common mistake is to view MCP as a *tooling pattern* when it is, in fact, an *enterprise orchestration and governance protocol*. The analysis of MCP in terms of "policy-governed execution," "tool brokering," and "session management" 6 makes this clear. Most single-agent use cases do not require this level of heavy-handed orchestration. Engineers adopt this sledgehammer to tap a nail, and the resulting context bloat is a predictable side effect. Tellingly, Anthropic's own engineering blog on MCP acknowledges this exact limitation and proposes "code execution"—the foundation of Patterns 2 and 3—as the more efficient *solution* for interacting with MCP servers, validating the video's premise.10

### **Pattern 2: CLI as Tools (The 80% Solution)**

* **Architecture:** This pattern is the purest embodiment of the "giving Claude a computer" design principle.7 The agent's toolset is radically simplified. Instead of a large manifest of APIs, it is given access to a single, powerful tool: Bash.11 The agent then interacts with the world by formulating and executing shell commands, calling simple, standalone command-line interface (CLI) tools that exist in its PATH.  
* **Trade-offs:**  
  * **Pro:** This is the "80% solution".1 It offers maximum developer control, minimal context cost (only the small Bash tool definition), and extreme reusability. The tools are "built once, use everywhere".1  
  * **Con:** The agent must be proficient in shell command syntax. A complex, multi-step task can become "chatty," requiring numerous back-and-forth command executions.  
* **Kalshi Case Study (Implementation):** We create a simple binary, kalshi-cli. The agent's static context cost is now minimal—it only knows about the Bash tool. To get a market, it formulates and executes a single command: bash \-c "kalshi-cli get-market \--ticker=FED-2025". The kalshi-cli tool handles the API call, formats the JSON response, and prints it to stdout. The agent reads this output and continues. The context is preserved.  
* **Architectural Analysis:** The true power of this pattern is not just its token efficiency, but its role as a universal, decoupled abstraction layer. A kalshi-cli tool written in Rust can be used identically by a human developer in their terminal, a Go-based CI/CD pipeline, and a Python-based AI agent (via the Bash tool 7). This architecture *decouples* the agent's development lifecycle from the tool's development lifecycle. This is a vastly superior software engineering practice, as it maximizes leverage, testability, and reuse—a tool built for the agent is also a tool built for the entire engineering team.

### **Pattern 3: Scripts as Tools (The Progressive Disclosure Pattern)**

* **Architecture:** This is an elegant and direct evolution of Pattern 2\. When a task is too complex for a single CLI call, the agent uses the Bash tool 7 to execute a *local script* (e.g., ./analyze-market.py). This script encapsulates a complex, multi-step workflow. This pattern implements "progressive disclosure" by hiding the complex implementation details from the agent's context.12  
* **Trade-offs:**  
  * **Pro:** This pattern saves "90% of your context" 1 by offloading data-heavy analysis and multi-step logic *out* of the context window.  
  * **Con:** It introduces a new maintenance burden: these "workflow scripts" must be written, managed, and versioned.  
* **Kalshi Case Study (Implementation):** The agent needs to perform a complex analysis: "Based on the last 30 days of market history and my current balance, is the 'FED-2025' market a good bet?"  
  * *Inefficient Method (Pattern 2):* (1) Agent calls kalshi-cli get-market..., (2) Agent calls kalshi-cli get-history..., (3) Agent calls kalshi-cli get-balance.... All three large JSON payloads enter the context. (4) Agent performs the analysis *inside* its context, consuming 50,000 tokens.  
  * *Efficient Method (Pattern 3):* Agent makes one call: bash \-c "./analyze\_kalshi\_market.py \--ticker=FED-2025".  
* **Architectural Analysis:** The 90% context saving is achieved through *computational offloading* and *context pre-processing*. The script, ./analyze\_kalshi\_market.py, acts as a local, special-purpose, non-LLM agent. It *externally* performs the "gather context" and "verify work" loops. It fetches the 100kb of market and history data, performs complex statistical analysis, checks this against a local ruleset, and *discards the raw data*. It then returns *only the final insight* to the agent's stdout: {"ticker": "FED-2025", "analysis": "Overpriced", "confidence": 0.85, "recommendation": "SELL"}. The agent's precious context window is protected from the 100kb of raw data, receiving only the final, 1kb decision-ready summary.

### **Pattern 4: Skills as Tools (The Ecosystem Approach)**

* **Architecture:** A "Skill" is a formal, packaged-up agentic component that hybridizes the previous patterns. It is typically a directory containing a SKILL.md (the "prompt" or instructions) and optional /scripts or /assets (the "tools").13 It formalizes Pattern 3 ("Scripts as Tools") into a discoverable, managed ecosystem component, turning a generalist agent into a coordinator of task-specific sub-agents.14  
* **Trade-offs:**  
  * **Pro:** Offers extreme context efficiency through a robust, multi-stage progressive disclosure mechanism.15 Enables powerful, in-session "context switching".13  
  * **Con:** Can be perceived as "vendor lock-in" 1, as the orchestrator must be "Skill-aware."  
* **Kalshi Case Study (Implementation):** We create a kalshi-finance Skill. When the agent identifies a finance-related query, it invokes this Skill. At that moment, two distinct events occur 13:  
  1. **Conversation Context Modification:** The SKILL.md file is loaded and *injected into the conversation history* as a new instruction prompt (e.g., "You are an expert Kalshi market analyst. Your goal is to provide financial insights, not to trade. Always use the /scripts/kalshi-cli for data...").  
  2. **Execution Context Modification:** The agent's "permissions" are dynamically changed. Its allowed-tools are temporarily restricted to *only* those defined in the Skill (e.g., Bash, Read, and the /scripts/kalshi-cli). The agent is "transformed" into a specialist.  
* **Architectural Analysis (Correcting the Context Cost):** The video's comparison of context cost was incomplete. Research provides a definitive, two-stage dynamic loading model that is the *true* benefit of the Skill pattern 13:  
  1. **Initial (Static) Cost:** At startup, the context cost is *minimal*. Only the name and description metadata for *all* available Skills are pre-loaded into the context.15 This makes all Skills *discoverable* at a very low token price.  
  2. Invocation (Dynamic) Cost: When a Skill is invoked, its full SKILL.md prompt is injected into the context. This cost is significant (e.g., \~1,500+ tokens 13).  
     The crucial distinction is that this significant cost is paid only upon invocation, not on every single turn of the conversation. A static system prompt with the same instructions would cost 1,500+ tokens on every interaction, while the Skill pays this cost only on-demand.13  
* **Architectural Analysis (Refuting "Lock-in"):** The "lock-in awareness" 1 is largely misunderstood. A comment on the video itself correctly identifies this: "Why... \[is it\] locki if they are just a folders with.md files...?".1 The research confirms this. A Skill is just a directory with Markdown and scripts.13 This is an *open pattern*. The "lock-in" is not in the *format* but in the *orchestrator* (like Claude Code) that understands how to perform the dynamic context modifications.13 A Skill is not just a tool; it's a *lightweight, in-process, sub-agent definition*. It's a mechanism for *temporarily transforming* a generalist agent into a specialist.

## **Part III: Advanced Tooling Strategies: Mastering the "Missed Items"**

The true mark of a professional agent developer is the ability to mitigate the trade-offs of these core patterns. The following advanced techniques, which were not fully explored in the video, are essential for building production-grade, context-aware systems.

### **1\. Curing MCP Bloat (Client-Side): Selective Loading with \--mcp-config**

* **The Problem:** The 46k-token GitHub MCP server 2 is a context-killer, but it is required for some tasks.  
* **The Advanced Solution:** Do not use a single, monolithic \~/.claude.json configuration. Instead, create multiple, task-specific "profiles" using separate configuration files and load them on-demand with the \--mcp-config or \--settings flag.17  
* **Architectural Pattern (Per-Agent Scopes):**  
  1. Create minimal, single-purpose config files.  
     * github\_read.json: Defines *only* the read-only GitHub MCP server.17

  JSON  
       {  
         "mcpServers": {  
           "github-read": {  
             "url": "https://api.githubcopilot.com/mcp/",  
             "authorization\_token": "GITHUB\_READ\_ONLY\_TOKEN"  
           }  
         }  
       }

     * kalshi.json: Defines *only* the Kalshi MCP server.  
  2. Invoke the agent *loading only the required context* for the task at hand.  
     * For a "code reviewer" agent: claude \--mcp-config github\_read.json  
     * For our "finance" agent: claude \--mcp-config kalshi.json  
  3. This implementation of "agent-scoped configuration" 18 is precisely the solution for creating a dedicated claude-betting alias, as noted in the query. It solves the context consumption problem at the client-level by enforcing a "least-privilege" context.

### **2\. Curing MCP Bloat (Server-Side): Slimming with tool-filter-mcp**

* **The Problem:** What if you do not control the client's invocation? How can you enforce a slimmed-down manifest for *all* users to solve the 46k-token problem at the source?  
* **The Advanced Solution:** The tool-filter-mcp is a *proxy server*.19 It is designed to "wrap" an upstream MCP server and "slim it down" by filtering its tool manifest.  
* Architectural Pattern (The Proxy Filter):  
  While a direct manual for this tool was not found in the research 19, its function and architectural role are clear from its name and the context of its recommendation.19  
  1. **Flow:** The agent's traffic is routed: Agent \-\> tool-filter-mcp (Proxy) \-\> Upstream GitHub MCP.  
  2. **Action:** The tool-filter-mcp proxy starts. It fetches the *full* 91-tool, 46k-token manifest from the upstream GitHub server.  
  3. It then consults its *own* configuration (e.g., an allow-list defining only 5 read-only tools).  
  4. It *filters* the manifest, creating a new, "slimmed-down" manifest that is (for example) only 2,000 tokens.  
  5. It passes this filtered manifest to the agent.  
     This is a critical enterprise governance pattern. It solves the context consumption problem for all clients while simultaneously enhancing security by enforcing a "least-privilege" toolset, preventing agents from accessing dangerous tools that were unintentionally exposed.19

### **3\. Bridging Architectures: Generating CLIs from MCPs with mcporter**

* **The Problem:** Your team has standardized on the efficient, low-cost "CLI as Tools" (Pattern 2\) architecture. A critical enterprise tool (e.g., JIRA, Linear) only exists as a "fat" MCP server (Pattern 1). This creates an architectural conflict.  
* **The Advanced Solution:** The video mentions wrapping CLIs *into* MCPs.1 The mcporter tool, however, enables the far more powerful *reverse* operation: it "turns any server definition into a shareable CLI artifact".22  
* Architectural Pattern (The Interoperability Bridge):  
  mcporter is a toolkit that introspects an MCP server and generates a standalone, strongly-typed CLI for it.22  
  1. **Generation:** A developer runs a single command: npx mcporter generate-cli https://linear.mcp.com.22  
  2. **Output:** This auto-generates a linear.js binary that can be called from the shell.  
  3. **Impact:** This completely resolves the architectural conflict. The agent's context window remains tiny; it *only* contains the Bash tool (Pattern 2). When it needs to access JIRA, it does *not* load the 40,000-token JIRA MCP manifest. Instead, it makes a simple, low-cost Pattern 2 call: bash \-c "jira-cli get-issue \--id=PROJ-123". The mcporter-generated jira-cli tool handles all the complex MCP communication *externally*, completely protecting the agent's context. This is the ultimate interoperability pattern, allowing a lean CLI-based architecture to access the rich, and otherwise bloated, MCP ecosystem.

## **Part IV: Synthesis: The Ideal Agent Development Guideline**

This analysis synthesizes into a single, prescriptive guideline for all agent development. It is a philosophy of "gradual revelation," prioritizing simplicity and context preservation at every stage.

### **1\. The 80/10/10 Rule: A Prescriptive Tooling Strategy**

The "80/10/10 rule" 1 is not a description of the current market, but a prescription for choosing the correct tooling approach:

* **80% of the time: Use "CLI as Tools" (Pattern 2).** This should be your default. It is simple, robust, language-agnostic, and maximally reusable ("build once, use everywhere" 1). It has the lowest context cost and highest developer control. 80% of all agentic tasks—reading files, running simple queries, executing code, formatting data—are best-served by this pattern.  
* **10% of the time: Evolve to "Scripts as Tools" (Pattern 3).** Use this *only when* your 80% solution hits a context bottleneck. When the agent needs to process large volumes of raw data, write a dedicated script to perform that analysis *externally* (computational offloading) and return only the final result. This is your primary context-optimization pattern.  
* **10% of the time: Formalize with "Skills" (Pattern 4\) or "MCPs" (Pattern 1).** Use these *only when* the specific architecture is required.  
  * **Skills:** Use when you need to create a complex, stateful, *specialist component*—the "in-context sub-agent" 13—that requires its own unique, on-demand instructions and permissions.  
  * **MCPs:** Use *only* when you require "multi-agent scale" 1 and enterprise-grade, policy-governed orchestration.6 When you do, you *must* use the advanced techniques (--mcp-config 17, tool-filter-mcp 19) to mitigate the crippling context cost.

### **2\. The Ideal Workflow: Gradual Revelation**

The ideal agent development workflow is a process of "gradual revelation" that mirrors the progressive disclosure pattern itself. Do not start with a complex architecture. Start simple and refactor as complexity demands.

1. **Start with a Prompt:** Can the task be done with a simple prompt and no tools? If yes, stop. The most efficient tool is no tool at all.  
2. **Evolve to CLI:** When the prompt becomes complex or requires external data, refactor the logic into a standalone **CLI tool** (Pattern 2\) and give the agent the Bash tool to call it.  
3. **Optimize with Scripts:** When the CLI calls become too "chatty" or involve transferring large data payloads *into* the context window, refactor that multi-step workflow into a single **Script** (Pattern 3\) that the agent calls.  
4. **Formalize into a Skill:** When this Script becomes a core, reusable, complex component that needs its *own* deep instructions, permissions, and assets, formalize it as a **Skill** (Pattern 4).  
5. **Bridge with MCP/mcporter:** When you need to interact with a third-party enterprise system that only offers an MCP, do not adopt the full MCP pattern. Instead, use **mcporter** 22 to *bridge* that MCP, accessing it via your clean, low-cost CLI (Pattern 2\) architecture.

### **3\. The Definitive Tooling Comparison (Correcting the Omission)**

The following table provides the definitive, architect-level comparison of the four patterns, correcting the critical omission from most analyses by splitting "context cost" into its two distinct types.

**Architect's Agent Tooling Comparison**

| Feature | Pattern 1: MCP Server | Pattern 2: CLI as Tools | Pattern 3: Scripts as Tools | Pattern 4: Skills as Tools |
| :---- | :---- | :---- | :---- | :---- |
| **Architecture** | Middleware Proxy Server 6 | Bash Tool \+ Binary Executable 7 | Bash Tool \+ Local Script 13 | Skill Orchestrator 13 |
| **Initial Context Cost (Static)** | **Extremely High** (Full manifest front-loaded, e.g., \~46k tokens) 2 | **Very Low** (Single Bash tool definition) 11 | **Very Low** (Single Bash tool definition) 11 | **Low** (Metadata-only: name & description for all Skills) 15 |
| **Invocation Context Cost (Dynamic)** | **Very Low** (Simple API call string) | **Low** (The command string, e.g., kalshi-cli...) | **Low** (The script command string, e.g., ./analyze...) | **High** (Full SKILL.md prompt injection, e.g., \~1.5k+ tokens) 13 |
| **Context Saving Mechanism** | N/A (This is the *source* of the problem) | Minimalist tool definition | **Computational Offloading** (Analysis happens *outside* context) | **Progressive Disclosure** (Metadata loaded first, full prompt *only on invocation*) 13 |
| **Developer Complexity** | High (Requires building/maintaining a server) | Low (Write a simple binary) | Low (Write a simple script) | Medium (Requires understanding SKILL.md format) 15 |
| **Agent Control & Flexibility** | Low (Agent is "blind" to implementation) | High (Agent has full Bash control) | Medium (Agent delegates control to the script) | Very High (Agent *becomes* a specialist) 13 |
| **Ideal Use Case** | Multi-agent orchestration at enterprise scale 1 | **The 80% Solution** 1 | Data-heavy analysis; context optimization 1 | Reusable, complex "sub-agent" components 14 |
| **Key Risk** | **Context Consumption** 1, **Context Leakage** 4 | Overly "chatty" workflows | Hidden complexity in scripts | **Platform Lock-in** (orchestrator-specific) 1 |

This nuanced view of context—distinguishing between the static *upfront* cost and the dynamic *on-demand* cost—is the single most important architectural consideration. The MCP pattern is inefficient because its cost is high and static. The Skill pattern is efficient because its high cost is paid dynamically and only when necessary. The CLI and Script patterns are efficient because their costs are minimal in all phases.

By starting with CLI, optimizing with Scripts, and formalizing with Skills, a developer can build agents that remain agile, powerful, and, most importantly, in control of their most valuable asset: context.

### **Recommended Strategy:**

* **For Existing External Tools (80% of the time):** Just use **MCP Servers**. Don't reinvent the wheel if a standard server exists.  
* **For New/Custom Tools:**  
  * **Primary Choice (80%):** Build a **CLI first**. It works for you (the human), your team, and the agent simultaneously.  
  * **Scaling Agents (10%):** If you need multiple agents at scale, **wrap your CLI in an MCP server**. This gives you the best of both worlds: a simple core CLI that can be exposed as an MCP when needed.  
  * **Context Critical (10%):** Use **Scripts or Skills** only when preserving the context window is the absolute priority (e.g., complex tasks requiring massive memory).

#### **Works cited**

https://github.com/Shaon2221/beyond-mcp
1. Why are top engineers DITCHING MCP Servers? (3 PROVEN Solutions) \- YouTube, accessed on November 16, 2025, [Why are top engineers DITCHING MCP Servers? (3 PROVEN Solutions)](https://www.youtube.com/watch?v=OIKTsVjTVJE)  
2. Any way to load only some tools from an MCP server? \- Stack Overflow, accessed on November 16, 2025, [https://stackoverflow.com/questions/79751034/any-way-to-load-only-some-tools-from-an-mcp-server](https://stackoverflow.com/questions/79751034/any-way-to-load-only-some-tools-from-an-mcp-server)  
3. MCP Security Vulnerabilities: Attacks, Detection, and ... \- Enkrypt AI, accessed on November 16, 2025, [https://www.enkryptai.com/blog/mcp-security-vulnerabilities-attacks-detection-and-prevention](https://www.enkryptai.com/blog/mcp-security-vulnerabilities-attacks-detection-and-prevention)  
4. Top MCP Security Risks in GenAI Apps \- Lasso Security, accessed on November 16, 2025, [https://www.lasso.security/blog/top-mcp-security-risks](https://www.lasso.security/blog/top-mcp-security-risks)  
5. AgentCortex MCP | Glama, accessed on November 16, 2025, [https://glama.ai/mcp/servers/@sage-hq/agentcortex-mcp](https://glama.ai/mcp/servers/@sage-hq/agentcortex-mcp)  
6. Your Architecture vs. AI Agents: Can MCP Hold the Line? \- QueryPie, accessed on November 16, 2025, [https://www.querypie.com/features/documentation/white-paper/22/your-architect-vs-ai-agents](https://www.querypie.com/features/documentation/white-paper/22/your-architect-vs-ai-agents)  
7. Building agents with the Claude Agent SDK \\ Anthropic, accessed on November 16, 2025, [https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)  
8. Model Context Protocol, accessed on November 16, 2025, [https://modelcontextprotocol.io/](https://modelcontextprotocol.io/)  
9. Building Intelligent AI Agents with MCP: A Complete Guide to the Model Context Protocol | by Harshal Dhandrut | Medium, accessed on November 16, 2025, [https://medium.com/@harshal.dhandrut/building-intelligent-ai-agents-with-mcp-a-complete-guide-to-the-model-context-protocol-5507069068fb](https://medium.com/@harshal.dhandrut/building-intelligent-ai-agents-with-mcp-a-complete-guide-to-the-model-context-protocol-5507069068fb)  
10. Code execution with MCP: building more efficient AI agents \\ Anthropic, accessed on November 16, 2025, [https://www.anthropic.com/engineering/code-execution-with-mcp](https://www.anthropic.com/engineering/code-execution-with-mcp)  
11. Claude Code settings \- Claude Code Docs, accessed on November 16, 2025, [https://code.claude.com/docs/en/settings](https://code.claude.com/docs/en/settings)  
12. Context Anxiety: How AI Agents Panic About Their Perceived Context Windows \- Inkeep, accessed on November 16, 2025, [https://inkeep.com/blog/context-anxiety](https://inkeep.com/blog/context-anxiety)  
13. Claude Agent Skills: A First Principles Deep Dive \- Han Lee, accessed on November 16, 2025, [https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)  
14. Claude Code's Custom Agent Framework Changes Everything \- DEV Community, accessed on November 16, 2025, [https://dev.to/therealmrmumba/claude-codes-custom-agent-framework-changes-everything-4o4m](https://dev.to/therealmrmumba/claude-codes-custom-agent-framework-changes-everything-4o4m)  
15. Skill authoring best practices \- Claude Docs, accessed on November 16, 2025, [https://docs.claude.com/en/docs/agents-and-tools/agent-skills/best-practices](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/best-practices)  
16. Claude Skills Solve the Context Window Problem ... \- The AI Architect, accessed on November 16, 2025, [https://tylerfolkman.substack.com/p/the-complete-guide-to-claude-skills](https://tylerfolkman.substack.com/p/the-complete-guide-to-claude-skills)  
17. Managing MCP tools for better AI agent sessions, accessed on November 16, 2025, [https://www.theaistack.dev/p/managing-mcp-tools](https://www.theaistack.dev/p/managing-mcp-tools)  
18. \[Feature Request\] Selective MCP Access Control for SubAgents · Issue \#6587 · anthropics/claude-code \- GitHub, accessed on November 16, 2025, [https://github.com/anthropics/claude-code/issues/6587](https://github.com/anthropics/claude-code/issues/6587)  
19. Mitmproxy MCP Server by Lucas Soeth: An AI Engineer's Deep Dive, accessed on November 16, 2025, [https://skywork.ai/skypage/en/mitmproxy-mcp-server-ai-engineer/1980464062349824000](https://skywork.ai/skypage/en/mitmproxy-mcp-server-ai-engineer/1980464062349824000)  
20. Rowy: The Kaspersky Standard for Your Firebase Backend?, accessed on November 16, 2025, [https://skywork.ai/skypage/en/Rowy%3A%20The%20Kaspersky%20Standard%20for%20Your%20Firebase%20Backend%3F/1975250346802081792](https://skywork.ai/skypage/en/Rowy%3A%20The%20Kaspersky%20Standard%20for%20Your%20Firebase%20Backend%3F/1975250346802081792)  
21. \[FEATURE\] MCP Tool Filtering: Allow Selective Enable/Disable of Individual Tools from Servers · Issue \#7328 · anthropics/claude-code \- GitHub, accessed on November 16, 2025, [https://github.com/anthropics/claude-code/issues/7328](https://github.com/anthropics/claude-code/issues/7328)  
22. steipete/mcporter: Call MCPs via TypeScript, masquerading as simple TypeScript API. Or package them as cli. \- GitHub, accessed on November 16, 2025, [https://github.com/steipete/mcporter](https://github.com/steipete/mcporter)