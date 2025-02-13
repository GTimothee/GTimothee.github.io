---
layout: page
title: Knowledge graphs project
description: Experimenting with knowledge graphs and RAG applications
img: assets/img/graph.png
importance: 1
category: RAG
related_publications: false
---

## Introduction

Goal of the project: I am trying to leverage knowledge graphs to create an interactive database. I am implementing a user interface with streamlit.

How it works:
1. put some text in a textbox
2. run the process button
3. data gets converted into graph objects in a multi-step process, using LLMs.
4. you can inspect the objects
5. click a button to push the new objects to database
6. switch to the query page
7. query in natural language, a LLM transforms the query into cypher query and runs it against the DB

Stack: 
- Python
- langchain (LLMs, retrieval chains)
- Neo4j (graphDB)
- streamlit (UI)
- MistralAI (LLMs)

## Preliminary results

I gave some input text to ingest: 

  In deep learning, pre-training refers to the process of optimizing a neural network before it is
  further trained/tuned and applied to the tasks of interest. This approach is based on an assumption
  that a model pre-trained on one task can be adapted to perform another task

The text gets processed and converted into graph objects:

[ui_img](assets/img/generation.png)

We can see the resulting graph in the Neo4j UI:

[ui_img](assets/img/graph.png)

Finally, I tried a query, and the result is pretty satisfying !

[ui_img](assets/img/query_output.png)
