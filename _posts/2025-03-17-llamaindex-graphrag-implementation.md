---
layout: post
title: graphRAG implementation in Llama Index
date: 2025-03-17 00:00:00
description: A graphRAG guide with llama index
tags: rag 
categories: rag
---

## 1. Loading the data

- Input: files or data in memory
- Methods:
  - Loading from files ? use SimpleDirectoryReader or FlatFileReader to load the files
  - Loading from memory (.e.g. loading from a dataframe) ? convert data to Documents (```docs = [Document(text=sample['text']) for sample in docs]```)
- Output: "Nodes" representing documents
- Ref: https://docs.llamaindex.ai/en/v0.10.19/module_guides/loading/node_parsers/modules.html

## 2. Initial preprocessing

### Transformations

- Input: Nodes
- Transformations can be used to perform preprocessing like chunking or metadata extraction for example.
- Output: Nodes
- Ref: https://docs.llamaindex.ai/en/stable/module_guides/loading/ingestion_pipeline/transformations/

### Chunking

A type of transformation. Utilities for splitting are called splitters.

- Loading from files ? use SimpleFileNodeParser on the docs you retrieved with FlatFileReader. It will use the best splitter regarding the extension of the file you loaded.
- Loading from memory ? choose the best splitter depending on your data
  - simple text: TokenTextSplitter, SentenceSplitter (split on sentences, with overlap), SentenceWindowNodeParser (split on sentences, with sentence-level overlap put in metadata), MarkdownNodeParser, SemanticSplitterNodeParser (by Greg Kamradt, requires an embedding model)
  - special format: JSONNodeParser, HTMLNodeParser, CodeSplitter
  - meh: HierarchicalNodeParser (advanced, will see later)
 
ref: https://docs.llamaindex.ai/en/v0.10.19/module_guides/loading/node_parsers/modules.html

### Metadata extraction 

Adds metadata to the chunks.

- Allows:
  - better context for the llm to generate an answer (""chunk dreaming" - each individual chunk can have more "holistic" details, leading to higher answer quality given retrieved results.")
  - using metadata to improve the search ("disambiguate similar-looking passages.")
- Example extractors:
  - SummaryExtractor: extracts summaries, not only within the current text, but also within adjacent texts.
  - QuestionsAnsweredExtractor: creates hypothetical question embeddings relevant to the document chunk
  - TitleExtractor: in case the file name is not perfect, which happens a lot, you may want to extrapolate a file name representative of the content
  - KeywordExtractor: "extracts entities (i.e. names of places, people, things) mentioned in the content of each Node"
    - default extraction model : https://huggingface.co/tomaarsen/span-marker-mbert-base-multinerd
    - will use these keywords during the search
    - See example here: https://docs.llamaindex.ai/en/stable/examples/metadata_extraction/EntityExtractionClimate/
  - BaseExtractor: to build your own
- third-party extractors: allows to define what you are looking for as an object. You can therefore reproduce all the above extractors and more!
  - MarvinMetadataExtractor
  - PydanticProgramExtractor
- Ref: https://docs.llamaindex.ai/en/stable/module_guides/indexing/metadata_extraction/
- Ref: https://docs.llamaindex.ai/en/stable/examples/metadata_extraction/...

### Example implementation

```python
from llama_index.core.extractors import (
    SummaryExtractor,
    TitleExtractor,
    KeywordExtractor,
    QuestionsAnsweredExtractor
)
from llama_index.extractors.entity import EntityExtractor
from llama_index.core.node_parser import TokenTextSplitter
from llama_index.core import SimpleDirectoryReader
from llama_index.core.ingestion import IngestionPipeline

# chunking
text_splitter = TokenTextSplitter(
    separator=" ", chunk_size=512, chunk_overlap=128
)

# metadata extraction
extractors = [
    TitleExtractor(nodes=5, llm=llm),
    QuestionsAnsweredExtractor(questions=3, llm=llm),
    EntityExtractor(prediction_threshold=0.5),
    SummaryExtractor(summaries=["prev", "self"], llm=llm),
    KeywordExtractor(keywords=10, llm=llm)
]

# concat transformations
transformations = [text_splitter] + extractors

uber_docs = SimpleDirectoryReader(input_files=["data/10k-132.pdf"]).load_data()
pipeline = IngestionPipeline(transformations=transformations, in_place=False, show_progress=True)
uber_nodes = pipeline.run(documents=uber_docs)
uber_nodes[1].metadata
```

Example output:
//TODO

Note: You can also define the transformations globally using the 'Settings' object.

## 3. Building an index

Definition of an index from the docs:
- "With your data loaded, you now have a list of Document objects (or a list of Nodes). It's time to build an Index over these objects so you can start querying them."
- "In LlamaIndex terms, an Index is a data structure composed of Document objects, designed to enable querying by an LLM. Your Index is designed to be complementary to your querying strategy."
- The index include a "store" which is where the data will be saved. By default it is in memory but you can also store on disk or in a database.
- We then query the index.

Indexes: 
- VectorStoreIndex: allows for vector search, computing embeddings for each chunk
- PropertyGraphIndex
  - collection of labelled nodes linked together by relationships
  - labelled nodes have properties
  - to build property graph you define a list of kg_extractors that will be applied on each chunk
  - extracted data are added as metadata to the input chunk (called "llama-index" nodes, they are just the input text chunks you feed)

