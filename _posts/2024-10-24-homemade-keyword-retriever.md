---
layout: post
title: How to write your own keyword retriever in 5 minutes
date: 2024-10-24 00:00:00
description: Tutorial on how to write your own keyword retriever using the whoosh library
tags: keyword-retriever rag whoosh python
categories: rag
---

> In this blog post I give you a tutorial on how to build a simple yet efficient keyword retriever for a RAG system, using the Whoosh library.

## Motivation
I found myself building a RAG system for question answering, and it is not as easy as you might think. 

The main problem comes when dealing with technical documentation. Implementing a similarity search-based RAG will not help you much. Firstly, the embedding model does not know the keywords, so it cannot embed them properly. Secondly, when a user asks a question, the text is rarely "similar" to the chunk that contains the answer, if that makes sense.

Although it does not solve all your problems, a first step towards improving your RAG in this setting is well known: you shall add a keyword retriever in addition to the similarity search-based retriever, as the both of them are complementary. 

This post will be useful for you in at least two ways: if you are curious about how a keyword retriever works in practice, and if, like me, you are using a database that does not have a keyword retriever included. Some might argue that I could have use the retriever from langchain but from what I understood it either loads all the documents in memory (not optimal) or needs elasticsearch as a backend, which my team and I did not want to do, as we weren't using it for anything else in the project at hand.

## Whoosh
> Before diving into the practical stuff, let me introduce Whoosh, which I will use to build the keyword retriever.

