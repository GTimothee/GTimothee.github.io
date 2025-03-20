---
layout: post
title: graphRAG implementation in Llama Index
date: 2025-03-17 00:00:00
description: The goal is to understand the implementation of graphRAG by llamaindex.
tags: rag 
categories: rag
---

# High level result of investigation

- Each chunk is a llama index Node
- For each chunk, we extract nodes and relationships and add them to the chunk (Node) metadata
- Extractors are documented here: https://docs.llamaindex.ai/en/stable/module_guides/indexing/lpg_index_guide/#default-simplellmpathextractor
- in LlamaIndex, we can combine several node retrieval methods at once
  - If no sub-retrievers are provided, the defaults are LLMSynonymRetriever and VectorContextRetriever (if embeddings are enabled)
- default store (in memory ) does not support embeddings, you need to use one of Neo4jPropertyGraphStore, TiDBPropertyGraphStore, FalkorDBPropertyGraphStore

base store: SimplePropertyGraphStore
- EntityNode, Relation are nodes and relationships extracted from chunk
- EntityNode are linked to their source chunk modeled as TextNode using Relation with label HAS_SOURCE

interesting methods:
- graph_store.get_rel_map([entity_node], depth=2): gets triples up to a certain depth
- graph_store.get_llama_nodes(['id1']): gets the original text nodes
- graph_store.structured_query("<cypher query>") - runs a cypher query (assuming the graph store supports it)

overview of useful extractors:
- SimpleLLMPathExtractor: simple extractor of `subject,predicate,object` triples, with a max_paths_per_chunk argument
- DynamicLLMPathExtractor: like neo4j approach, using node types

All retrievers currently include: 
- LLMSynonymRetriever: retrieve based on LLM generated keywords/synonyms
  - a prompt to extract synonyms
  - fetch matching kg nodes (not llama nodes) and apply get_rel_map on them (gets triples up to a certain depth) => get nodes and neighborhood in form of triplets, then convert triplets to NodeWithScore
- VectorContextRetriever: retrieve based on embedded graph nodes
- TextToCypherRetriever: ask the LLM to generate cypher based on the schema of the property graph
  - NOTE: Since the SimplePropertyGraphStore is not actually a graph database, it does not support cypher queries.
- CypherTemplateRetriever: use a cypher template with params inferred by the LLM
  - This is a more constrained version of the TextToCypherRetriever. Rather than letting the LLM have free-range of generating any cypher statement, we can instead provide a cypher template and have the LLM fill in the blanks.
- CustomPGRetriever: easy to subclass and implement custom retrieval logic

Define retrievers like this: 
```
sub_retrievers = [
    VectorContextRetriever(index.property_graph_store, ...),
    LLMSynonymRetriever(index.property_graph_store, ...),
]

retriever = PGRetriever(sub_retrievers=sub_retrievers)
```

Source: https://docs.llamaindex.ai/en/stable/module_guides/indexing/lpg_index_guide/

# Implementation break down used for investigation

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
  - add nodes to docstore
  - _build_index_from_nodes: an abstract class implemented in propertygraphindex
  - returns index_struct

### _build_index_from_nodes implementation in propertygraphindex

when talking about "llama nodes" they talk about the TextNodes representing chunks


self._insert_nodes(nodes or [])
  - applies kg extractors on nodes
  - builds two lists : one with all nodes and one with all relationships
  - filters pure node duplciates between our list of all nodes and the list of nodes already in the store (filter applied on "node.id")
  - filter out duplicate llama nodes 
  - if _embed_kg_nodes:
    - embed nodes
    - embed llama nodes
  - insert lalma nodes in property graph
  - insert nodes in property graph
  - insert relationships in property graphs


llama nodes are ChunkNode(s) from Llama-index

## querying

1. create query engine from the index
  - possible response modes: https://docs.llamaindex.ai/en/stable/module_guides/deploying/query_engine/response_modes/
  - if you want to tune it, construct it yourself instead of using the as_query_engine method as it lacks parameters
2. just call the query engine with the query
  1. creates a query bundle from the query (see below)
  2. calls the _query implementation of the query engine you chose

Implementation of _query for the default RetrieverQueryEngine:
```
nodes = self.retrieve(query_bundle)
response = self._response_synthesizer.synthesize(
    query=query_bundle,
    nodes=nodes,
)
```

The retriever just applies the retrievers that have been registered, and somehow (could not find where it is called but found the method) it also retrieves the source nodes (chunk nodes) for each triplet found. As a result you get the triplets found AND the source texts in your retrieved nodes. I do see it in my code, too.
  
Query bundle: 
  Can embed strings and images (give the image_path parameter)
  
  query_str (str): the original user-specified query string.
      This is currently used by all non embedding-based queries.
  custom_embedding_strs (list[str]): list of strings used for embedding the query.
      This is currently used by all embedding-based queries.
  embedding (list[float]): the stored embedding for the query.


## Implementation tips

- use Settings to define llm and embedding project wide
