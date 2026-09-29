# Overview

Koog is an open-source JetBrains framework for building AI agents designed specifically for the JVM ecosystem.
It provides a first-class development experience for both Kotlin and Java developers, featuring an idiomatic, type-safe Kotlin DSL and fluent builder-style Java APIs.

While Java developers can leverage the full power of Koog on the JVM using idiomatic APIs, Kotlin developers can also deploy agents across JS, WasmJS, Android, and iOS targets using Kotlin Multiplatform.

- [**Quickstart**](quickstart/)

  ---

  Build and run your first AI agent
- [**Glossary**](glossary/)

  ---

  Learn the essential terms
- [**Module versioning**](module-versioning/)

  ---

  Understand stable vs. beta modules and API guarantees

## Agents

Learn about [agents in general](agents/) and how to create different types of agents using Koog:

- [**Basic agents**](agents/basic-agents/)

  ---

  Use a predefined strategy that works for most common use cases
- [**Functional agents**](agents/functional-agents/)

  ---

  Define custom logic as a lambda function in plain Kotlin or Java
- [**Graph-based agents**](agents/graph-based-agents/)

  ---

  Implement a custom workflow as a strategy graph
- [**Planner agents**](agents/planner-agents/) beta

  ---

  Iteratively build and execute a plan until the state matches the desired conditions

## Core components

Learn about the core components of Koog agents in detail:

- [**Prompts**](prompts/)

  ---

  Create, manage, and run prompts that drive the agent's interaction with the LLM
- [**Strategies**](predefined-agent-strategies/)

  ---

  Design the agent's intended workflow as a directed graph
- [**Tools**](tools/)

  ---

  Enable the agent to interact with external data sources and services
- [**Features**](features/)

  ---

  Extend and enhance the functionality of AI agents

## Advanced usage

- [**History compression**](history-compression/)

  ---

  Optimize token usage while maintaining context in long-running conversations using advanced techniques
- [**Agent persistence**](features/agent-persistence/)

  ---

  Restore the agent state at specific points during execution
- [**Structured output**](structured-output/)

  ---

  Generate responses in structured formats
- [**Streaming API**](streaming-api/)

  ---

  Process responses in real-time with streaming support and parallel tool calls
- [**Knowledge retrieval**](embeddings/) beta

  ---

  Retain and retrieve knowledge across conversations using [vector embeddings](embeddings/) and [RAG](retrieval-augmented-generation/)
- [**Tracing**](features/tracing/)

  ---

  Debug and monitor agent execution with detailed, configurable tracing
- [**Long Term Memory**](features/long-term-memory/) beta

  ---

  Integrate vector databases and memory providers for RAG and persistent memory.

## Integrations

- [**Model Context Protocol (MCP)**](model-context-protocol/) beta

  ---

  Use MCP tools directly in AI agents
- [**Spring Boot**](spring-boot/) beta

  ---

  Add Koog to your Spring applications
- [**Ktor**](ktor-plugin/) beta

  ---

  Integrate Koog with Ktor servers
- [**OpenTelemetry**](features/open-telemetry/)

  ---

  Trace, log, and measure your agent with popular observability tools
- [**A2A Protocol**](a2a/) beta

  ---

  Connect agents and services over a shared protocol
