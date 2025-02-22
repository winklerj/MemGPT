---
title: Adding support for new LLMs
excerpt: Adding new LLMs via model wrappers
category: 6580da9a40bb410016b8b0c3
---

> ⚠️ MemGPT + local LLM failure cases
>
> When using open LLMs with MemGPT, **the main failure case will be your LLM outputting a string that cannot be understood by MemGPT**. MemGPT uses function calling to manage memory (eg `edit_core_memory(...)` and interact with the user (`send_message(...)`), so your LLM needs generate outputs that can be parsed into MemGPT function calls.

### What is a "wrapper"?

To support function calling with open LLMs for MemGPT, we utilize "wrapper" code that:

1. turns `system` (the MemGPT instructions), `messages` (the MemGPT conversation window), and `functions` (the MemGPT function set) parameters from ChatCompletion into a single unified prompt string for your LLM
2. turns the output string generated by your LLM back into a MemGPT function call

Different LLMs are trained using different prompt formats (eg `#USER:` vs `<im_start>user` vs ...), and LLMs that are trained on function calling are often trained using different function call formats, so if you're getting poor performance, try experimenting with different prompt formats! We recommend starting with the prompt format (and function calling format) recommended in the HuggingFace model card, and experimenting from there.

We currently only support a few prompt formats in this repo ([located here](https://github.com/cpacker/MemGPT/tree/main/memgpt/local_llm/llm_chat_completion_wrappers))! If you write a new parser, please open a PR and we'll merge it in.

### Adding a new wrapper (change the prompt format + function parser)

To make a new wrapper (for example, because you want to try a different prompt format), you just need to subclass `LLMChatCompletionWrapper`. Your new wrapper class needs to implement two functions:

- One to go from ChatCompletion messages/functions schema to a prompt string
- And one to go from raw LLM outputs to a ChatCompletion response

```python
class LLMChatCompletionWrapper(ABC):

    @abstractmethod
    def chat_completion_to_prompt(self, messages, functions):
        """Go from ChatCompletion to a single prompt string"""
        pass

    @abstractmethod
    def output_to_chat_completion_response(self, raw_llm_output):
        """Turn the LLM output string into a ChatCompletion response"""
        pass
```

You can follow our example wrappers ([located here](https://github.com/cpacker/MemGPT/tree/main/memgpt/local_llm/llm_chat_completion_wrappers)).


### Example with [Airoboros](https://huggingface.co/jondurbin/airoboros-l2-70b-2.1) (llama2 finetune)

To help you get started, we've implemented an example wrapper class for a popular llama2 model **finetuned on function calling** (Airoboros). We want MemGPT to run well on open models as much as you do, so we'll be actively updating this page with more examples. Additionally, we welcome contributions from the community! If you find an open LLM that works well with MemGPT, please open a PR with a model wrapper and we'll merge it ASAP.

```python
class Airoboros21Wrapper(LLMChatCompletionWrapper):
    """Wrapper for Airoboros 70b v2.1: https://huggingface.co/jondurbin/airoboros-l2-70b-2.1"""

    def chat_completion_to_prompt(self, messages, functions):
        """
        Examples for how airoboros expects its prompt inputs: https://huggingface.co/jondurbin/airoboros-l2-70b-2.1#prompt-format
        Examples for how airoboros expects to see function schemas: https://huggingface.co/jondurbin/airoboros-l2-70b-2.1#agentfunction-calling
        """

    def output_to_chat_completion_response(self, raw_llm_output):
        """Turn raw LLM output into a ChatCompletion style response with:
        "message" = {
            "role": "assistant",
            "content": ...,
            "function_call": {
                "name": ...
                "arguments": {
                    "arg1": val1,
                    ...
                }
            }
        }
        """
```

See full file [here](https://github.com/cpacker/MemGPT/tree/main/memgpt/local_llm/llm_chat_completion_wrappers/airoboros.py).

---

## Wrapper FAQ

### Status of ChatCompletion w/ function calling and open LLMs

MemGPT uses function calling to do memory management. With [OpenAI's ChatCompletion API](https://platform.openai.com/docs/api-reference/chat/), you can pass in a function schema in the `functions` keyword arg, and the API response will include a `function_call` field that includes the function name and the function arguments (generated JSON). How this works under the hood is your `functions` keyword is combined with the `messages` and `system` to form one big string input to the transformer, and the output of the transformer is parsed to extract the JSON function call.

In the future, more open LLMs and LLM servers (that can host OpenAI-compatable ChatCompletion endpoints) may start including parsing code to do this automatically as standard practice. However, in the meantime, when you see a model that says it supports “function calling”, like Airoboros, it doesn't mean that you can just load Airoboros into a ChatCompletion-compatable endpoint like WebUI, and then use the same OpenAI API call and it'll just work.

1. When a model page says it supports function calling, they probably mean that the model was finetuned on some function call data (not that you can just use ChatCompletion with functions out-of-the-box). Remember, LLMs are just string-in-string-out, so there are many ways to format the function call data. E.g. Airoboros formats the function schema in YAML style (see https://huggingface.co/jondurbin/airoboros-l2-70b-3.1.2#agentfunction-calling) and the output is in JSON style. To get this to work behind a ChatCompletion API, you still have to do the parsing from `functions` keyword arg (containing the schema) to the model's expected schema style in the prompt (YAML for Airoboros), and you have to run some code to extract the function call (JSON for Airoboros) and package it cleanly as a `function_call` field in the response.

2. Partly because of how complex it is to support function calling, most (all?) of the community projects that do OpenAI ChatCompletion endpoints for arbitrary open LLMs do not support function calling, because if they did, they would need to write model-specific parsing code for each one.

### What is this all this extra code for?

Because of the poor state of function calling support in existing ChatCompletion API serving code, we instead provide a light wrapper on top of ChatCompletion that adds parsers to handle function calling support. These parsers need to be specific to the model you're using (or at least specific to the way it was trained on function calling). We hope that our example code will help the community add additional compatability of MemGPT with more function-calling LLMs - we will also add more model support as we test more models and find those that work well enough to run MemGPT's function set.
