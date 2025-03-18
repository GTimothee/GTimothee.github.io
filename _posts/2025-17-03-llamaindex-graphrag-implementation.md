---
layout: post
title: graphRAG implementation in Llama Index
date: 2025-17-03 00:00:00
description: The goal is to understand the implementation of graphRAG by llamaindex.
tags: rag 
categories: rag
---

// work in progress

## graph building from unstructured text

1. docs = [Document(text=sample['text']) for sample in docs]
2. PropertyGraphIndex.from_documents

### "from_documents" inner workings: 
1. Convert your docs to chunks (=Nodes) (See documentation on Documents and Nodes here https://docs.llamaindex.ai/en/stable/module_guides/loading/documents_and_nodes/#documents-nodes) using "transformations" (passed as argument or DEFAULTs TO Settings.transformations.
2. create the instance form the nodes (means you could have parse your documents yourself and create the instance yourself with the constructor)

### Constructor call
arguments:
- kg_extractors:A list of transformations to apply to the nodes to extract triplets. Defaults to [SimpleLLMPathExtractor(llm=llm), ImplicitEdgeExtractor()]
- default is to embed_kg_nodes (True)
- show_progress for progress bar

build_index_from_nodes 
  add nodes to docstore
  _build_index_from_nodes: an abstract class implemented in propertygraphindex
  returns index_struct

### _build_index_from_nodes implementation in propertygraphindex



## querying

1. create query engine from the index
2. just call the query engine with the query

## Implementation tips

- use Settings to define llm and embedding project wide