Note on KnowledgeGraphIndex:
- You may also see KnowledgeGraphIndex in the docs, it is the previous version of PropertyGraphIndex.
- "expands our knowledge graph capabilities to be more flexible, extendible, and robust." (More info in the following [blog post](https://www.llamaindex.ai/blog/introducing-the-property-graph-index-a-powerful-new-way-to-build-knowledge-graphs-with-llms))
- New features introduced with PropertyGraphIndex:
  - Assign labels and properties to nodes and relationships
  - Represent text nodes as vector embeddings
  - Perform both vector and symbolic retrieval

Example with propertygraphindex:
```python
index = PropertyGraphIndex.from_documents(
    nodes,
    kg_extractors=[extractor1, extractor2, ...],
)
```

Example with knowledgegraphindex:
```python
from llama_index.core import KnowledgeGraphIndex

index = KnowledgeGraphIndex.from_documents(
    nodes,
    max_triplets_per_chunk=2
)
```

```.from_documents``` inner workings: 
1. Convert your docs to chunks (=Nodes) using "transformations" (passed as argument or defaults to Settings.transformations)
2. create the instance form the nodes (means you could have parse your documents yourself and create the instance yourself with the constructor)
- default is to embed_kg_nodes (True)
- show_progress for progress bar
- inside constructor, main method called is:
  - self._insert_nodes(nodes or [])
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

Save and load from disk:
```
# save
index.storage_context.persist("./storage")

# load
storage_context = StorageContext.from_defaults(persist_dir="./storage")
index = load_index_from_storage(storage_context)
```

kg_extractors:
- SimpleLLMPathExtractor: extracts single-hop paths of the form (entity1, relation, entity2). Prompt can be customized.
  - warning: Tune the max_paths_per_chunk argument
- ImplicitPathExtractor: not very useful in real case, it is when you already have the relationships in the data (so it reads the relationships rather than extracting them). No DL model is used.
- DynamicLLMPathExtractor: you can give a schema of node and relationship types that the llm will try to follow. In case no type fits its needs, it will add its own types.
- SchemaLLMPathExtractor: Follows a strict schema without possibility of creating new types.

Stores:
- Default store: SimplePropertyGraphStore
  - EntityNode, Relation are nodes and relationships extracted from chunk
  - EntityNode are linked to their source chunk modeled as TextNode using Relation with label HAS_SOURCE
- Supporting embeddings: Neo4jPropertyGraphStore, TiDBPropertyGraphStore, FalkorDBPropertyGraphStore

- Ref: https://docs.llamaindex.ai/en/stable/understanding/indexing/indexing/
- Ref (property graph): https://docs.llamaindex.ai/en/stable/module_guides/indexing/lpg_index_guide/

## 4. Querying the graph

- Build a query engine from the index, passing in the retrievers you want.
  - possible response modes: https://docs.llamaindex.ai/en/stable/module_guides/deploying/query_engine/response_modes/
  - if you want to tune it, construct it yourself instead of using the as_query_engine method as it lacks parameters
- Default retrievers: LLMSynonymRetriever and VectorContextRetriever (if embeddings are enabled).
  - To enable embeddings you must use a store that supports them. Default in memory store does not support embeddings.
- Using the query engine:
  1. it creates a query bundle from the query (see below)
  2. it calls the _query implementation of the query engine you chose. Basically this method just fetches nodes and pass them to the llm to generate a response.

query bundle:
```
Query bundle: 
  Can embed strings and images (give the image_path parameter)
  
  query_str (str): the original user-specified query string.
      This is currently used by all non embedding-based queries.
  custom_embedding_strs (list[str]): list of strings used for embedding the query.
      This is currently used by all embedding-based queries.
  embedding (list[float]): the stored embedding for the query.
```

base retrievers: 
- LLMSynonymRetriever: retrieve based on LLM generated keywords/synonyms
  - a prompt to extract synonyms
  - fetch matching kg nodes (not llama nodes) and apply get_rel_map on them (gets triples up to a certain depth) => get nodes and neighborhood in form of triplets, then convert triplets to NodeWithScore
  - I am wondering if by fetching the neighborhood they retrieve the text nodes (input chunks) as the same time, as I can see the text nodes being retrieved also, but I could not find somewhere else from which they could be retrieved. 
- VectorContextRetriever: retrieve based on embedded graph nodes
- CustomPGRetriever: easy to subclass and implement custom retrieval logic

cypher-based retrievers
- TextToCypherRetriever: ask the LLM to generate cypher based on the schema of the property graph
  - NOTE: Since the SimplePropertyGraphStore is not actually a graph database, it does not support cypher queries.
- CypherTemplateRetriever: use a cypher template with params inferred by the LLM
  - This is a more constrained version of the TextToCypherRetriever. Rather than letting the LLM have free-range of generating any cypher statement, we can instead provide a cypher template and have the LLM fill in the blanks.

Example of passing the retrievers list:
```python
# create a retriever
retriever = index.as_retriever(sub_retrievers=[retriever1, retriever2, ...])

# create a query engine
query_engine = index.as_query_engine(
    sub_retrievers=[retriever1, retriever2, ...]
)
```

Simple example for creating a query engine from an index:
```
query_engine = index.as_query_engine(
    include_text=True,
    response_mode="tree_summarize",
    embedding_mode="hybrid",
    similarity_top_k=5,
)
response = query_engine.query(
    "Tell me more about what the author worked on at Interleaf",
)
```

interesting methods on graph stores:
- graph_store.get_rel_map([entity_node], depth=2): gets triples up to a certain depth
- graph_store.get_llama_nodes(['id1']): gets the original text nodes
- graph_store.structured_query("<cypher query>") - runs a cypher query (assuming the graph store supports it)

