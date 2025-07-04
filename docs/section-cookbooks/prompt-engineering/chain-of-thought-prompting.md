# Chain-of-Thought Prompting

{% embed url="https://youtu.be/Ys6aQjSioyk?si=MXgYt0JCIzcpkPke" %}

{% embed url="https://colab.research.google.com/github/Arize-ai/phoenix/blob/dbb46f76ef3292066858280d9c7718b674c6cebc/tutorials/prompts/chain_of_thought_prompting.ipynb" %}

LLMs excel at text generation, but their reasoning abilities depend on how we prompt them. **Chain of Thought (CoT)** prompting enhances logical reasoning by guiding the model to think step by step, improving accuracy in tasks like math, logic, and multi-step problem solving.

In this tutorial, you will:

* Examine how different prompting techniques influence reasoning by evaluating model performance on a dataset.
* Refine prompting strategies, progressing from basic approaches to structured reasoning.
* Utilize Phoenix to assess accuracy at each stage and explore the model's thought process.
* Learn how to apply CoT prompting effectively in real-world tasks.

⚠️ You'll need an OpenAI Key for this tutorial.

Let’s dive in! 🚀

## **Set up Dependencies and Keys**

```python
!pip install -qqqq "arize-phoenix>=8.0.0" datasets openinference-instrumentation-openai
```