[Whoosh](https://github.com/mchaput/whoosh) is a popular, fast, pure Python search engine library designed for adding search functionality to applications. It’s often used for indexing and searching textual data within Python applications, and it’s popular in cases where a lightweight, easy-to-integrate search solution is required.

Key Features of Whoosh:
- Written in Pure Python: Whoosh is implemented entirely in Python, which makes it easy to install and run without external dependencies or a specific backend.
- Full-Text Search: It provides full-text search capabilities, including indexing and retrieving text-based data efficiently.
- Customizable: Whoosh is highly flexible and allows developers to customize search functionality with features like tokenizers, filters, and analyzers.
- Simple Integration: Whoosh is lightweight and can be embedded directly into Python applications without the need for external services, making it ideal for smaller-scale applications.
- Phrase and Boolean Search: Supports phrase searches, Boolean operators (AND, OR, NOT), and other advanced search features.

Important note about Whoosh: It is now unmaintained, so you may want to use ["Whoosh Reloaded"](https://github.com/Sygil-Dev/whoosh-reloaded) instead, a fork and continuation of the Whoosh project, which is actively maintained. The code I will give here works for Whoosh as well as for Whoosh Reloaded.

## Building the index

The first thing we need to do is to build a reverse index. Whoosh gives a clear definition in its documentation:

> A reverse index is "Basically a table listing every word in the corpus, and for each word, the list of documents in which it appears. It can be more complicated (the index can also list how many times the word appears in each document, the positions at which it appears, etc.) but that’s how it basically works." (Whoosh documentation)

To build a second retriever for my RAG system, I reuse the chunks from the Chroma database I built for a previous blog post. That way, I will have two complementary retievers, retrieving on the same chunks. I build a little generator as follows: 

```python
def database_iterator(database_dirpath: str, batch_size: int = 100):
    client = get_db(database_dirpath)
    offset = 0
    while offset < client._collection.count():
        print(f"offset={offset}")
        batch = client.get(offset=offset, limit=batch_size)
        offset += len(batch["documents"])
        yield batch
```

Find the full script [here](https://github.com/GTimothee/RAG_experiments/blob/main/blogs/keyword_retriever/chromadb_iterator.py)

The next step consists in building a Schema. 

> "Whoosh requires that you specify the fields of the index before you begin indexing. The Schema associates field names with metadata about the field, such as the format of the postings and whether the contents of the field are stored in the index." (Whoosh documentation)

The schema represents a document in the index. We will use *TEXT* and *STORED* fields, for each document. The *TEXT* field specifies the text to be indexed. The document and its terms will be added to the reverse index. The *STORED* fields are the fields that will be retrieved, if the document is retrieved. 

Here is my very simple schema:

```python
from whoosh.fields import Schema, TEXT, STORED
from whoosh.analysis import StemmingAnalyzer

stem_ana = StemmingAnalyzer()
schema = Schema(
    content=TEXT(analyzer=stem_ana),
    text_content=STORED,
    metadata=STORED,
)
```

As you can see, I index the content of the document, and I store the content and the metadata. I also added a custom analyzer. The goal of the StemmingAnalyzer is to store the stems of the words, instead of the words themselves, which allows to generalize better.

More about analyzers: "An analyzer is a function [...] that takes a unicode string and returns a generator of tokens". (Whoosh documentation)

More about the StemmingAnalyzer: It is a "pre-packaged analyzer that combines a tokenizer, lower-case filter, optional stop filter, and stem filter" (Whoosh documentation)

We can now create an index from our schema...

```python
from whoosh.index import create_in

index_dirpath = # where you want to store your index
ix = create_in(str(index_dirpath), schema)
writer = ix.writer()
```

...And load our chunks into the index:

```python
total = 0
for batch in database_iterator('data/chroma_db_1000'):
    batch_size = len(batch["documents"])
    total += batch_size 

    for i in range(batch_size):
        writer.add_document(
            text_content=batch["documents"][i],
            content=batch["documents"][i],
            metadata=batch["metadatas"][i]
        )

writer.commit()
```

Find the full script [here](https://github.com/GTimothee/RAG_experiments/blob/main/blogs/keyword_retriever/build_index.py)

## Querying the index

Now that we have the index, let us write the retriever itself. Given a query, the retrieving process is pretty simple: 
1. preprocess the query: we want to apply the same preprocessing to the query that to the indexed documents, so that we can use the words stems for retrieval.
2. translate the query into Whoosh query language (the same way we would translate our string into SQL for a SQL database)
3. use that query against the index, leveraging a dedicated search algorithm

To preprocess the query we remove stopwords using the *nltk* package, and we stem the words using the ```whoosh.lang.porter.stem``` function. Let us first download the stopwords (they will be downloaded only once).

```python 
import nltk 
from nltk.corpus import stopwords

stop_words = set(stopwords.words("english"))
stop_words.add("?")
```

For step 2, it is as simple as initializing a query parser. I use a simple QueryParser, but if you have several fields you want to look into at retrieval time, you can use the MultifieldParser instead. By default, the parser matches a document that contains all the terms specified in the query (using the AND operator). For example, "physically based rendering" will be parser "physically AND based AND rendering". I change this behaviour by using the OR operator instead, and the "qparser.OrGroup.factory(0.9)" statement allows to give more weight to a document if it contains the words we are looking for multiple times.

```python
from whoosh import qparser
from whoosh.qparser import QueryParser
from whoosh.index import open_dir

parser = QueryParser(
    "content",
    ix.schema,
    group=qparser.OrGroup.factory(0.9),
)
ix = open_dir(index_dirpath)
```

Putting everything together, I get this preprocessing function:

```python
from nltk.tokenize import word_tokenize
from whoosh.lang.porter import stem

def _process_query(query):
    # tokenize 
    word_tokens = word_tokenize(query)
    print(f"tokenized query: {word_tokens}")

    # stem and filter out stop words
    filtered_query = " ".join(
        [
            stem(word)
            for word in word_tokens
            if word.lower() not in stop_words
        ]
    )
    print(f"stemmed-filtered_query: {filtered_query}")

    parsed_query = parser.parse(filtered_query)
    print(f"parsed_query: {parsed_query}")
    return parsed_query
```

We can now use the ix.searcher method to search for the most relevant documents:

```python
from whoosh import scoring
from langchain_core.documents import Document

k = # integer representing the number of documents you want to retrieve
query = # a string

with ix.searcher(weighting=scoring.BM25F()) as searcher:
    formatted_query = _process_query(query)

    results = searcher.search(formatted_query, limit=k)
    return [
        Document(
            metadata=result["metadata"], 
            page_content=result["text_content"]
        )
        for result in results
    ]
```

As you can see, we retrieve the STORED fields "text_content" and "metadata" from the schema.

Find the full script [here](https://github.com/GTimothee/RAG_experiments/blob/main/library/keyword_retriever.py)

## The search algorithm

As you may notice, I am using the BM25F algorithm (the default in Whoosh) to perform the search. From what I saw, only the 'frequency-based', 'TF-IDF' and 'BM25F' algorithms are supported in Whoosh. Whoosh-reloaded supports two additional algorithms: PL2 and DFREE. The BM25 algorithm is considered better than TF-IDF and is pretty robust. PL2 and DFREE can be better depending on the dataset at hand, but do not guarantee better results. 

The BM25F algorithm is a variant of the BM25 algorithm that enables searching over multiple fields at the same time, which suits Whoosh well as we can define a schema with multiple fields to look into. 

## Evaluation

On the first 200 samples of the huggingface documentation dataset, the traditional RAG with similarity search gave: 73% mean completeness and 52% mean conciseness. Here is the associated disrtibution of the target document rank in the results:
```
0.0    100
1.0     19
3.0     11
2.0     11
```
It means that for half of the samples, we were able to find the target chunk with rank 0.

I tried the same pipeline, replacing the similarity search-based retriever by the keyword retriever we built. I got 80% mean completeness and 56% mean conciseness. Here is the associated disrtibution of the target document rank in the results:
```
0.0    132
1.0     19
3.0      8
2.0      4
```

We can see that the keyword retiever actually performs better than the similarity search based retriever ! Depending on the dataset, it may not always be the case, but the point is that a keyword retriever actually performs pretty good. In a future blog post, we will combine both retrievers and add a reranker on top to get the best of both worlds and hopefully get better results than using only one of the retrievers.

If you want the details and the code of the evaluation framework I used, checkout the dedicated [blog post](https://gtimothee.github.io/blog/2024/simple-test-procedure-for-rag/).

## Conclusion

We saw how to build a robust yet simple keyword retriever using Whoosh.

Medium link:
