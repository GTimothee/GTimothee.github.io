---
layout: post
title: RAG agent with Neo4j tuto update
date: 2025-02-11 17:10:00-0100
inline: false
related_posts: false
---

Just open sourced a new repo: [RAG agent with Neo4j](https://github.com/GTimothee/neo4j-rag-agent/tree/main)

This is a repo to showcase how to quickly setup a RAG agent with Neo4j graph database. It include vector search and graph RAG using Neo4j's Cypher queries.

Disclaimer: I just followed a tutorial to create a RAG agent working with a Neo4j DB.

The added value of this repo is:
- I updated the code (imports/API usage were outdated)
- They created an agent with the deprecated langchain's agent object, whereas I created the agent with Huggingface's smolagents.
- Its simplicity.

Reference: [Neo4j tutorial](https://neo4j.com/developer-blog/knowledge-graph-rag-application/)
