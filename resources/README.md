# Databricks Resources

Handy external resources for going beyond Spark internals: how Databricks fits into real-world industry architectures, and how to keep up with the platform as it evolves.

---

## 1. Databricks Reference Architecture 2.0 (Blog)

**Link:** [Interactive Databricks Reference Architectures Across 70 Industries](https://www.databricks.com/blog/interactive-databricks-reference-architectures-across-70-industries)

The official Databricks blog post (Nov 2026) that introduces the interactive reference architecture board and explains why it replaces static architecture slides. It covers:

- How an industry board is organised: **source systems, data formats, teams, medallion layers, use cases, Genie Agents, AI/BI Dashboards and Databricks Apps**.
- The **70 industries across 11 sectors** (financial services, healthcare and life sciences, manufacturing, energy, retail, telecom, and more).
- **Region and release-stage checks**: see which products and features run in a region, and whether they are GA, Public Preview or Beta, with links to the docs.
- **Exports and editing**: PowerPoint, PDF, PNG, draw.io, Visio, Excalidraw and YAML, in 16 languages and on AWS, Azure and GCP.

Best for: understanding the *what* and *why* behind the tool before you open it.

---

## 2. Databricks Reference Architecture App

**Link:** [Databricks Reference Architecture (interactive app)](https://databricks-industry-solutions.github.io/databricks-reference-architecture/)

The public, interactive architecture board itself, published by Databricks Industry Solutions. Pick an industry and a cloud, then click any box to see what it is, who uses it, and where to learn more. Useful features:

- **Industry picker** with a search box, plus a neutral *Standard Reference Architecture*.
- **Simplified and Detailed views** of every board.
- **Region and stage filters** that dim products unavailable in your region or not yet GA.
- **Team learning paths** built from Databricks Academy courses, customer stories and demos.
- **AI Architecture Assistant** that builds an editable architecture from a description or a whiteboard photo (runs as a Databricks App in your own workspace).

Best for: designing or reviewing a lakehouse architecture for a specific industry, or preparing material for a customer meeting.

---

## 3. The Databricks Radar (brickster.ai)

**Link:** [brickster.ai](https://www.brickster.ai/)

An independent, free site that tracks the Databricks ecosystem daily. It collects releases, news, community questions and videos, and summarises them with AI. Main tools:

- **Pulse**: a live bubble cloud of what the Databricks world is talking about.
- **Roadmap**: every tracked feature with its status (GA, public preview, announced, deprecated) and a link to the source post.
- **Knowledge Graph**: topics linked by how often they appear together.
- **Topics and Digest**: about 60 topic landing pages (Unity Catalog, Genie, Delta Lake, MLflow, MCP and more) plus narrative digests.
- **Assistant and MCP**: ask questions with cited sources on the site, or plug the same retrieval into Claude Desktop, Cursor or any MCP-aware client.

Best for: staying current on new releases and feature status. It is a community project, not an official Databricks site, so confirm important details in the official docs.

---

## How these fit together

| Need | Start with |
|------|-----------|
| Understand the idea behind the reference architectures | [Reference Architecture blog](https://www.databricks.com/blog/interactive-databricks-reference-architectures-across-70-industries) |
| Explore or export an architecture for your industry | [Reference Architecture app](https://databricks-industry-solutions.github.io/databricks-reference-architecture/) |
| Track new releases and feature status | [The Databricks Radar](https://www.brickster.ai/) |

Looking for Spark internals instead? Head back to the [story index](../README.md).
