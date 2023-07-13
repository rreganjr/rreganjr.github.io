---
layout: post
title: deeplearning.ai ChatGPT Prompt Engineering for Developers Course notes
---

## deeplearning.ai ChatGPT Prompt Engineering for Developers Course notes

see https://learn.deeplearning.ai/chatgpt-prompt-eng/lesson/5/inferring

setup python environment

```
import openai
import os

from dotenv import load_dotenv, find_dotenv
_ = load_dotenv(find_dotenv()) # read local .env file

openai.api_key  = os.getenv('OPENAI_API_KEY')

def get_completion(prompt, model="gpt-3.5-turbo"):
    messages = [{"role": "user", "content": prompt}]
    response = openai.ChatCompletion.create(
        model=model,
        messages=messages,
        temperature=0, # this is the degree of randomness of the model's output
    )
    return response.choices[0].message["content"]
```

setup the data

```
lamp_review = """
Needed a nice lamp for my bedroom, and this one had \
additional storage and not too high of a price point. \
Got it fast.  The string to our lamp broke during the \
transit and the company happily sent over a new one. \
Came within a few days as well. It was easy to put \
together.  I had a missing part, so I contacted their \
support and they very quickly got me the missing piece! \
Lumina seems to me to be a great company that cares \
about their customers and products!!
"""
```

create a prompt

describe to chatgpt what output is desired from the review data

```
prompt = f"""
Give a one sentence response.
First, identify the product being reviewed delimited with triple backticks.
Based on what you identified, what is the primary purpose of the product.
Review text: '''{lamp_review}'''

"""
response = get_completion(prompt)
print(response)
```

The output from ChatGPT

```
Product: Lamp with additional storage.
Primary purpose: To provide lighting and storage in a bedroom.
```

### more prompts and output

-------------------------------
```
prompt = f"""

First, identify the product being reviewed delimited with triple backticks.
Based on what you identified, List the adjectives used to describe the product.
Review text: '''{lamp_review}'''

"""
response = get_completion(prompt)
print(response)
```
The output from ChatGPT

Product being reviewed: lamp

Adjectives used to describe the product: nice, additional, not too high, fast, easy, missing, great, cares
-------------------------------
```
prompt = f"""

In the text delimited with triple backticks.
List out the nouns in JSON format
Review text: '''{lamp_review}'''

"""
response = get_completion(prompt)
print(response)
```
The output from ChatGPT

["lamp", "bedroom", "storage", "price point", "string", "transit", "company", "new one", "days", "missing part", "support", "missing piece", "Lumina", "company", "customers", "products"]
-------------------------------
```
prompt = f"""

In the text delimited with triple backticks.
Map out the nouns as keys and adjectives used on the specific noun as a list for the values in JSON format
text: '''{lamp_review}'''

"""
response = get_completion(prompt)
print(response)
```

The output from ChatGPT

{
  "lamp": ["nice", "additional", "not too high", "new", "easy", "missing", "great"],
  "bedroom": ["nice"],
  "storage": ["additional"],
  "price point": ["not too high"],
  "string": ["new"],
  "transit": ["during"],
  "company": ["happily", "new", "great"],
  "days": ["few"],
  "support": ["missing", "very quickly"],
  "piece": ["missing"],
  "Lumina": ["great", "cares"],
  "customers": ["great"],
  "products": ["great"]
}
​
-------------------------------
```
prompt = f"""

In the text delimited with triple backticks.
In the sentence 'It was easy to put together.'
Indicate the subject in each sentence.
text: '''{lamp_review}'''

"""
response = get_completion(prompt)
print(response)
```
The output from ChatGPT

The subject in each sentence is as follows:

- Needed a nice lamp for my bedroom
- this one
- The string to our lamp
- the company
- It
- I
- their support
- Lumina
-------------------------------
```
prompt = f"""

In the text delimited with triple backticks.
In the sentence 'It was easy to put together.'
What does 'It' refer to.
text: '''{lamp_review}'''

"""
response = get_completion(prompt)
print(response)
```
The output from ChatGPT

In the sentence 'It was easy to put together.', 'It' refers to the lamp mentioned earlier in the text.
​
-------------------------------
```
prompt = f"""

In the text delimited with triple backticks.
Rewrite the text convting pronouns to the referenced noun.
text: '''{lamp_review}'''

"""
response = get_completion(prompt)
print(response)
```
The output from ChatGPT

I needed a nice lamp for my bedroom, and the lamp had additional storage and not too high of a price point. I got the lamp fast. The string to the lamp broke during the transit and Lumina happily sent over a new string. The new string came within a few days as well. The lamp was easy to put together. I had a missing part, so I contacted Lumina's support and Lumina very quickly got me the missing piece! Lumina seems to me to be a great company that cares about its customers and products!!
-------------------------------
```
prompt = f"""

In the text delimited with triple backticks.
Rewrite the text convting references to the writer to "Willy" and other pronouns to the referenced noun.
text: '''{lamp_review}'''

"""
response = get_completion(prompt)
print(response)
```
The output from ChatGPT

Needed a nice lamp for Willy's bedroom, and this one had additional storage and not too high of a price point. Got it fast. The string to Willy's lamp broke during the transit and the company happily sent over a new one. Came within a few days as well. It was easy to put together. Willy had a missing part, so Willy contacted their support and they very quickly got Willy the missing piece! Lumina seems to me to be a great company that cares about their customers and products!!
-------------------------------
