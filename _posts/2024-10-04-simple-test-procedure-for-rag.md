---
layout: post
title: Simple test procedure for RAG in 5 minutes
date: 2024-10-04 00:00:00
description: A simple test procedure for RAG without use of external libraries
tags: llm rag evaluation procedure
categories: rag
---

# Simple test procedure for your Q/A RAG without external libraries in 5 minutes

In this blog post I show you how I build my simple evaluation procedure for a basic Q/A RAG system without use of external library or anything fancy. I also introduce two metrics I usually use to evaluate RAG systems, which are in my opinion simple, quite robust, and easily interpretable: completeness and conciseness.  

## Q/A RAG ?
By Q/A RAG I mean that each question of the test set is associated to (and can be answered with) one document chunk. This evaluation procedure is for Q/A RAG only because of course if you are looking for an answer for which you must gather data from multiple chunks, this procedure would not evaluate that. Still, I believe it is a good starting point when evaluating your RAG, as if you cannot reliably find one document chunk, how could you find multiple target document chunks? You could probably start with this procedure and then add another procedure for more complex use cases.

## Dataset generation
The dataset generation part is pretty simple. Go over each document chunk of your database and generate a question/answer pair for each chunk. Keep track of the chunk used for generation so that we can evaluate the RAG performance (i.e. output a triple for each chunk).

In the following of the post I use only a test set, but of course you can split it into a validation set and a test set, ensuring that the proportion of each source document is the same in each set. 

## Procedure's pseudocode
```
for each document 
	for each chunk in the document
		generate a (question, answer) pair
		rename answer as ground truth
		save the (question, ground truth, context) triple
	run your system on the question
	gather the answer and the documents used as context
	compute performance metrics of the whole system by comparing answer and ground truth
	compute rag performance by saving the rank of the target context in the contexts list that have been retrieved by the system. If the target context is not there, rank is set to None
	Save everything
```

You get a csv file with all the data, you can now compute statistics like completeness, conciseness, ranking distribution, %match, %misses. A nice to have is to compute statistics per document (add a column to the csv file with the index or the name of the document. 

Now you have a pretty good idea of how your rag app performs. Now every time you want to add a document to the knowledge base, add it to the test and run the test. You will know how much the addition of the new chunks in the database interferes with the existing documents, and what is the performance of your rag system on your new document.


