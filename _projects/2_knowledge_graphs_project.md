---
layout: page
title: Knowledge graphs project
description: Experimenting with knowledge graphs and RAG applications
img: assets/img/resultaggregation.png
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

{% include figure.liquid loading="eager" path="assets/img/generation.png" title="ingestion page" class="img-fluid rounded z-depth-1" %} 

We can see the resulting graph in the Neo4j UI:

{% include figure.liquid loading="eager" path="assets/img/graph.png" title="resulting graph" class="img-fluid rounded z-depth-1" %} 

Finally, I tried a query in natural language, and the result is pretty satisfying !

{% include figure.liquid loading="eager" path="assets/img/query_output.png" title="querying the graph, first example" class="img-fluid rounded z-depth-1" %} 

{% include figure.liquid loading="eager" path="assets/img/query_output2.png" title="querying the graph, second example" class="img-fluid rounded z-depth-1" %} 

## Improving the draft
### Grouping attribute nodes
Looking at the graph, something directly catches my eyes: we have lots of nodes linked to our main node "Pretraining" with the same relationship "has_challenge" or "has_example". 
From my human point of vue when I am looking at the graph (and when I will be searching it later) I would prefer to see a group node like "pretraining examples" linked to the Pretraining node, and directing to all the examples.
It has several advantages: 
- if i ask myself what do i have in DB about a particular concept I can just retrieve the 1st connected nodes and I will get all the information like : "definition", "examples", "related concepts", "applications".
- hopefully it may reduce the probability of missing an interesting node. For example if I want ALL the examples for a certain concept and I get only 10 results maximum, I cannot retrieve all interesting data if there are > 10 examples. I could increase the number of results fetched or go deeper in the graph, but it seems more optimal to me to just fetch the concept node and its first level relatives first using vector search for example, and then fetch all the examples using a precise and simple cypher query.
- i can implement that when i do a search, whenever I collect a group node, I must collect all related nodes

{% include figure.liquid loading="eager" path="assets/img/resultaggregation.png" title="aggregation result" class="img-fluid rounded z-depth-1" %} 

## Some lessons learnt from this project

- As huggingface recommends, outputting code instead of JSON or formatted text like in neo4j tutorials proves to be more reliable.
- The main challenge is how to generate the graph objects from the input text.
  - Instead of using a single prompt to extract data and do the conversion into Cypher I broke it down into several steps. Indeed, I remarked that generating Cypher is complicated for the LLM, so it struggles to do everything reliably at once. That is why I made the formatting into Cypher query an independent step.
  - One must think about the application when building the graph. Depending on the application, nodes and relationships do not have the same meaning. I struggled a bit before understanding how I wanted to create my knowledge base; considering nodes as concepts and relationships as attributes.
