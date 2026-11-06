# Code mode, atomic tools, and agent harnesses

Notes from a conversation about how an agent searches the web, calls tools, executes code, and might handle large-scale data work. The core epiphany: **code execution and atomic tools are complementary layers**. A harness can give an agent small, controlled capabilities and let it write a program that composes those capabilities when a task requires orchestration.

## 1. Web search is not the same thing as fetching a URL

A web-search tool accepts a query expressed as keywords or a question and returns discovered pages, typically with titles, URLs, and excerpts. You do not need to know the destination URL in advance. A web-fetch tool instead retrieves content from one or more *known* URLs. A shell command such as `curl` can make an HTTP request to a known endpoint if networking and permissions allow, but `curl` by itself is not a search index or search engine. It can query a search API if one is available and appropriately authorized.

In the environment used for this discussion, the web-search and web-fetch capabilities were exposed as tools, discoverable and callable through code mode. Search used a configured provider. Availability, retention, and pricing depend on the installation and organization policy. Search queries go to the provider; do not assume they are private to the local machine.

## 2. Code mode is a programmable interface to tools

In this environment, code mode accepts a small JavaScript program that can discover and invoke available tools. The JavaScript is not itself the web-search service, the shell, or the filesystem: it coordinates calls to the tools that provide those capabilities. In particular, this JavaScript sandbox does not directly expose the local filesystem or network; it accesses capabilities through provided tools.

Conceptually:

```text
agent chooses or writes a program
              ↓
code-mode runtime executes the program
              ↓
program calls exposed, permissioned tools
              ↓
program processes results and returns selected output
              ↓
agent reasons about the result
```

Tool discovery can also be lazy: rather than placing the definitions of every possible tool in the model's context, the agent can search for relevant tools and inspect their interfaces when needed. Discovering a tool is not the same as using it: looking up the search tool's interface need not send an actual web search.

This arrangement is an implementation pattern, not a universal definition: other harnesses expose different languages, permissions, runtimes, and tool interfaces.

## 3. Why keep atomic tools if code mode exists?

Tools such as `read`, `edit`, shell execution, web search, and web fetch provide concrete capabilities with defined inputs and outputs. Code mode can call some of them, but it does not make the underlying capabilities unnecessary. The runner still needs something that can read a file or start a shell command.

Direct calls remain valuable:

- A single file read or `git status` is simpler as a direct call than as a JavaScript wrapper.
- A purpose-built edit tool can offer precise replacement semantics rather than requiring a custom script for every edit.
- Separate tools have explicit interfaces and can have distinct permissions, auditing, validation, or safety controls. Those properties depend on the actual implementation; separation alone does not guarantee them.
- A shell is expressive for local command-line work, while an API-specific tool may provide structured inputs and results.

Conversely, atomic calls alone can be awkward when the task involves many steps, intermediate transformations, or conditional logic. The useful combination is **capabilities as tools; orchestration as code when needed**. Neither layer makes the other obsolete.

## 4. One command, several commands, and orchestration

Suppose four independent shell operations are needed. Depending on the harness, the agent can:

1. Call the shell tool four times sequentially. This is appropriate when a later call depends on an earlier result, or when avoiding concurrent load matters.
2. Ask a parallel-call mechanism to run the four calls concurrently. This is simple when their inputs are already known and the calls are independent.
3. Write code that starts four tool calls concurrently and gathers their results. This is useful when the program also needs to transform outputs, handle failures, or select subsequent work.

Illustrative pseudocode, not a portable API guarantee:

```js
const inputs = ["one", "two", "three", "four"];
const results = await Promise.allSettled(
  inputs.map(input => callTool({ input }))
);
const successes = results
  .filter(result => result.status === "fulfilled")
  .map(result => result.value);
return summarize(successes);
```

