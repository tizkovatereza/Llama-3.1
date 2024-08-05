# Llama 3.1 with Code Interpreter

## Tutorial for running code with Llama 3.1

Llama 3.1, the latest release from Meta, is an agentic framework featuring multi-step reasoning, integrated tool search, and a code interpreter. It comes in three sizes: 8B, 70B, and 405B. The 405B version is the first openly available model by Meta that rivals other top LLMs in areas like general knowledge, math, tool use, and multilingual translation.

At this moment, Llama 3.1 is one of the most capable open-source models.


In this step-by-step guide, we show how to give it the code interpreter capabilities and test it on coding and data analysis tasks.


## Llama 3.1 function calling
Now, Llama 3.1 natively supports function calling.

The Llama 3.1 family of models has “Built-In Tools” for function calling, in particular

Brave Search used to perform web searches.
Wolfram Alpha used to perform complex mathematical calculations.
Code Interpreter used to evaluate the generated Python code.

Built-in support only means that the models are better trained at triggering the use of these functions. The functions still need to be implemented locally, for which you can follow the tutorial.

## Different types of LLM code generation

The function calling feature is not automatic for LLMs. 

### A) Function calling
Also called tool calling. It allows adding “tools” that the LLM can decide to call. It uses tool definitions via JSON schemas.

The tools usually have a format like this:

```js
const tools: Array<Tool> = [
  {
    name: 'execute_python',
    description: 'Execute python code in a Jupyter notebook cell and returns any result, stdout, stderr, display_data, and error.',
    input_schema: {
      type: 'object',
      properties: {
        code: {
          type: 'string',
          description: 'The python code to execute in a single cell.'
        }
      },
      required: ['code']
    }
  }
]
```


There is great tutorial by Together AI for function calling with Llama 3.1.

### B) “Manual” parsing of code

When a LLM lacks a built-in support for function calling, we can still achieve the same result. In such case, we have to instruct the model manually on how to use each “tool” we want to add.

```js
Generally, you follow these rules:

ALWAYS FORMAT YOUR RESPONSE IN MARKDOWN
ALWAYS RESPOND ONLY WITH CODE IN CODE BLOCK LIKE THIS:
```python
{code}
```


```js
const codeBlockMatch = responseMessage.match(/```python\n([\s\S]*?)\n```/)

if (codeBlockMatch && codeBlockMatch[1]) {
  const pythonCode = codeBlockMatch[1]
  console.log('CODE TO RUN')
  console.log(pythonCode)
  const codeInterpreterResults = await codeInterpret(codeInterpreter, pythonCode)
  return codeInterpreterResults
} else {
  console.log('Failed to match any Python code in model\'s response')
  return []
}
```




LLM code interpreting with Llama 3.1
Generating the correct code is one thing, but running the code is another. To immediately execute (and hence test) the code provided by Llama 3.1, we can use the Code Interpreter SDK.






Get full code
We have a full code on GitHub.
Code Interpreter SDK
We will show how to build a code interpreter with Llama 3 on Groq, and powered by open-source Code Interpreter SDK by E2B. The E2B Code Interpreter SDK quickly creates a secure cloud sandbox powered by Firecracker. Inside this sandbox is a running Jupyter server that the LLM can use.
Key links
Get started with E2B
Get started with Llama
Get started with Groq
Follow E2B


Overview
Prerequisites






### 1. Prerequisites

You can choose a JavaScript & TypesScript or Python version of this example. In the former case, create an `index.ts` file for the main program, and `.env` file that looks like this:

```
# TODO: Get your Together AI API key from https://api.together.xyz/settings/api-keys
TOGETHER_API_KEY = ""

# TODO: Get your E2B API key from https://e2b.dev/docs
E2B_API_KEY = ""
```

In the latter case, create just `main.ipynb` file.

Get the E2B API key [here](https://e2b.dev/docs/getting-started/api-key) and the Together AI API key [here](https://api.together.xyz/settings/api-keys).

Download the CSV file from [here](https://www.kaggle.com/datasets/nishanthsalian/socioeconomic-country-profiles/code) and upload it to the same directory as your program. Rename it to `data.csv`.


### 2. Install the SDKs

1️⃣ JavaScript & TypeScript version:

```sh
npm install @e2b/code-interpreter@0.0.5 together-ai@0.6.0-alpha.4 dotenv@16.4.5

```

2️⃣ Python version:

```sh
pip install together==0.6.0 e2b-code-interpreter==0.0.10 dotenv==1.0.0

