 - provide the clear and specific instructions for model to guide through 
- don't confuse, that writing a clear prompt doesn't means that it should be a short one. in many cases longer prompt provide more clarity and context for the model 

## tactic 1 : use delimiters 
- triple quotes : """
- triple backticks : ``
- triple dashes : ---
- angle brackets : <>
- xml tags : <> </tag>

- Use delimiters to clearly indicate distinct parts of the input. for an example 
```
text = f"""
You should express what you want a model to do by \ 
providing instructions that are as clear and \ 
specific as you can possibly make them. \ 
This will guide the model towards the desired output, \ 
and reduce the chances of receiving irrelevant \ 
or incorrect responses. Don't confuse writing a \ 
clear prompt with writing a short prompt. \ 
In many cases, longer prompts provide more clarity \ 
and context for the model, which can lead to \ 
more detailed and relevant outputs.
"""
prompt = f"""
Summarize the text delimited by triple backticks \ 
into a single sentence.
```{text}```
"""
```
- delimiters also help you for avoid prompt injections. 
	- prompt injection  : if users are allowed to add some instructions your prompt. they might give conflicting instructions to the model rather than your doing what you wanted to do 
	- so in previous example models knows that it only need to summarise the text that inside the triple backticks 

## tactic 2 : Asked for structured output 
- for make parsing the model output easier we can ask for the specific output. like JSON or HTML. for an example 
```python
prompt = f"""
Generate a list of three made-up sinhala book titles along \ 
with their authors and genres. 
Provide them in JSON format with the following keys: 
book_id, title, author, genre.
"""

```
## tactic 3 : Ask the model to check whether conditions are satisfied.

- we can ask for model to check whether all the assumptions are satisfied are not. if not completed we can ask the model to show some kind of an error message. also we can think of some edge cases and give instructions for model to how to handle them. to avoid unexpected errors and results. 
- as an example 
```python
text_1 = f"""
Making a cup of tea is easy! First, you need to get some \ 
water boiling. While that's happening, \ 
grab a cup and put a tea bag in it. Once the water is \ 
hot enough, just pour it over the tea bag. \ 
Let it sit for a bit so the tea can steep. After a \ 
few minutes, take out the tea bag. If you \ 
like, you can add some sugar or milk to taste. \ 
And that's it! You've got yourself a delicious \ 
cup of tea to enjoy.
"""
prompt = f"""
You will be provided with text delimited by triple quotes. 
If it contains a sequence of instructions, \ 
re-write those instructions in the following format:

Step 1 - ...
Step 2 - …
…
Step N - …

If the text does not contain a sequence of instructions, \ 
then simply write \"No steps provided.\"

\"\"\"{text_1}\"\"\"
"""

```

- `\` in here is used differentiate the double quote from the python syntax double quote.
- if there is a text that contains no instructions then the result will be `no steps provided`.

## tactic 4 : Few shot prompting 

- give successful examples of completing tasks then ask model to perform the task. for an example 

```python 
prompt = f"""
Your task is to answer in a consistent style.

<child>: Teach me about patience.

<grandparent>: The river that carves the deepest \ 
valley flows from a modest spring; the \ 
grandest symphony originates from a single note; \ 
the most intricate tapestry begins with a solitary thread.

<child>: Teach me about resilience.
"""
```