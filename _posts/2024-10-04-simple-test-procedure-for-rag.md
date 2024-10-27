---
layout: post
title: Test procedure and metrics for RAG in 5 minutes
date: 2024-10-04 00:00:00
description: Simple test procedure and metrics for your RAG in 5 minutes
tags: llm rag evaluation procedure metrics
categories: rag
---

> In this blog post I show you how I build my simple evaluation procedure for a basic Q/A RAG system without use of external library or anything fancy. All you need is… an existing RAG system and to have access to an LLM. I also introduce two metrics I usually use to evaluate RAG systems, which are simple, quite robust, and easily interpretable in my opinion: completeness and conciseness.

## Motivation
*When you work in R&D like me, you have limited time to build a PoC.* When working on RAG systems and AI agents in general, it is already a pain to select your tools, learn how to use them and build something robust. I will not explain you why you also need to evaluate the system, as it is pretty obvious.

*So you are faced with the following problem*: How do I focus on what I build, while also evaluating my RAG system fast, and in such a way that I can rapidly debug/improve the system. And the speed and interpretability criteria are key, in my opinion. In this blog post I show you how to build such system very fast, for rapid prototyping, without having to benchmark, test or dive into complicated frameworks with lots of metrics and settings. My solution is not fancy or complicated, but that is the whole point of it. If your PoC gets validated, you will have plenty of time to select a good RAG testing framework like RAGAS, spend time on elaborating advanced datasets, setup automated testing, etc. (I may dive into these advanced topics in future posts)

## Step 1 — Dataset generation

The dataset generation part is pretty simple. Go over each document chunk of your database and generate a question/answer pair for each chunk with the help of an LLM. Keep track of the chunk used for generation so that we can evaluate the RAG performance later. In other words, output a triple (question, answer, chunk) for each chunk.

Here is a piece of code that does just that:

```python
SYSTEM_PROMPT = """You are an AI teacher, writing an exam out of course material.
Your task is to generate a (question, answer) pair from a given chunk from the course that is given to you.  
Return a JSON object with two keys:
- 'question': a question generated from the given chunk
- 'answer': the answer to the question
Just return the JSON, without any premamble or comment.

Chunk of the course material:
{chunk}
"""


class QAPair(BaseModel):
    question: str = Field(description="question generated from the given chunk")
    answer: str = Field(description="the answer to the question")


if __name__ == "__main__":
    assert os.path.isdir(
        args.output_dir
    ), f"Output directory not found: {args.output_dir}"
    assert os.path.isdir(args.chroma_dir), f"Chroma db not found: {args.chroma_dir}"

    load_dotenv()
    db = get_db(args.chroma_dir)
    llm = # your llm here
    parser = JsonOutputParser(pydantic_object=QAPair)
    prompt = PromptTemplate(
        template=SYSTEM_PROMPT,
        input_variables=["chunk"],
        partial_variables={"format_instructions": parser.get_format_instructions()},
    )
    chain = prompt | llm | parser

    data = db.get()

    if args.limit > 0:
        n_chunks = args.limit
        output_filename = f"qa_dataset_limit={n_chunks}.csv"
    else:
        n_chunks = len(data["documents"])
        output_filename = "qa_dataset.csv"

    dataset = {"question": [], "ground_truth_answer": [], "chunk_id": []}
    for i in tqdm(range(n_chunks)):
        chunk = data["documents"][i]
        output = chain.invoke({"chunk": chunk})
        dataset["question"].append(output["question"])
        dataset["ground_truth_answer"].append(output["answer"])
        dataset["chunk_id"].append(data["ids"][i])

    df = pd.DataFrame(dataset)
    df.to_csv(str(Path(args.output_dir, output_filename)), index=False)
```