```

### 3. Set up the API keys and model instructions.
In this step you upload your E2B and Together API keys to the program. In the JS & TS case, the API keys are stored in the `.env` file, in the Python case, they are added directly to the notebook.
You pick the model of your choice by uncommenting it. There are some recommended models that are great at code generation, but you can add a different one from [here](https://api.together.ai/models).

The model is assigned a data scientist role and explained the uploaded CSV. You can choose different data but need to update the instructions accordingly.
   
1️⃣ JavaScript & TypeScript version:


```js
import fs from 'node:fs'
import { CodeInterpreter, Result, ProcessMessage } from '@e2b/code-interpreter'
import * as dotenv from 'dotenv'
import Together from 'together-ai/index.mjs'

dotenv.config()

const TOGETHER_API_KEY = process.env.TOGETHER_API_KEY || ''
const E2B_API_KEY = process.env.E2B_API_KEY || ''

if (!TOGETHER_API_KEY) {
    console.error('Error: TOGETHER_API_KEY is not provided. Please set the TOGETHER_API_KEY in your environment variables.')
    process.exit(1)
}

if (!E2B_API_KEY) {
    console.error('Error: E2B_API_KEY is not provided. Please set the E2B_API_KEY in your environment variables.')
    process.exit(1)
}

// Choose from the codegen models:

const MODEL_NAME = 'meta-llama/Meta-Llama-3.1-405B-Instruct-Turbo'
// const MODEL_NAME = 'meta-llama/Meta-Llama-3.1-70B-Instruct-Turbo'
// const MODEL_NAME = 'meta-llama/Meta-Llama-3.1-8B-Instruct-Turbo'
// const MODEL_NAME = 'Qwen/Qwen2-72B-Instruct'
// const MODEL_NAME = "codellama/CodeLlama-70b-Instruct-hf"
// const MODEL_NAME = "deepseek-ai/deepseek-coder-33b-instruct"

// See the complete list of Together AI models here: https://api.together.ai/models.


const SYSTEM_PROMPT = `
You're a python data scientist. You are given tasks to complete and you run Python code to solve them.

Information about the csv dataset:
- It's in the \`/home/user/data.csv\` file
- The CSV file is using , as the delimiter
- It has the following columns (examples included):
    - country: "Argentina", "Australia"
    - Region: "SouthAmerica", "Oceania"
    - Surface area (km2): for example, 2780400
    - Population in thousands (2017): for example, 44271
    - Population density (per km2, 2017): for example, 16.2
    - Sex ratio (m per 100 f, 2017): for example, 95.9
    - GDP: Gross domestic product (million current US$): for example, 632343
    - GDP growth rate (annual %, const. 2005 prices): for example, 2.4
    - GDP per capita (current US$): for example, 14564.5
    - Economy: Agriculture (% of GVA): for example, 10.0
    - Economy: Industry (% of GVA): for example, 28.1
    - Economy: Services and other activity (% of GVA): for example, 61.9
    - Employment: Agriculture (% of employed): for example, 4.8
    - Employment: Industry (% of employed): for example, 20.6
    - Employment: Services (% of employed): for example, 74.7
    - Unemployment (% of labour force): for example, 8.5
    - Employment: Female (% of employed): for example, 43.7
    - Employment: Male (% of employed): for example, 56.3
    - Labour force participation (female %): for example, 48.5
    - Labour force participation (male %): for example, 71.1
    - International trade: Imports (million US$): for example, 59253
    - International trade: Exports (million US$): for example, 57802
    - International trade: Balance (million US$): for example, -1451
    - Education: Government expenditure (% of GDP): for example, 5.3
    - Health: Total expenditure (% of GDP): for example, 8.1
    - Health: Government expenditure (% of total health expenditure): for example, 69.2
    - Health: Private expenditure (% of total health expenditure): for example, 30.8
    - Health: Out-of-pocket expenditure (% of total health expenditure): for example, 20.2
    - Health: External health expenditure (% of total health expenditure): for example, 0.2
    - Education: Primary gross enrollment ratio (f/m per 100 pop): for example, 111.5/107.6
    - Education: Secondary gross enrollment ratio (f/m per 100 pop): for example, 104.7/98.9
    - Education: Tertiary gross enrollment ratio (f/m per 100 pop): for example, 90.5/72.3
    - Education: Mean years of schooling (female): for example, 10.4
    - Education: Mean years of schooling (male): for example, 9.7
    - Urban population (% of total population): for example, 91.7
    - Population growth rate (annual %): for example, 0.9
    - Fertility rate (births per woman): for example, 2.3
    - Infant mortality rate (per 1,000 live births): for example, 8.9
    - Life expectancy at birth, female (years): for example, 79.7
    - Life expectancy at birth, male (years): for example, 72.9
    - Life expectancy at birth, total (years): for example, 76.4
    - Military expenditure (% of GDP): for example, 0.9
    - Population, female: for example, 22572521
    - Population, male: for example, 21472290
    - Tax revenue (% of GDP): for example, 11.0
    - Taxes on income, profits and capital gains (% of revenue): for example, 12.9
    - Urban population (% of total population): for example, 91.7

Generally, you follow these rules:
- ALWAYS FORMAT YOUR RESPONSE IN MARKDOWN
- ALWAYS RESPOND ONLY WITH CODE IN CODE BLOCK LIKE THIS:
\`\`\`python
{code}
\`\`\`
- the Python code runs in jupyter notebook.
- every time you generate Python, the code is executed in a separate cell. it's okay to make multiple calls to \`execute_python\`.
- display visualizations using matplotlib or any other visualization library directly in the notebook. don't worry about saving the visualizations to a file.
- you have access to the internet and can make api requests.
- you also have access to the filesystem and can read/write files.
- you can install any pip package (if it exists) if you need to be running \`!pip install {package}\`. The usual packages for data analysis are already preinstalled though.
- you can run any Python code you want, everything is running in a secure sandbox environment
`
```

2️⃣ Python version:

```python
import os
from dotenv import load_dotenv
import os
import json
import re
from together import Together
from e2b_code_interpreter import CodeInterpreter

load_dotenv()

# TODO: Get your Together AI API key from https://api.together.xyz/settings/api-keys
TOGETHER_API_KEY = os.getenv("TOGETHER_API_KEY")

# TODO: Get your E2B API key from https://e2b.dev/docs
E2B_API_KEY = os.getenv("E2B_API_KEY")

# Choose from the codegen models:

MODEL_NAME = "meta-llama/Meta-Llama-3.1-405B-Instruct-Turbo"
# MODEL_NAME = "meta-llama/Meta-Llama-3.1-70B-Instruct-Turbo"
# MODEL_NAME = "meta-llama/Meta-Llama-3.1-8B-Instruct-Turbo"
# MODEL_NAME = "codellama/CodeLlama-70b-Instruct-hf"
# MODEL_NAME = "deepseek-ai/deepseek-coder-33b-instruct"
# MODEL_NAME = "Qwen/Qwen2-72B-Instruct"
# See the complete list of Together AI models here: https://api.together.ai/models.

SYSTEM_PROMPT = """You're a Python data scientist. You are given tasks to complete and you run Python code to solve them.

Information about the csv dataset:
- It's in the `/home/user/data.csv` file
- The CSV file is using , as the delimiter
- It has the following columns (examples included):
    - country: "Argentina", "Australia"
    - Region: "SouthAmerica", "Oceania"
    - Surface area (km2): for example, 2780400
    - Population in thousands (2017): for example, 44271
    - Population density (per km2, 2017): for example, 16.2
    - Sex ratio (m per 100 f, 2017): for example, 95.9
    - GDP: Gross domestic product (million current US$): for example, 632343
    - GDP growth rate (annual %, const. 2005 prices): for example, 2.4
    - GDP per capita (current US$): for example, 14564.5
    - Economy: Agriculture (% of GVA): for example, 10.0
    - Economy: Industry (% of GVA): for example, 28.1
    - Economy: Services and other activity (% of GVA): for example, 61.9
    - Employment: Agriculture (% of employed): for example, 4.8
    - Employment: Industry (% of employed): for example, 20.6
    - Employment: Services (% of employed): for example, 74.7
    - Unemployment (% of labour force): for example, 8.5
    - Employment: Female (% of employed): for example, 43.7
    - Employment: Male (% of employed): for example, 56.3
    - Labour force participation (female %): for example, 48.5
    - Labour force participation (male %): for example, 71.1
    - International trade: Imports (million US$): for example, 59253
    - International trade: Exports (million US$): for example, 57802
    - International trade: Balance (million US$): for example, -1451
    - Education: Government expenditure (% of GDP): for example, 5.3
    - Health: Total expenditure (% of GDP): for example, 8.1
    - Health: Government expenditure (% of total health expenditure): for example, 69.2
    - Health: Private expenditure (% of total health expenditure): for example, 30.8
    - Health: Out-of-pocket expenditure (% of total health expenditure): for example, 20.2
    - Health: External health expenditure (% of total health expenditure): for example, 0.2
    - Education: Primary gross enrollment ratio (f/m per 100 pop): for example, 111.5/107.6
    - Education: Secondary gross enrollment ratio (f/m per 100 pop): for example, 104.7/98.9
    - Education: Tertiary gross enrollment ratio (f/m per 100 pop): for example, 90.5/72.3
    - Education: Mean years of schooling (female): for example, 10.4
    - Education: Mean years of schooling (male): for example, 9.7
    - Urban population (% of total population): for example, 91.7
    - Population growth rate (annual %): for example, 0.9
    - Fertility rate (births per woman): for example, 2.3
    - Infant mortality rate (per 1,000 live births): for example, 8.9
    - Life expectancy at birth, female (years): for example, 79.7
    - Life expectancy at birth, male (years): for example, 72.9
    - Life expectancy at birth, total (years): for example, 76.4
    - Military expenditure (% of GDP): for example, 0.9
    - Population, female: for example, 22572521
    - Population, male: for example, 21472290
    - Tax revenue (% of GDP): for example, 11.0
    - Taxes on income, profits and capital gains (% of revenue): for example, 12.9
    - Urban population (% of total population): for example, 91.7

Generally, you follow these rules:
- ALWAYS FORMAT YOUR RESPONSE IN MARKDOWN
- ALWAYS RESPOND ONLY WITH CODE IN CODE BLOCK LIKE THIS:
      ```python
      {code}
      ```
   - the Python code runs in jupyter notebook.
   - every time you generate Python, the code is executed in a separate cell. it's okay to make multiple calls to `execute_python`.
   - display visualizations using matplotlib or any other visualization library directly in the notebook. don't worry about saving the visualizations to a file.
   - you have access to the internet and can make api requests.
   - you also have access to the filesystem and can read/write files.
   - you can install any pip package (if it exists) if you need to be running `!pip install {package}`. The usual packages for data analysis are already preinstalled though.
   - you can run any Python code you want, everything is running in a secure sandbox environment
   """
```


### 4. Add code interpreting capabilities and initialize the model

Now we define the function that will use the code interpreter by E2B. Everytime the LLM assistant decides that it needs to execute code, this function will be used. Read more about the Code Interpreter SDK [here](https://e2b.dev/docs/code-interpreter/installation).
We also initialize the Together AI client. The function for matching code blocks is important because we need to pick the right part of the output that contains the code produced by the LLM. The chat function takes care of the interaction with the LLM. It calls the E2B code interpreter anytime there is a code to be run.

1️⃣ JavaScript & TypeScript version:


```js
const together = new Together()

async function codeInterpret(codeInterpreter: CodeInterpreter, code: string): Promise<Result[]> {
    console.log('Running code interpreter...')

    const exec = await codeInterpreter.notebook.execCell(code, {
        onStderr: (msg: ProcessMessage) => console.log('[Code Interpreter stderr]', msg),
        onStdout: (stdout: ProcessMessage) => console.log('[Code Interpreter stdout]', stdout)
    })

    if (exec.error) {
        console.error('[Code Interpreter ERROR]', exec.error)
        throw new Error(exec.error.value)
    }

    return exec.results
}

async function chat(codeInterpreter: CodeInterpreter, userMessage: string): Promise<Result[]> {
    console.log(`\n${'='.repeat(50)}\nUser Message: ${userMessage}\n${'='.repeat(50)}`)

    const messages = [
        { role: 'system', content: SYSTEM_PROMPT },
        { role: 'user', content: userMessage }
    ]

    try {
        const response = await together.chat.completions.create({
            model: MODEL_NAME,
            messages: messages
        })

        const responseMessage = response.choices[0].message.content
        const codeBlockMatch = responseMessage.match(/```python\n([\s\S]*?)\n```/)

        if (codeBlockMatch && codeBlockMatch[1]) {
            const pythonCode = codeBlockMatch[1]
            console.log('CODE TO RUN')
            console.log(pythonCode)
            const codeInterpreterResults = await codeInterpret(codeInterpreter, pythonCode)
            return codeInterpreterResults
        } else {
            console.error('Failed to match any Python code in model\'s response')
            return []
        }
    } catch (error) {
        console.error('Error during API call:', error)
        throw error
    }
}
```

2️⃣ Python version:

```python
def code_interpret(e2b_code_interpreter, code):
    print("Running code interpreter...")
    exec = e2b_code_interpreter.notebook.exec_cell(
        code,
        on_stderr=lambda stderr: print("[Code Interpreter]", stderr),
        on_stdout=lambda stdout: print("[Code Interpreter]", stdout),
        # You can also stream code execution results
        # on_result=...
    )

    if exec.error:
        print("[Code Interpreter ERROR]", exec.error)
    else:
        return exec.results

client = Together(api_key=TOGETHER_API_KEY)

pattern = re.compile(
    r"```python\n(.*?)\n```", re.DOTALL
)  # Match everything in between ```python and ```


def match_code_blocks(llm_response):
    match = pattern.search(llm_response)
    if match:
        code = match.group(1)
        print(code)
        return code
    return ""


def chat_with_llm(e2b_code_interpreter, user_message):
    print(f"\n{'='*50}\nUser message: {user_message}\n{'='*50}")

    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": user_message},
    ]

    response = client.chat.completions.create(
        model=MODEL_NAME,
        messages=messages,
    )

    response_message = response.choices[0].message
    python_code = match_code_blocks(response_message.content)
    if python_code != "":
        code_interpreter_results = code_interpret(e2b_code_interpreter, python_code)
        return code_interpreter_results
    else:
        print(f"Failed to match any Python code in model's response {response_message}")
        return []
```

### 6. Upload the dataset

The CSV data is uploaded programmatically, not via AI-generated code. The code interpreter by E2B runs inside the E2B sandbox. Read more about the file upload [here](https://e2b.dev/docs/sandbox/api/upload).

1️⃣ JavaScript & TypeScript version:


```js
async function uploadDataset(codeInterpreter: CodeInterpreter): Promise<string> {
    console.log('Uploading dataset to Code Interpreter sandbox...')
    const datasetPath = './data.csv'

    if (!fs.existsSync(datasetPath)) {
        throw new Error('Dataset file not found')
    }

    const fileBuffer = fs.readFileSync(datasetPath)

    try {
        const remotePath = await codeInterpreter.uploadFile(fileBuffer, 'data.csv')
        if (!remotePath) {
            throw new Error('Failed to upload dataset')
        }
        console.log('Uploaded at', remotePath)
        return remotePath
    } catch (error) {
        console.error('Error during file upload:', error)
        throw error
    }
}
```

2️⃣ Python version:

```python
def upload_dataset(code_interpreter):
    print("Uploading dataset to Code Interpreter sandbox...")
    dataset_path = "./data.csv"

    if not os.path.exists(dataset_path):
        raise FileNotFoundError("Dataset file not found")

    try:
        with open(dataset_path, "rb") as f:
            remote_path = code_interpreter.upload_file(f)

        if not remote_path:
            raise ValueError("Failed to upload dataset")

        print("Uploaded at", remote_path)
        return remote_path
    except Exception as error:
        print("Error during file upload:", error)
        raise error
```

### 7. Put everything together

Finally we put everything together and let the AI assistant upload the data, run an analysis, and generate a PNG file with a chart. 
You can update the task for the assistant in this step. If you decide to change the CSV file you are using, don't forget to update the prompt too.

1️⃣ JavaScript & TypeScript version:


```js
async function run() {
    const codeInterpreter = await CodeInterpreter.create()

    try {
        const remotePath = await uploadDataset(codeInterpreter)
        console.log('Remote path of the uploaded dataset:', remotePath)

        const codeInterpreterResults = await chat(
            codeInterpreter,
            // Task for the model
            'Make a chart showing linear regression of the relationship between GDP per capita and life expectancy from the data. Filter out any missing values or values in wrong format.'
        )
        console.log('codeInterpreterResults:', codeInterpreterResults)

        const result = codeInterpreterResults[0]
        console.log('Result object:', result)

        if (result && result.png) {
            fs.writeFileSync('image_1.png', Buffer.from(result.png, 'base64'))
            console.log('Success: Image generated and saved as image_1.png')
        } else {
            console.error('Error: No PNG data available.')
        }

    } catch (error) {
        console.error('An error occurred:', error)
    } finally {
        await codeInterpreter.close()
    }
}

run()

