---
layout: post
title: How to write your own keyword retriever in 5 minutes
date: 2024-10-24 00:00:00
description: Tutorial on how to write your own keyword retriever using the whoosh library
tags: keyword-retriever rag whoosh python
categories: rag
---

> In this blog post I show you how I build a simple yet efficient keyword retriever for a RAG system, using the whoosh library.

## Motivation
I found myself building a RAG system for question answering, and it is not as easy as you might think. 

The main problem comes when dealing with technical documentation. Implementing a similarity search-based RAG will not help you much. Firstly, the embedding model does not know the keywords, so it cannot embed them properly. Secondly, when a user asks a question, the text is rarely "similar" to the chunk that contains the answer, if that makes sense.

Although it does not solve all your problems, a first step towards improving your RAG in this setting is well known: you shall add a keyword retriever in addition to the similarity search-based retriever, as the both of them are complementary. 

This post will be useful for you in at least two ways: if you are curious about how a keyword retriever works in practice, and if, like me, you are using a database that does not have a keyword retriever included. Some might argue that I could have use the retriever from langchain but from what I understood it either loads all the documents in memory (not optimal) or needs elasticsearch as a backend, which my team and I did not want to do, as we weren't using it for anything else in the project at hand.

## Whoosh

[Whoosh](https://github.com/mchaput/whoosh) is a popular (580 stars, 72 forks), fast, pure Python search engine library designed for adding search functionality to applications. It’s often used for indexing and searching textual data within Python applications, and it’s popular in cases where a lightweight, easy-to-integrate search solution is required.

Key Features of Whoosh:
- Written in Pure Python: Whoosh is implemented entirely in Python, which makes it easy to install and run without external dependencies or a specific backend.
- Full-Text Search: It provides full-text search capabilities, including indexing and retrieving text-based data efficiently.
- Customizable: Whoosh is highly flexible and allows developers to customize search functionality with features like tokenizers, filters, and analyzers.
- Simple Integration: Whoosh is lightweight and can be embedded directly into Python applications without the need for external services, making it ideal for smaller-scale applications.
- Phrase and Boolean Search: Supports phrase searches, Boolean operators (AND, OR, NOT), and other advanced search features.

Important note about Whoosh: It is now unmaintained, so you may want to use ["Whoosh Reloaded"](https://github.com/Sygil-Dev/whoosh-reloaded) instead, a fork and continuation of the Whoosh project, which is actively maintained. The code I will give here works for Whoosh as well as for Whoosh Reloaded.

## Steps for building our retriever

// in progress

## Conclusion

// in progress

Medium link:
