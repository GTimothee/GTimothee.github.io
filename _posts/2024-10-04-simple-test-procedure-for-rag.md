---
layout: post
title: Simple test procedure for RAG in 5 minutes
date: 2024-10-04 00:00:00
description: A simple test procedure for RAG without use of external libraries
tags: llm rag evaluation procedure
categories: rag
---

# Simple test procedure for your RAG without external libraries in 5 minutes

In this blog post I show you how I build my simple evaluation procedure for a basic Q/A RAG system without use of external library or anything fancy. All you need is... an existing RAG system and to have access to an LLM. I also introduce two metrics I usually use to evaluate RAG systems, which are simple, quite robust, and easily interpretable in my opinion: completeness and conciseness.  

## A Q/A evaluation
By Q/A evaluation I mean that each question of the test set is associated to (and can be answered with) one document chunk. Consequently, this evaluation procedure is for Q/A RAG only; Indeed, if you are looking for an answer for which you must gather data from multiple chunks, this procedure would not evaluate that. Still, I believe it is a good starting point when evaluating your RAG, as if you cannot reliably find one document chunk, how could you find multiple target document chunks? You could probably start with this procedure and then add another procedure for more complex use cases.

## Dataset generation
The dataset generation part is pretty simple. Go over each document chunk of your database and generate a question/answer pair for each chunk with the help of an LLM. Keep track of the chunk used for generation so that we can evaluate the RAG performance later (i.e. output a triple for each chunk).

In the following of the post I only use a test set, but of course you can split it into a validation set and a test set, ensuring that the proportion of each source document is approximately the same in each set. 

## Procedure's pseudocode
Without further delay, let's have a look at the full test procedure:
```
for each document
	for each chunk in the document
		# 1- generate the Q/A pair
		generate a (question, answer) pair
		rename answer to 'ground_truth'

		# 2- retrieve the answer from your system
		run your RAG system on the question
		gather the answer and the list of chunks used as context

		# 3- evaluate
		compute your performance metrics of the whole system by comparing the answer and the ground_truth
		compute your rag performance by saving the rank of the target context in the contexts list that has been retrieved by your system. If the target context is not there, the rank is set to None or something equivalent.

		Save everything as a line in a csv file
```

That's it. You get a csv file as output with all the data you need, you can now compute statistics like the completeness, conciseness and ranking distributions, the %match, %misses. A nice to have is to compute statistics per document (add a column to the csv file with the index or the name of the document. 

Now you have a pretty good idea of how your rag app performs. Now every time you want to add a document to the knowledge base, add it to the test and run the test. You will know how much the addition of the new chunks in the database interferes with the existing documents, and what is the performance of your rag system on your new document.


