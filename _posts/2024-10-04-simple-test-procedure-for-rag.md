---
layout: post
title: Simple test procedure for RAG in 5 minutes
date: 2024-10-04 00:00:00
description: A simple test procedure for RAG without use of external libraries
tags: llm rag evaluation procedure
categories: rag
---

# Simple test procedure for your Q/A RAG without external libraries in 5 minutes

Q/A because of course if you are looking for an answer for which you must gather data from multiple chunks, this procedure would not evaluate that. Still, this procedure could probably be adapted to that use case (answering based on multiple chunks).

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

You get a csv file with all the data, you can now compute statistics like completeness, conciseness, ranking distribution, %match, %misses. A nice to have is to compute statistics per document (add a column to the csv file with the index or the name of the document. 

Now you have a pretty good idea of how your rag app performs. Now every time you want to add a document to the knowledge base, add it to the test and run the test. You will know how much the addition of the new chunks in the database interferes with the existing documents, and what is the performance of your rag system on your new document.

of course you can also define a validation set and a test set
