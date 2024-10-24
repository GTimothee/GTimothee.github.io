---
layout: post
title: Test procedure and metrics for RAG in 5 minutes
date: 2024-10-04 00:00:00
description: Simple test procedure and metrics for your RAG in 5 minutes
tags: llm rag evaluation procedure
categories: rag
---

> In this blog post I show you how I build my simple evaluation procedure for a basic Q/A RAG system without use of external library or anything fancy. All you need is… an existing RAG system and to have access to an LLM. I also introduce two metrics I usually use to evaluate RAG systems, which are simple, quite robust, and easily interpretable in my opinion: completeness and conciseness.

## Motivation
*When you work in R&D like me, you have limited time to build a PoC.* When working on RAG systems and AI agents in general, it is already a pain to select your tools, learn how to use them and build something robust. I will not explain you why you also need to evaluate the system, as it is pretty obvious.

*So you are faced with the following problem*: How do I focus on what I build, while also evaluating my RAG system fast, and in such a way that I can rapidly debug/improve the system. And the speed and interpretability criteria are key, in my opinion. In this blog post I show you how to build such system very fast, for rapid prototyping, without having to benchmark, test or dive into complicated frameworks with lots of metrics and settings. My solution is not fancy or complicated, but that is the whole point of it. If your PoC gets validated, you will have plenty of time to select a good RAG testing framework like RAGAS, spend time on elaborating advanced datasets, setup automated testing, etc. (I may dive into these advanced topics in future posts)

## Step 1 — Dataset generation

The dataset generation part is pretty simple. Go over each document chunk of your database and generate a question/answer pair for each chunk with the help of an LLM. Keep track of the chunk used for generation so that we can evaluate the RAG performance later. In other words, output a triple (question, answer, chunk) for each chunk.

In the following of the post I only use a test set, but of course you can split it into a validation set and a test set, ensuring that the proportion of each source document is approximately the same in each set.

## Step 2 — Procedure’s pseudo code

Without further delay, let's have a look at the full test procedure:
```python
for each document
  for each chunk in the document
    # 1- generate the Q/A pair
    generate a (question, answer) pair
    rename answer to 'ground_truth'
    
    # 2- retrieve the answer from your system
    run your RAG system on the question
    gather the answer and the list of chunks retrieved and used as context
    
    # 3- evaluate
    compute your performance metrics of the whole system by comparing the answer and the ground_truth
    compute your rag performance by saving the rank of the target context in the contexts list that has been retrieved by your system. If the target context is not there, the rank is set to None or something equivalent.
    Save everything as a line in a csv file
```

That’s it. You get a csv file as output with all the data you need, you can now compute statistics like the completeness, conciseness and ranking distributions, the %match, %misses. A nice to have is to compute statistics per document (add a column to the csv file with the index or the name of the document).

## Completeness and conciseness

Here is an example prompt for computing completeness and conciseness:

```python
prompt = """You are a grading software for a teacher.
You will be given a JSON with a 'question' (for context), an 'answer' and a 'ground truth answer'.
Your task is to grade the 'answer' by returning two float scores from 0 to 1: 'completeness' and 'conciseness'. 

For completeness:
- A score of 1 means that all the information in the 'ground truth answer' can be found in the 'answer'. No matter if the answer contains more information than expected.
- A score of 0 means that the 'answer' contains no information present in the 'ground truth answer'.

For conciseness: It is the percentage of the answer that is information present in the ground truth.

Provide your answer as a JSON with four keys: 
- 'completeness' is the completeness score (float)
- 'conciseness' is the conciseness score (float)

Only return the JSON, with no additional text.

Here is the JSON data to evaluate: 
{input}

Answer:"""```

Additional comments:
- The input is a python dict with keys question, answer and ground_truth_answer. You can pass it as a string using json.dumps
- Completeness evaluates whether or not the answer answers the question, while conciseness answers the question “how much of the answer is actually relevant”. If the completeness is low, then the system had trouble retrieving the relevant documents. If the conciseness is low, and the completeness is high, you are retrieving too much documents. So try to focus on improving the rank of the target document in the set of retrieved documents so that you can reduce the number of documents retrieved and reduce the noise. You could also add a reranker, which is probably a good idea in any RAG system.
- This will return JSON, that you can easily parse with a JsonOutputParser from langchain
- I tried adding some other keys like ‘comments’ or ‘reasons’ to leverage the idea of chain of thoughts, but it did not provide any useful information
- I use floats here, but it may be that using integers from 1 to 10 instead would be more efficient or precise.

## Disclaimer: It is for Q/A evaluation

By Q/A evaluation I mean that each question of the test set is associated to (and can be answered with) one document chunk. Consequently, this evaluation procedure is for Q/A RAG only; Indeed, if you are looking for an answer for which you must gather data from multiple chunks, this procedure would not evaluate that. Still, I believe it is a good starting point when evaluating your RAG, as if you cannot reliably find one document chunk, how could you find multiple target document chunks? You could probably start with this procedure and then add another procedure for more complex use cases.

## Conclusion

Now you have a pretty good idea of how your RAG app performs. Now every time you want to add a document to the knowledge base, add it to the dataset and run the test. You will know how much the addition of the new chunks in the database interferes with the existing documents, and what is the performance of your RAG system on your new document.

Medium link:
https://medium.com/@timothee.guedon/simple-test-procedure-and-metrics-for-your-rag-in-5-minutes-a86b329a5f7a
