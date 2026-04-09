---
layout: post
title: "Technical Hurdles in AI & How to Solve Them [P1]"
date: 2026-04-08
---

This post talks about technical hurdles that I encountered and solutions for them. This is a simple collection of the issues and they are not sorted or split by any particular criteria.

# How do you handle LLMs that fail to follow a JSON schema?

The fastest way is to explicitly ask LLMs to give responses in JSON format, but this still can be violated. So we need stronger ways to enforce the schema. Enter **Pydantic validation** and **constrained decoding.** 

## Pydantic validation

### How it Works: "Parsing, not just Validating"

Pydantic's philosophy is that it doesn't just check your data; it coerces it into the correct format.

For example, if you tell Pydantic a field should be an integer and you give it the string `"123"`, Pydantic will automatically convert it to the number `123`. If you give it `"abc"`, it will
raise a clear, readable error.

### A Simple Example

Imagine you are building an agent that extracts contact info from an email. You define a "Schema" using a Pydantic Model:

```python
from pydantic import BaseModel, EmailStr, Field

class Contact(BaseModel):
    name: str
    age: int = Field(gt=0, lt=120)  # Age must be between 0 and 120
    email: EmailStr                 # Must be a valid email format
    is_lead: bool = False           # Default value if not provided
```

If an LLM outputs a messy JSON, Pydantic acts as the "Bouncer":

**Input:**
```json
{"name": "Victor", "age": "35", "email": "victor@example.com"}
```
**Result:** Pydantic converts `"35"` to an `int` and returns a clean Python object.

**Invalid Input:**
```json
{"name": "Victor", "age": -5, "email": "not-an-email"}
```
**Result:** Pydantic throws a `ValidationError` explaining exactly what failed.

### Why Pydantic is critical for AI agents/workflows?

**TLDR:** Pydantic ensures that a **data contract** between an agent and a downstream API is not broken. Below are more details.

- *Reliable Function Calling:* When an agent needs to write updates that require specific JSON schema, Pydantic can be used to define that schema, ensuring the LLM never sends a "broken" request to  external APIs.

- *Structured Outputs:* Most modern LLM frameworks (like Instructor or LangChain) use Pydantic under the hood to "force" the LLM to return valid JSON instead of conversational prose.

- *Self-Correction Loops:* If Pydantic catches a validation error (e.g., the LLM forgot a required field), you can actually send the error message back to the LLM and ask it to fix its own mistake. This is a core pattern in "Agentic" workflows.


## Constrained Decoding

While Pydantic validates the data *after* the LLM has already generated it, **Constrained Decoding** (also known as **Guided Generation**) forces the LLM to follow a specific structure *while it is still thinking*.

It is the difference between checking a recipe for mistakes after it's cooked versus physically only allowing the chef to reach for specific ingredients.

### The Technical Mechanism: Logit Bias

To understand this, remember that LLMs predict the next "token" (word fragment) by calculating a probability for every single word in their vocabulary.

- **Standard Decoding:** The model picks the most likely next token. It might decide to say, `"Sure! Here is the JSON..."` which breaks your code.
- **Constrained Decoding:** You apply a mask or a **logit bias**. You tell the model: "For the next token, the only valid options are `{` or `[`." The probability of every other word in the
dictionary is set to $-\infty$.

### Common Tools

Some industry-standard libraries:

- **Outlines:** A popular library that uses Context-Free Grammars (CFG) to guarantee JSON or Regex-compliant output.
- **Guidance (Microsoft):** Allows you to interleave natural language with specific variable slots.
- **vLLM / llama.cpp:** These serving engines have built-in support for "Grammar" files (GBNF) to restrict outputs at the inference level.


## Table comparing two solutions

| Feature | Post-Generation (Pydantic) | Constrained Decoding (Outlines/vLLM) |
|---|---|---|
| Strategy | "Try and Catch" | "Prevent by Design" |
| Reliability | 90–95% (LLM might fail to follow instructions) | 100% (mathematically impossible to break schema) |
| Latency | Low, but requires retries if validation fails. | Slightly higher per-token overhead, but no retries. |
| Use Case | Complex logic and internal data cleaning. | API calls, Tool-use, and strict JSON outputs. |


