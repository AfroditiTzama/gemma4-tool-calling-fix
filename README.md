# Gemma 4 — Tool Calling Parser Bug Fix

## Overview

This project documents an independent fix for a tool-call parsing issue in the Gemma 4 integration of Hugging Face Transformers.

The original parser failed to correctly recognize function names containing hyphens (`-`) or periods (`.`). This could cause malformed tool-call arguments and JSON parsing errors.

The implementation fixes the tool-name matching expression in both the Gemma 4 response template and the CLI serving fallback configuration, with regression tests verifying the behavior.

**Technical focus:** LLM Tool Calling · Response Parsing · CLI Serving · Python · Regular Expressions · Regression Testing

## 1. Problem Description

Gemma 4 response templates originally used the following regular expression to extract tool names:

`r"<\|tool_call>call:(?P<name>\w+)"`

The `\w+` character class matches letters, numbers, and underscores, but does not match hyphens or periods.

As a result, the parser could not correctly process tool calls using function names such as `get-weather` or `weather.api`.

### Example

Consider the following Gemma 4 tool call:

`<|tool_call>call:get-weather{location:<|"|>Athens<|"|>}<tool_call|>`

**Expected behavior:**

- Tool name: `get-weather`
- Arguments: `{"location": "Athens"}`

**Actual behavior before the fix:**

- Extracted tool name: `get`
- Remaining content: `-weather{location:<|"|>Athens<|"|>}`

The parser treated the remaining portion of the function name as part of the arguments, resulting in a JSON parsing failure.

## 2. Root Cause Analysis

The root cause was an overly restrictive regular expression used in the Gemma 4 response-template configuration.

**Original expression:**

`r"<\|tool_call>call:(?P<name>\w+)"`

Because the named capture group only accepted word characters, it terminated prematurely when encountering `-` or `.`.

The same pattern existed in the CLI serving fallback configuration, meaning that fixing only the model response template would leave another parsing path affected.

## 3. Implementation

The original expression was replaced with:

`r"<\|tool_call>call:(?P<name>[\w.-]+)"`

The updated character class supports hyphens and periods while preserving existing support for alphanumeric characters and underscores.

### Modified Files

| File | Modification |
|---|---|
| `src/transformers/models/gemma4/convert_gemma4_weights.py` | Updated Gemma 4 tool-name matching |
| `src/transformers/cli/serving/utils.py` | Updated CLI serving fallback regex |
| `tests/utils/test_chat_parsing.py` | Updated the Gemma 4 test template and added regression coverage |
| `tests/cli/test_serve.py` | Added a regression test using the actual serving fallback configuration |

## 4. Regression Testing

Regression tests were developed to validate the corrected parser against different function-name formats.

| Function Name | Before Fix | After Fix |
|---|---|---|
| `get-weather` | Failed | Passed |
| `weather.api` | Failed | Passed |
| `get_weather` | Passed | Passed |

Each test verifies that the complete function name is preserved and the arguments are correctly extracted as `{"location": "Athens"}`.

### CLI Serving Validation

A separate unit test was added to verify the Gemma 4 serving fallback using the Transformers response parser.

The test uses a mocked model and processor to retrieve the real fallback response template.

This approach validates serving-layer parsing without starting an inference server or downloading Gemma 4 model weights.

## 5. Red–Green Testing

A red–green testing procedure was performed to confirm that the regression test detects the original defect.

**RED — Original Implementation**

The original `\w+` expression was temporarily restored in the CLI serving configuration.

The regression test reproduced the failure for hyphenated and dotted function names.

Example error:

`ValueError: json: could not parse after dialect transforms.`

Original content:

`-weather{location:<|"|>Athens<|"|>}`

Transformed content:

`-weather{"location":"Athens"}`

**GREEN — Corrected Implementation**

After restoring the corrected expression, all three test cases passed.

This verified that the tests detect the original defect and that the implemented change resolves the observed parsing failures.

## 6. Validation Results

**CLI Serving Regression Test**

`1 passed, 196 deselected, 3 subtests passed`

**Additional Validation**

- Chat-parsing test suite: 81 passed, 1 skipped
- Ruff linting: passed
- Git diff validation: passed
- Red–green verification: completed successfully

The tests were executed locally using Python, pytest, and Ruff.

**Scope:** These tests validate the response-template parsing behavior. They do not constitute an end-to-end Gemma 4 inference test.

## 7. Source Code

The complete implementation is available in the personal Hugging Face Transformers fork.

**Implementation branch:**

[AfroditiTzama/transformers — fix/gemma4-tool-parsing](https://github.com/AfroditiTzama/transformers/tree/fix/gemma4-tool-parsing)

**Latest commit:** `b51d44ccab`

**Related upstream references:**

- [Original Transformers Issue #49422](https://github.com/huggingface/transformers/issues/49422)
- [Existing Upstream Pull Request #49423](https://github.com/huggingface/transformers/pull/49423)

## 8. Contribution Status

This case study documents an independently implemented and locally validated solution in a personal fork of Hugging Face Transformers.

A separate upstream Pull Request addresses the same underlying issue. This implementation is therefore presented as an independent exercise rather than a merged contribution to the official Transformers repository.
