# Gemma 4 — Tool Calling Reliability

## Overview

An independent bug-fixing case study based on the Hugging Face Transformers codebase.

**Area:** LLM tool calling, response parsing, inference serving  
**Language:** Python  
**Techniques:** Regular expressions, regression testing, red–green testing, pytest, Ruff

## Problem

Gemma 4 response templates used the following regular expression to capture tool names:

```python
r"<\|tool_call>call:(?P<name>\w+)"