`Promise.allSettled` preserves individual success and failure outcomes; `Promise.all` rejects when one input rejects, though it does not automatically stop other work. Parallel execution is not inherently faster when tasks compete for the same CPU, disk, network, or provider quota. A shell can also coordinate several commands internally; the best boundary depends on visibility, permissions, error handling, and the task.

A parallel wrapper and a JavaScript program may both arrange concurrent calls, but one should not assume they use the same internal scheduler or provide identical cancellation and error guarantees.

## 5. The extra power is programmatic control flow

A simple parallel wrapper is often enough when all calls are known in advance. A program can additionally:

- Fan out over a variable number of inputs and collect results.
- Filter, aggregate, deduplicate, join, or rank results before they enter model context.
- Branch on a returned status or execute a second step only when a condition is met.
- Implement bounded concurrency, retries, and backoff **where the runtime supports the necessary primitives**, taking care with side effects.
- Combine sequential dependencies with independent parallel branches.
- Return a compact summary while preserving useful errors and references to artifacts.

This can reduce model round trips and the amount of intermediate data placed in the model's context. It is not guaranteed to reduce total compute, elapsed time, or cost in every task: sandbox execution, tool calls, network requests, and downstream data engines all have their own costs.

## 6. Racing and cancellation require precise semantics

Concurrency patterns do not automatically imply safe cancellation:

- `Promise.race` settles with the first promise to settle, whether it succeeds or fails. It does **not** automatically cancel the remaining promises.
- `Promise.any` instead resolves on the first fulfillment, or rejects if all inputs reject; it likewise does not itself cancel the losers.
- A runtime may provide cancellation signals, deadlines, or explicit cancellation APIs. Whether those reach an underlying tool, subprocess, or remote service is a separate question.
- In the code-mode environment discussed here, tool calls still running when the script ends are cancelled. That statement is specific to that environment. Cancellation is not a transactional rollback: a remote request, write, or subprocess may already have caused an effect.
- A timeout that stops waiting is not necessarily a timeout that stops the underlying work. Retrying a non-idempotent operation can duplicate its side effects.

For production designs, specify who owns each task, when cancellation is sent, what an interrupted operation guarantees, and whether cleanup or compensation is needed. Do not infer these guarantees just from the presence of async syntax.

## 7. Would Go, Erlang, or Rust change the picture?

Potentially, yes. A different host language can make particular concurrency structures easier to express:

- **Go:** goroutines, channels, and context-based cancellation.
- **Erlang:** lightweight processes, message passing, and supervision.
- **Rust:** async tasks, channels, and compile-time safety properties.
- **JavaScript:** promises and async/await are well suited to many concurrent I/O workflows.

But a language feature cannot manufacture a capability the harness does not expose. A Go `context` cannot guarantee that a remote API undoes a write; a Rust async task cannot necessarily terminate a tool if the tool interface has no cancellation mechanism. Runtime isolation, scheduling, supported APIs, resource limits, and the semantics of each tool matter at least as much as syntax.

## 8. Data cleaning and big data: orchestration is not the data engine

A data-focused agent could use this pattern:

```text
user's objective
    ↓
agent plans and generates a program or workflow
    ↓
harness runs it with controlled tool access
    ↓
inspect schema → profile columns → identify candidate issues
    ↓
issue SQL / data-engine commands → validate results → save artifacts
    ↓
return compact summaries, samples, and failures to the model
    ↓
agent explains findings or revises the plan
```

Examples of atomic capabilities include schema inspection, SQL query execution, data profiling, applying a proposed transformation, running a data-quality check, and writing a report. The orchestration code can coordinate those tools, including executing independent profiling checks concurrently when the data engine and resource limits allow it.

**The orchestration sandbox need not load the entire dataset.** For large datasets, the program can ask a database, warehouse, or distributed engine to scan, aggregate, or transform data near where it lives, then return a small result. A sandbox running a JavaScript loop is not automatically a distributed query engine. Similarly, a `pandas` example is not proof that a system can process data larger than available memory.

A realistic workflow should also account for:

- Schema drift, sampling bias, nulls, duplicate records, and ambiguous business rules.
- Validation before and after changes, including row counts and domain-specific invariants.
- Dry runs, previews, provenance, and human approval for destructive or consequential changes.
- Idempotency, retry behavior, transaction boundaries, and partial failure.
- Data access controls, sensitive values in logs and model context, and egress to external providers.
- Compute costs, query limits, bounded concurrency, and backpressure.
- Persisted artifacts and reproducibility: record the input version, transformation, parameters, validation, and output location where practical.

Compact summaries are useful, but they can hide important exceptions. Return sufficient evidence and representative failures for a human or model to assess the result.

## 9. Related systems and research found in web search

These examples illustrate related approaches; they are not interchangeable products, and their inclusion is not a recommendation or a claim that they all implement the same mechanism.

- **Anthropic, [Code execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp):** describes writing code to call MCP-connected tools and process results before returning them to the model, including a spreadsheet example. Its [programmatic tool-calling documentation](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling) identifies large-result processing and fan-out as use cases. Reported token savings in examples are workload-specific, not a universal performance guarantee.
- **Cloudflare, [Code Mode](https://blog.cloudflare.com/code-mode/) and [Agents implementation](https://github.com/cloudflare/agents/blob/main/docs/codemode.md):** describes presenting tools as a TypeScript API so generated code can compose tool calls, with a sandboxed execution model.
- **Redpanda, [Code Mode](https://docs.redpanda.com/agentic-data-plane/gateway/code-mode):** describes searching for tools on demand and executing code in a limited JavaScript sandbox. Its documented constraints illustrate that code mode is still governed by per-runtime limits.
- **Glean, [Agent Sandbox & Programmatic Tool Calling](https://docs.glean.com/security/agent-sandbox-ptc):** describes code fetching, filtering, and merging enterprise data before returning results to the agent.
- **LangChain, [data analysis agent](https://docs.langchain.com/oss/python/deepagents/data-analysis) and [agent from scratch](https://docs.langchain.com/oss/python/langchain/deep-agent-from-scratch):** describe agents that plan data analysis and execute Python in a sandbox, producing scripts, reports, or visualizations. This is related to, but not necessarily identical to, MCP-style programmatic tool orchestration.
- **Snowflake, [Cortex Agent code execution](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-code-execution-tool):** describes sandboxed scripts for processing data, calculations, and visualizations in an agent session. Do not infer distributed-scale capability from the existence of a sandbox alone.
- **DataSQRL, [agentic data-engineering toolkit](https://docs.datasqrl.com/):** describes an open-source toolkit for data-engineering agents focused on human control, correctness, and safety. Its exact architecture warrants separate inspection.
- **AutoDCWorkflow, [paper](https://arxiv.org/html/2412.06724v2) and [LLM4DC repository](https://github.com/LanLi2017/LLM4DC):** research on using LLM planning to generate data-cleaning workflows and execute them with OpenRefine. It is an example of a model planning operations for a specialized data tool, not evidence of general production readiness.

**Evidence limitation:** these pointers and summaries were gathered from web-search result excerpts. A subsequent attempt to fetch the full pages failed because the configured fetch service had insufficient credits. They were not independently verified by a full reading or by running the projects. Documentation and implementations can change; check primary sources before relying on operational details.

## 10. The consolidated epiphany

A capable agent system does not have to choose between a huge collection of isolated, model-driven tool calls and one unrestricted code tool. It can provide **small, well-defined capabilities**, then allow the agent to **compose them in a sandboxed program** for the parts of a task that benefit from ordinary control flow and concurrency. For data-intensive work, that program can coordinate specialized engines and pass compact evidence back to the model.

The strength of the pattern comes from the division of labor: the model plans and interprets; code coordinates; tools and data engines perform concrete work; and the harness enforces the actual boundaries. The safety and performance of that division depend on the real implementations, not on the label “code mode.”