Next you need to connect to Phoenix. The code below will connect you to a Phoenix Cloud instance. You can also [connect to a self-hosted Phoenix instance](https://arize.com/docs/phoenix/deployment) if you'd prefer.

```python
import os
from getpass import getpass

os.environ["PHOENIX_COLLECTOR_ENDPOINT"] = "https://app.phoenix.arize.com"
if not os.environ.get("PHOENIX_CLIENT_HEADERS"):
    os.environ["PHOENIX_CLIENT_HEADERS"] = "api_key=" + getpass("Enter your Phoenix API key: ")

if not os.environ.get("OPENAI_API_KEY"):
    os.environ["OPENAI_API_KEY"] = getpass("Enter your OpenAI API key: ")
```

## **Load Dataset Into Phoenix**

This dataset includes math word problems, step-by-step explanations, and their corresponding answers. As we refine our prompt, we'll test it against the dataset to measure and track improvements in performance.

Here, we also import the Phoenix Client, which enables us to create and modify prompts directly within the notebook while seamlessly syncing changes to the Phoenix UI.

```python
import uuid

from datasets import load_dataset

import phoenix as px
from phoenix.client import Client as PhoenixClient

ds = load_dataset("syeddula/math_word_problems")["train"]
ds = ds.to_pandas()
ds.head()

unique_id = uuid.uuid4()

# Upload the dataset to Phoenix
dataset = px.Client().upload_dataset(
    dataframe=ds,
    input_keys=["Word Problem"],
    output_keys=["Answer"],
    dataset_name=f"wordproblems-{unique_id}",
)
```

## **Zero-Shot Prompting** - Baseline

**Zero-shot prompting** is the simplest way to interact with a language model—it involves asking a question without providing any examples or reasoning steps. The model generates an answer based solely on its pre-trained knowledge.

This serves as our baseline for comparison. By evaluating its performance on our dataset, we can see how well the model solves math word problems without explicit guidance. In later sections, we’ll introduce structured reasoning techniques like **Chain of Thought (CoT)** to measure improvements in accuracy and answers.

```python
from openai import OpenAI
from openai.types.chat.completion_create_params import CompletionCreateParamsBase

from phoenix.client.types import PromptVersion

params = CompletionCreateParamsBase(
    model="gpt-3.5-turbo",
    temperature=0,
    messages=[
        {
            "role": "system",
            "content": "You are an evaluator who outputs the answer to a math word problem. Only respond with the integer answer. Be sure not include words, explanations, symbols, labels, or units and round all decimals answers.",
        },
        {"role": "user", "content": "{{Problem}}"},
    ],
)

prompt_identifier = "wordproblems"

prompt = PhoenixClient().prompts.create(
    name=prompt_identifier,
    prompt_description="A prompt for computing answers to word problems.",
    version=PromptVersion.from_openai(params),
)
```

At this stage, this initial prompt is now available in Phoenix under the Prompt tab. Any modifications made to the prompt moving forward will be tracked under **Versions**, allowing you to monitor and compare changes over time.

Prompts in Phoenix store more than just text—they also include key details such as the prompt template, model configurations, and response format, ensuring a structured and consistent approach to generating outputs.

![](https://storage.googleapis.com/arize-phoenix-assets/assets/images/CoT_Demo_1.png)

Next, we will define a task and evaluator for the experiment. Then, we run our experiment.

Because our dataset has ground truth labels, we can use a simple function to extract the answer and check if the calculated answer matches the expected output.

```python
import nest_asyncio

from phoenix.experiments import run_experiment

nest_asyncio.apply()


def zero_shot_prompt(input):
    client = OpenAI()
    resp = client.chat.completions.create(
        **prompt.format(variables={"Problem": input["Word Problem"]})
    )
    return resp.choices[0].message.content.strip()


def evaluate_response(output, expected):
    if not output.isdigit():
        return False
    return int(output) == int(expected["Answer"])


initial_experiment = run_experiment(
    dataset,
    task=zero_shot_prompt,
    evaluators=[evaluate_response],
    experiment_description="Zero-Shot Prompt",
    experiment_name="zero-shot-prompt",
    experiment_metadata={"prompt": "prompt_id=" + prompt.id},
)
```

We can review the results of the experiment in Phoenix. We achieved \~75% accuracy in this run. In the following sections, we will iterate on this prompt and see how our evaluation changes!

**Note**: Throughout this tutorial, you will encounter various evaluator outcomes. At times, you may notice a decline in performance compared to the initial experiment. However, this is not necessarily a flaw. Variations in results can arise due to factors such as the choice of LLM, inherent model behaviors, and randomness.

![](https://storage.googleapis.com/arize-phoenix-assets/assets/images/CoT_Demo_2.png)

## **Zero-Shot CoT Prompting**

Zero-shot prompting provides a direct answer, but it often struggles with complex reasoning. **Zero-Shot Chain of Thought (CoT)** prompting improves this by explicitly instructing the model to think step by step before arriving at a final answer.

By adding a simple instruction like _“Let’s think through this step by step,”_ we encourage the model to break down the problem logically. This structured reasoning can lead to more accurate answers, especially for multi-step math problems.

In this section, we'll compare Zero-Shot CoT against our baseline to evaluate its impact on performance. First, let's create the prompt.

```python
zero_shot_COT_template = """
You are an evaluator who outputs the answer to a math word problem.

You must always think through the problem logically before providing an answer.

First, show some of your reasoning.

Then output the integer answer ONLY on a final new line. In this final answer, be sure not include words, commas, labels, or units and round all decimals answers.

"""

params = CompletionCreateParamsBase(
    model="gpt-3.5-turbo",
    temperature=0,
    messages=[
        {"role": "system", "content": zero_shot_COT_template},
        {"role": "user", "content": "{{Problem}}"},
    ],
)

zero_shot_COT = PhoenixClient().prompts.create(
    name=prompt_identifier,
    prompt_description="Zero Shot COT prompt",
    version=PromptVersion.from_openai(params),
)
```

This updated prompt is now lives in Phoenix as a new prompt version.

Next, we run our task and evaluation by extracting the answer from the output of our LLM.

```python
import re


def zero_shot_COT_prompt(input):
    client = OpenAI()
    resp = client.chat.completions.create(
        **zero_shot_COT.format(variables={"Problem": input["Word Problem"]})
    )
    response_text = resp.choices[0].message.content.strip()
    lines = response_text.split("\n")
    final_answer = lines[-1].strip()
    final_answer = re.sub(r"^\*\*(\d+)\*\*$", r"\1", final_answer)
    return {"full_response": response_text, "final_answer": final_answer}


def evaluate_response(output, expected):
    final_answer = output["final_answer"]
    if not final_answer.isdigit():
        return False
    return int(final_answer) == int(expected["Answer"])


initial_experiment = run_experiment(
    dataset,
    task=zero_shot_COT_prompt,
    evaluators=[evaluate_response],
    experiment_description="Zero-Shot COT Prompt",
    experiment_name="zero-shot-cot-prompt",
    experiment_metadata={"prompt": "prompt_id=" + zero_shot_COT.id},
)
```

By clicking into the experiment in Phoenix, you can take a look at the steps the model took the reach the answer. By telling the model to think through the problem and output reasoning, we see a performance improvement.

![](https://storage.googleapis.com/arize-phoenix-assets/assets/images/CoT_Demo_3.png)

![](https://storage.googleapis.com/arize-phoenix-assets/assets/images/CoT_Demo_4.png)

## **Self-Consistency CoT Prompting**

Even with Chain of Thought prompting, a single response may not always be reliable. **Self-Consistency CoT** enhances accuracy by generating multiple reasoning paths and selecting the most common answer. Instead of relying on one response, we sample multiple outputs and aggregate them, reducing errors caused by randomness or flawed reasoning steps.

This method improves robustness, especially for complex problems where initial reasoning steps might vary. In this section, we'll compare Self-Consistency CoT to our previous promppts to see how using on multiple responses impacts overall performance.

Let's repeat the same process as above with a new prompt and evaluate the outcome.

```python
consistency_COT_template = """

You are an evaluator who outputs the answer to a math word problem.

Follow these steps:
1. Solve the problem **multiple times independently**, thinking through the solution carefully each time.
2. Show some of your reasoning for each independent attempt.
3. Identify the integer answer that appears most frequently across your attempts.
4. On a **new line**, output only this majority answer as a plain integer with **no words, commas, labels, units, or special characters**.
"""

params = CompletionCreateParamsBase(
    model="gpt-3.5-turbo",
    temperature=0,
    messages=[
        {"role": "system", "content": consistency_COT_template},
        {"role": "user", "content": "{{Problem}}"},
    ],
)

self_consistency_COT = PhoenixClient().prompts.create(
    name=prompt_identifier,
    prompt_description="self consistency COT prompt",
    version=PromptVersion.from_openai(params),
)
```

```python
def self_consistency_COT_prompt(input):
    client = OpenAI()
    resp = client.chat.completions.create(
        **self_consistency_COT.format(variables={"Problem": input["Word Problem"]})
    )
    response_text = resp.choices[0].message.content.strip()
    lines = response_text.split("\n")
    final_answer = lines[-1].strip()
    final_answer = re.sub(r"^\*\*(\d+)\*\*$", r"\1", final_answer)
    return {"full_response": response_text, "final_answer": final_answer}


def evaluate_response(output, expected):
    final_answer = output["final_answer"]
    if not final_answer.isdigit():
        return False
    return int(final_answer) == int(expected["Answer"])


initial_experiment = run_experiment(
    dataset,
    task=self_consistency_COT_prompt,
    evaluators=[evaluate_response],
    experiment_description="Self Consistency COT Prompt",
    experiment_name="self-consistency-cot-prompt",
    experiment_metadata={"prompt": "prompt_id=" + self_consistency_COT.id},
)
```

We've observed a significant improvement in performance! Since the prompt instructs the model to compute the answer multiple times independently, you may notice that the experiment takes slightly longer to run. You can click into the experiement explore to view the independent computations the model performed for each problem.

![](https://storage.googleapis.com/arize-phoenix-assets/assets/images/CoT_Demo_6.png)

![](https://storage.googleapis.com/arize-phoenix-assets/assets/images/CoT_Demo_5.png)

## Few Shot CoT Prompting

**Few-shot CoT prompting** enhances reasoning by providing worked examples before asking the model to solve a new problem. By demonstrating step-by-step solutions, the model learns to apply similar logical reasoning to unseen questions.

This method leverages **in-context learning**, allowing the model to generalize patterns from the examples.

In this final section, we’ll compare Few-Shot CoT against our previous prompts.

First, let's construct our prompt by sampling examples from a test dataset.

```python
ds = load_dataset("syeddula/math_word_problems")["test"]
few_shot_examples = ds.to_pandas().sample(5)
few_shot_examples
```

We now will construct our final prompt, run the experiement, and view the results. Under the **Prompts tab** in Phoenix, you can track the version history of your prompt and see what random examples were chosen.

```python
few_shot_COT_template = """
You are an evaluator who outputs the answer to a math word problem. You must always think through the problem logically before providing an answer. Show some of your reasoning.

Finally, output the integer answer ONLY on a final new line. In this final answer, be sure not include words, commas, labels, or units and round all decimals answers.

Here are some examples of word problems, step by step explanations, and solutions to guide your reasoning:

{examples}
"""
params = CompletionCreateParamsBase(
    model="gpt-3.5-turbo",
    temperature=0,
    messages=[
        {"role": "system", "content": few_shot_COT_template.format(examples=few_shot_examples)},
        {"role": "user", "content": "{{Problem}}"},
    ],
)

few_shot_COT = PhoenixClient().prompts.create(
    name=prompt_identifier,
    prompt_description="Few Shot COT prompt",
    version=PromptVersion.from_openai(params),
)
```

![](https://storage.googleapis.com/arize-phoenix-assets/assets/images/CoT_Demo_8.png)

```python
def few_shot_COT_prompt(input):
    client = OpenAI()
    resp = client.chat.completions.create(
        **few_shot_COT.format(variables={"Problem": input["Word Problem"]})
    )
    response_text = resp.choices[0].message.content.strip()
    lines = response_text.split("\n")
    final_answer = lines[-1].strip()
    final_answer = re.sub(r"^\*\*(\d+)\*\*$", r"\1", final_answer)
    return {"full_response": response_text, "final_answer": final_answer}


def evaluate_response(output, expected):
    final_answer = output["final_answer"]
    if not final_answer.isdigit():
        return False
    return int(final_answer) == int(expected["Answer"])


import nest_asyncio

from phoenix.experiments import run_experiment

nest_asyncio.apply()

initial_experiment = run_experiment(
    dataset,
    task=few_shot_COT_prompt,
    evaluators=[evaluate_response],
    experiment_description="Few-Shot COT Prompt",
    experiment_name="few-shot-cot-prompt",
    experiment_metadata={"prompt": "prompt_id=" + few_shot_COT.id},
)
```

## **Final Results**

After running all of your experiments, you can compare the performance of different prompting techniques. Keep in mind that results may vary due to randomness and the model's non-deterministic behavior.

You can review your prompt version history in the **Prompts tab** and explore the **Playground** to iterate further and run additional experiments.

To refine and test these prompts against other datasets, experiment with Chain of Thought (CoT) prompting to see its relevance to your specific use cases. With Phoenix, you can seamlessly integrate this process into your workflow using the TypeScript and Python Clients.

![](https://storage.googleapis.com/arize-phoenix-assets/assets/images/CoT_Demo_7.png)

From here, you can check out more [examples on Phoenix](https://arize.com/docs/phoenix/notebooks), and if you haven't already, [please give us a star on GitHub!](https://github.com/Arize-ai/phoenix) ⭐️
