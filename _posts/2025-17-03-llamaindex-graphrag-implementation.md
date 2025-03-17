---
layout: post
title: graphRAG implementation in Llama Index
date: 2025-17-03 00:00:00
description: The goal is to understand the implementation of graphRAG by llamaindex.
tags: llm rag 
categories: rag
---

// work in progress

## graph building from unstructured text

1. Convert your chunks to Document
2. select an extractor SimpleLLMPathExtractor
3. create a PropertyGraphIndex using the extractor on the list of documents

## querying

1. create query engine from the index
2. just call the query engine with the query

## Implementation tips

- use Settings to define llm and embedding project wide
