> this is whole guide is based developer on prospective that you are developing applications using OpenAI's chatGPT API. 

## Introduction

there are two types of LLMs 
- Base LLM 
	- predict the next word based on the text training data. its typical ML model, its not giving any generative answers.
- Instructions tuned LLM 
	- this model  will give a generative answer based on the large amount of data that he has been trained on.


## Prompting principles 

-  [[write clear and specific instructions]]
- [[Give the model time to think]]
	 

## Model limitations 

- model doesn't know the boundary of its knowledge so some times it might make up the things. 
- this is called as hallucination. 

##### To avoid hallucination 
- give instructions to model to first find the relevant information 
- then answer the question. 
- based on the relevant information. 