Find the full code example [here](https://github.com/GTimothee/RAG_experiments/blob/main/test_procedure_for_rag/generate_qa_pairs.py).

In the following of the post I only use a test set, but of course you can split it into a validation set and a test set, ensuring that the proportion of each source document is approximately the same in each set.

## Step 2 — Procedure’s pseudo code

Without further delay, let's have a look at the full test procedure:
```python
for each document
  for each chunk in the document
    # 1- retrieve the answer from your system
    run your RAG system on the question
    gather the answer and the list of chunks retrieved and used as context
    
    # 2- evaluate
    compute your performance metrics of the whole system by comparing the answer and the ground_truth
    compute your rag performance by saving the rank of the target context in the contexts list that has been retrieved by your system. If the target context is not there, the rank is set to None or something equivalent.
    Save everything as a line in a csv file
```

That’s it. You get a csv file as output with all the data you need, you can now compute statistics like the completeness, conciseness and ranking distributions, the %match, %misses. A nice to have is to compute statistics per document (add a column to the csv file with the index or the name of the document).

Here is an interpretation of the first part of the peudocode, to generate answers from the dataset:

```python
rag_chain, retriever, db = get_rag_chain_eval(chroma_db_dirpath=path_to_your_db)
df = pd.read_csv(args.dataset_filepath)
outputs = {
    'answers': [], 'ranks': []
}

for row in tqdm(df.itertuples(), total=len(df), desc='Generating answers...'):
    documents = retriever.invoke(row.question)

    # generate answer
    output = rag_chain.invoke({
        "question": row.question,
        "context": '\n'.join([doc.page_content for doc in documents])
    })
    outputs['answers'].append(output)

    # compute rank of the target documents in the list of retrieved documents
    target_chunk = db.get(row.chunk_id)['documents'][0]
    rank = None
    for i, chunk in enumerate(documents):
        if chunk.page_content == target_chunk:
            rank = i
    outputs['ranks'].append(rank)

pd.DataFrame(outputs).to_csv(
    str(Path(args.output_dir, f"{Path(args.dataset_filepath).stem}" + "_answers.csv")), index=False)
```

You can find the full code [here](https://github.com/GTimothee/RAG_experiments/blob/main/test_procedure_for_rag/generate_answers.py)

Below is an interpretation of the second part of the pseudocode, to evaluate the answers. I use two metrics, completeness and conciseness to evaluate the RAG answers. 

Completeness evaluates whether or not the answer answers the question, while conciseness answers the question “how much of the answer is actually relevant”. If the completeness is low, then the system had trouble retrieving the relevant documents. If the conciseness is low, and the completeness is high, you are retrieving too much documents. So try to focus on improving the rank of the target document in the set of retrieved documents so that you can reduce the number of documents retrieved and reduce the noise. You could also add a reranker, which is probably a good idea in any RAG system.

Additional comments about the evaluation prompt:
- I tried adding some other keys like ‘comments’ or ‘reasons’ to leverage the idea of chain of thoughts, but it did not provide any useful information
- I use floats here, but it may be that using integers from 1 to 10 instead would be more efficient or precise.

```python
SYSTEM_PROMPT = """You are a top-tier grading software belonging to a school.
Your task is to give a grade to evaluate the answer goodness to a given question, given the ground truth answer.

You will be given a piece of data containing: 
- a 'question'
- an 'answer': the answer to the question from the student
- a 'ground truth answer': the expected answer to the question

Provide your answer as a JSON with two keys: 
- 'completeness': A float between 0 and 1. The percentage of the ground truth answer that is present in the student's answer. A score of 1 means that all the information in the 'ground truth answer' can be found in the 'answer'. No matter if the answer contains more information than expected. A score of 0 means that no information present in the 'ground truth answer' can be found in the 'answer'.
- 'conciseness': A float between 0 and 1. The percentage of the answer that is part of the ground truth. Conciseness measures how much of the answer is really useful.

Here is the data to evaluate: 
- 'question': {question}
- 'answer': {answer}
- 'ground truth answer': {ground_truth_answer}

Provide your answer as a JSON, with no additional text.
"""


class Evaluation(BaseModel):
    completeness: float = Field(description="A float between 0 and 1. The percentage of the ground truth answer that is present in the student's answer. A score of 1 means that all the information in the 'ground truth answer' can be found in the 'answer'. No matter if the answer contains more information than expected. A score of 0 means that no information present in the 'ground truth answer' can be found in the 'answer'.")
    conciseness: float = Field(description="A float between 0 and 1. The percentage of the answer that is part of the ground truth. Conciseness measures how much of the answer is really useful.")


if __name__ == "__main__":

    load_dotenv()

    df = pd.concat([
        pd.read_csv(args.dataset_filepath),
        pd.read_csv(args.answers_filepath)
    ], axis=1)

    llm = OpenAI(
        openai_api_base=os.getenv("OPENAI_BASE_URL"),
        openai_api_key=os.getenv("OPENAI_API_KEY"),
        model_name="Llama-3-70B-Instruct",
        temperature=0.0,
    )
    parser = JsonOutputParser(pydantic_object=Evaluation)
    prompt = PromptTemplate(
        template=SYSTEM_PROMPT,
        input_variables=["question", "answer", "ground_truth_answer"],
        partial_variables={"format_instructions": parser.get_format_instructions()},
    )
    chain = prompt | llm | parser

    conciseness, completeness = 0., 0.
    ranks = []
    for row in tqdm(df.itertuples(), total=len(df), desc='Evaluating answers...'):
        output = chain.invoke({
            "question": row.question,
            "answer": row.answers,
            "ground_truth_answer": row.ground_truth_answer
        })
        completeness += output['completeness']
        conciseness += output['conciseness']
        ranks.append(row.ranks)
    
    mean_conciseness = conciseness / len(df)
    mean_completeness = completeness / len(df)

    print({
        "mean_completeness": f"{round(mean_completeness*100)} %",
        "mean_conciseness": f"{round(mean_conciseness*100)} %"
    })

    print(pd.Series(ranks).value_counts())
```

Again, the full code is [here](https://github.com/GTimothee/RAG_experiments/blob/main/test_procedure_for_rag/evaluate.py)

## Disclaimer: It is for Q/A evaluation

By Q/A evaluation I mean that each question of the test set is associated to (and can be answered with) one document chunk. Consequently, this evaluation procedure is for Q/A RAG only; Indeed, if you are looking for an answer for which you must gather data from multiple chunks, this procedure would not evaluate that. Still, I believe it is a good starting point when evaluating your RAG, as if you cannot reliably find one document chunk, how could you find multiple target document chunks? You could probably start with this procedure and then add another procedure for more complex use cases.

## Conclusion

Now you have a pretty good idea of how your RAG app performs. Now every time you want to add a document to the knowledge base, add it to the dataset and run the test. You will know how much the addition of the new chunks in the database interferes with the existing documents, and what is the performance of your RAG system on your new document.

Medium link:
https://medium.com/@timothee.guedon/simple-test-procedure-and-metrics-for-your-rag-in-5-minutes-a86b329a5f7a
