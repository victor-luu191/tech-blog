---
layout: post
title: "Technical Hurdles in AI & How to Solve Them [P1]"
date: 2026-04-08
---

This post talks about technical hurdles that I encountered and solutions for them. This is a simple collection of the issues and they are not sorted or split by any particular criteria.

# How do you handle LLMs that fail to follow a JSON schema?

The fastest way is to explicitly ask LLMs to give responses in JSON format, but this still can be violated. So we need stronger ways to enforce the schema. Enter **Pydantic validation** and **constrained decoding.** 

## 1. How it Works: "Parsing, not just Validating"

Pydantic's philosophy is that it doesn't just check your data; it coerces it into the correct format.

For example, if you tell Pydantic a field should be an integer and you give it the string `"123"`, Pydantic will automatically convert it to the number `123`. If you give it `"abc"`, it will
raise a clear, readable error.

## 2. A Simple Example

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

## Why Pydantic is critical for AI agents/workflows?

**TLDR:** Pydantic ensures that a **data contract** between an agent and a downstream API is not broken. Below are more details.

- *Reliable Function Calling:* When an agent needs to write updates that require specific JSON schema, Pydantic can be used to define that schema, ensuring the LLM never sends a "broken" request to  external APIs.

- *Structured Outputs:* Most modern LLM frameworks (like Instructor or LangChain) use Pydantic under the hood to "force" the LLM to return valid JSON instead of conversational prose.

- *Self-Correction Loops:* If Pydantic catches a validation error (e.g., the LLM forgot a required field), you can actually send the error message back to the LLM and ask it to fix its own mistake. This is a core pattern in "Agentic" workflows.