```

2️⃣ Python version:

```python
with CodeInterpreter(api_key=E2B_API_KEY) as code_interpreter:
    # Upload the dataset to the code interpreter sandbox
    upload_dataset(code_interpreter)

    code_results = chat_with_llm(
        code_interpreter,
        "Make a chart showing linear regression of the relationship between GDP per capita and life expectancy from the data. Filter out any missing values or values in wrong format.",
    )
    if code_results:
        first_result = code_results[0]
    else:
        raise Exception("No code interpreter results")


# This will render the image
# You can also access the data directly
# first_result.png
# first_result.jpg
# first_result.pdf
# ...
first_result
```


### 8. Run the program and see the results

In the JS & TS version the resulting chart is saved to the same directory as a PNG file. In the Python version, the file is generated within the notebook. The plot shows the linear regression of the relationship between GDP per capita and life expectancy from the CSV data.


1️⃣ JavaScript & TypeScript version:

```js
> together-code-interpreter@1.0.0 start
> tsx index.ts

(node:21539) [DEP0040] DeprecationWarning: The `punycode` module is deprecated. Please use a userland alternative instead.
(Use `node --trace-deprecation ...` to show where the warning was created)
Uploading dataset to Code Interpreter sandbox...
Uploaded at /home/user/data.csv
Remote path of the uploaded dataset: /home/user/data.csv

==================================================
User Message: Make a chart showing linear regression of the relationship between GDP per capita and life expectancy from the data. Filter out any missing values or values in wrong format.
==================================================
CODE TO RUN
import pandas as pd
import matplotlib.pyplot as plt
from sklearn.linear_model import LinearRegression

# Load the data
data = pd.read_csv('/home/user/data.csv', delimiter=',')

# Filter out missing values or values in wrong format
data = data.dropna(subset=['GDP per capita (current US$)', 'Life expectancy at birth, total (years)'])

# Convert columns to numeric
data['GDP per capita (current US$)'] = pd.to_numeric(data['GDP per capita (current US$)'], errors='coerce')
data['Life expectancy at birth, total (years)'] = pd.to_numeric(data['Life expectancy at birth, total (years)'], errors='coerce')

# Filter out any remaining non-numeric values
data = data.dropna(subset=['GDP per capita (current US$)', 'Life expectancy at birth, total (years)'])

# Fit linear regression model
X = data['GDP per capita (current US$)'].values.reshape(-1, 1)
y = data['Life expectancy at birth, total (years)'].values.reshape(-1, 1)
model = LinearRegression().fit(X, y)

# Plot the data and the regression line
plt.scatter(X, y, color='blue')
plt.plot(X, model.predict(X), color='red')
plt.xlabel('GDP per capita (current US$)')
plt.ylabel('Life expectancy at birth, total (years)')
plt.show()
Running code interpreter...
codeInterpreterResults: [
...
...
...
Success: Image generated and saved as image_1.png
```

2️⃣ Python version:

```python
Uploading dataset to Code Interpreter sandbox...
Uploaded at /home/user/data.csv

==================================================
User message: Make a chart showing linear regression of the relationship between GDP per capita and life expectancy from the data. Filter out any missing values or values in wrong format.
==================================================
import pandas as pd
import matplotlib.pyplot as plt
from sklearn.linear_model import LinearRegression

# Load the data
data = pd.read_csv('/home/user/data.csv', delimiter=',')

# Clean the data
data = data.dropna(subset=['GDP per capita (current US$)', 'Life expectancy at birth, total (years)'])
data['GDP per capita (current US$)'] = pd.to_numeric(data['GDP per capita (current US$)'], errors='coerce')
data['Life expectancy at birth, total (years)'] = pd.to_numeric(data['Life expectancy at birth, total (years)'], errors='coerce')

# Fit the linear regression model
X = data['GDP per capita (current US$)'].values.reshape(-1, 1)
y = data['Life expectancy at birth, total (years)'].values.reshape(-1, 1)
model = LinearRegression().fit(X, y)

# Plot the data and the regression line
plt.scatter(X, y, color='blue')
...
plt.xlabel('GDP per capita (current US$)')
plt.ylabel('Life expectancy at birth, total (years)')
plt.show()
Running code interpreter...
```

![image](https://github.com/user-attachments/assets/392a9c6a-9e26-4408-bcc6-b8c324ca92a4)




## Resources
- [Code Interpreter SDK](https://github.com/e2b-dev/code-interpreter)
- [E2B docs](https://e2b.dev/docs)
- [E2B Cookbook](https://github.com/e2b-dev/e2b-cookbook/tree/main)


