# OSINT Agentic Workflow

***

## Goal
I want to test out the agentic AI workflow and see how I can implement that into my OSINT workflow. My design idea is to have a central orchestrator but multiple AI Agents. Each agent will be specialized for a specific task, like one for email osint, one for domain osint, etc. The orchestrator will use a routing script that determines based on the prompt which agent to call up. Each agent will use tools and APIs specific to their design.

## Test 1 - Email Agent
I have 3 sources I use a lot for email OSINT, Emailrep.io, Hunter.io and HaveIBeenpwned. I have API keys for all of these so I want to make an email micro agent that takes the input of an email address and does the API calls and gets data. This would then be normalized and sent to an Ollama Model for analysis and confidence scoring. It maps to something like this:
```
User Input
    ↓
Email OSINT Agent
    ↓
API Tool Calls
    ├── HIBP
    ├── Hunter
    └── EmailRep
    ↓
Normalized Results
    ↓
Ollama Local LLM
    ↓
OSINT Summary
```

## Environment
A Linux machine

## Folder Structure
```
email-osint-agent/
│
├── agents/
│   └── email_agent.py
│
├── tools/
│   ├── hibp_tool.py
│   ├── hunter_tool.py
│   └── emailrep_tool.py
│
├── main.py
├── .env
├── requirements.txt
└── .gitignore
```

## Install Dependencies
First we start with installing Ollama
```
curl -fsSL https://ollama.com/install.sh | sh
```
Next we will pull a light model to test with but then eventually move to a bigger model like LLama 3
```
ollama pull phi3:mini
```
Why this model?
1. lightweight
2. fast
3. ideal for orchestration
4. good enough for summarization

Next we start Ollama
```
ollama serve
```
Next we navigate to our email-osint-agent folder and install a Python virtual environment, so we dont have dependency conflicts
```
python3 -m venv venv
source venv/bin/activate
```

Now we make our requirements file to do pip installs
```
nano requirements.txt
requests
python-dotenv
ollama
rich
Ctrl +x/Y/Enter
```
Now we install these:
```
pip3 install -r requirements.txt
```

We do not want to hardcode our API keys in our scripts so we make a separate .env file and put our keys in there and pull them at runtime
```
nano .env

HIBP_API_KEY=YOUR_KEY
HUNTER_API_KEY=YOUR_KEY
EMAILREP_API_KEY=YOUR_KEY
```

## Build Tool Wrappers
These are NOT AI. These are deterministic data collection modules. This is one of the most important concepts in agent engineering. These will be used to pull what data I need from the API's. After that its normalized and sent to an AI.

### HaveIBeenPwned
Let's start with our HaveIBeenPwned tool
```
nano tools/hibp_tool.py
```
Then put in this code:
```
import os
import requests
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv("HIBP_API_KEY")

def check_breaches(email):

    headers = {
        "hibp-api-key": API_KEY,
        "user-agent": "email-osint-agent"
    }

    url = f"https://haveibeenpwned.com/api/v3/breachedAccount/{email}"

    try:

        response = requests.get(
            url,
            headers=headers,
            timeout=15
        )

        if response.status_code == 200:
            return response.json()

        elif response.status_code == 404:
            return []

        else:
            return {
                "status": response.status_code,
                "error": response.text
            }

    except Exception as e:
        return {
            "error": str(e)
        }
```

Let's breakdown what this code is doing in sections:
```
import os
import requests
from dotenv import load_dotenv
```
These load:
1. environment variables
2. HTTP client
3. .env loader

***


```
load_dotenv()
```
This loads in my environment variables from the .env file

***


```
API_KEY = os.getenv("HIBP_API_KEY")
```
This one retrieves the HIBP_API_KEY value from my .env file

***


```
def check_breaches(email):
```
Def in Python represents a function. This is the callable tool. The agent we will make later invokes this function. As I mentioned, there will be a central orchestrator that will call a specific agent, and this agent will know here to call this function from this specific tool.

***


```
headers = {
    "hibp-api-key": API_KEY,
    "user-agent": "email-osint-agent"
}
```
When we build a curl command to reach the API, there are some required headers that need to get inserted into this curl statement. HIBP API calls require these two headers.

***


```
url = f"https://haveibeenpwned.com/api/v3/breachedAccount/{email}"
```
This is the actual API endpoint we need to reach out to for getting the data we need. Be aware of the capitalization of breachedAccount, this is specific in the HIBP documentation.

***


```
timeout=15
```
HIBP is strict on rate limiting. Without this the API can hang forever and the whole agent stalls.

***

```
try:
``` 
In Python the Try/Catch block is used for error handling. This one prevents the entire agent from crashing if API fails.


### Hunter
Now we move onto the Hunter Tool script
```
nano tools/hunter_tool.py
```
Then put this script in:
```
import os
import requests
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv("HUNTER_API_KEY")

def lookup_email(email):

    url = "https://api.hunter.io/v2/email-verifier"

    params = {
        "email": email,
        "api_key": API_KEY
    }

    try:

        response = requests.get(
            url,
            params=params,
            timeout=15
        )

        return response.json()

    except Exception as e:
        return {
            "error": str(e)
        }
```

Most of the explanation of the script is the same as the HIBP but there is one that's different:
```
params=
```
Hunter API uses parameters instead of headers. Python automatically converts:
```
params = {
    "email": email
}
```
Into:
```
?email=test@example.com
```

### Emailrep
Now we move to our Emailrep tool
```
nano tools/emailrep_tool.py
```
Put this script in
```
import os
import requests
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv("EMAILREP_API_KEY")

def reputation_check(email):

    headers = {
        "Key": API_KEY
    }

    url = f"https://emailrep.io/{email}"

    try:

        response = requests.get(
            url,
            headers=headers,
            timeout=15
        )

        return response.json()

    except Exception as e:
        return {
            "error": str(e)
        }
```
The same explanation as HIBP tool script.

That concludes our Tool Wrappers. Conceptually:

Tool Wrapper
1. Abstracts external service
2. Provides clean reusable interface

The agent doesn’t care:
1. HOW the API works
2. WHAT auth method it uses
3. WHAT headers it needs

The tool wrapper handles that complexity. This abstraction is foundational in agent systems.

## Build the Email Micro-Agent
This will be the actual agent being called by some kind of orchestrator.
```
nano agents/email_agent.py
```
Put in this script:
```
from tools.hibp_tool import check_breaches
from tools.hunter_tool import lookup_email
from tools.emailrep_tool import reputation_check

class EmailOSINTAgent:

    def investigate(self, email):

        results = {
            "email": email,
            "hibp": check_breaches(email),
            "hunter": lookup_email(email),
            "emailrep": reputation_check(email)
        }

        return results
```

Let's break down this script.
```
from tools.hibp_tool import check_breaches
```
These are what's called imports. This is importing specialized tools.

***


```
class EmailOSINTAgent:
```
This is a Class Definition. In Python, a Class is essentially a blueprint or a template for creating Objects. If you think of an object as a house, the class is the architectural blueprint. It defines the structure, what characteristics the house will have (like doors, windows, and walls), and what it can do, but it isn’t an actual house itself. You use that blueprint to build as many individual houses (objects) as you want.

This defines a specialized micro-agent. Eventually we will want a fleet of these agents, as an example:
1. CVEAgent
2. CompanyAgent
3. DNSAgent
4. GitHubLeakAgent

***


```
def investigate(self, email):
```
This is the agent’s workflow. Investigate the email address supplied.

***


```
"hibp": check_breaches(email),
```
The agent coordinates multiple tools. This is orchestration.


## Main Application
This is what we will run to utilize all we have built to give us a result.
```
nano main.py
```
Put this script in
```
import time
import ollama
from agents.email_agent import EmailOSINTAgent
from rich import print

agent = EmailOSINTAgent()

email = input("Enter email: ")

print("[cyan]Starting investigation...[/cyan]")

start = time.time()

results = agent.investigate(email)

print(f"[green]API collection done in {time.time() - start:.2f}s[/green]")

hibp_breaches = []

if isinstance(results["hibp"], list):

    hibp_breaches = [
        x.get("Name", "Unknown")
        for x in results["hibp"]
    ]

summary_data = {
    "email": email,
    "breach_count": len(hibp_breaches),
    "breaches": hibp_breaches,
    "hunter_status": results["hunter"].get("data", {}).get("status"),
    "deliverable": results["hunter"].get("data", {}).get("result"),
    "emailrep_reputation": results["emailrep"].get("reputation"),
    "emailrep_suspicious": results["emailrep"].get("suspicious")
}

print(summary_data)

prompt = f"""
You are an OSINT analyst.

Provide:
1. Risk summary
2. Key findings
3. Confidence assessment

DATA:
{summary_data}
"""

print("[yellow]Sending to Ollama...[/yellow]")

llm_start = time.time()

response = ollama.chat(
    model="phi3:mini",
    messages=[
        {
            "role": "user",
            "content": prompt
        }
    ]
)

print(f"[green]LLM completed in {time.time() - llm_start:.2f}s[/green]")

print(response["message"]["content"])
```

Let's breakdown this script

```
import time
```
Imports Python’s timing module. We will use it to measure API execution time and LLM inference time. 

***


```
import ollama
```
Imports the Python client for Ollama. This is your bridge between Python and the local LLM server and model execution (phi3, llama3, etc.). Without this, you cannot call the model.

***


```
from agents.email_agent import EmailOSINTAgent
```
Imports your custom micro-agent class. This is your tool orchestration layer. It abstracts:
1. HIBP calls
2. Hunter calls
3. EmailRep calls

So main.py does NOT need to know API details.

***


```
from rich import print
```
Replaces standard print() with prettier terminal output.

***


```
agent = EmailOSINTAgent()
```
Instantiates the micro-agent we created earlier. Because its a Class, it basically creates an instance of your email OSINT agent.

***


```
email = input("Enter email: ")
```
User provides investigation target. When the script is run, it will prompt us with the words Enter email: when we put the email in and hit enter that now becomes the value for the variable email=

***


```
print("[cyan]Starting investigation...[/cyan]")
```
Just a printout of the current status of the task. 

***


```
start = time.time()
```
Captures the current UNIX timestamp. Used to measure how long API calls take.

***


```
results = agent.investigate(email)
```
This is the core of the system. Inside EmailOSINTAgent.investigate():
```
HIBP API call
    ↓
Hunter API call
    ↓
EmailRep API call
    ↓
return combined dictionary
```
The output would look like:
```
{
    "email": "...",
    "hibp": [...],
    "hunter": {...},
    "emailrep": {...}
}
```
This is your tool execution layer. No AI involved yet.

***


```
print(f"[green]API collection done in {time.time() - start:.2f}s[/green]")
```
Calculates current_time - start_time. An example of the output may be: "API collection done in 4.78s"

***


```
print(results)
```
Prints the full raw dictionary. This is CRITICAL during development because it lets you see:
1. API response structure
2. missing fields
3. unexpected formats

***


```
hibp_breaches = []
```
Creates an empty list. You will fill this with cleaned breach names. This is part of data normalization.

***


```
if isinstance(results["hibp"], list):
```
Checks: "Is HIBP output a list?". This prevents crashes because HIBP can return:
1. list (breaches found)
2. empty list
3. error dict

***


```
hibp_breaches = [
    x.get("Name", "Unknown")
    for x in results["hibp"]
]
```
Loops through each breach entry and extracts the Name field value. So if the JSON result was:
```
{
  "Name": "LinkedIn"
}
```
The output would be:
```
["LinkedIn"]
```
This is data reduction, essential for LLM efficiency.

***


```
summary_data = {
    "email": email,
    "breach_count": len(hibp_breaches),
    "breaches": hibp_breaches,
    "hunter_status": results["hunter"].get("data", {}).get("status"),
    "deliverable": results["hunter"].get("data", {}).get("result"),
    "emailrep_reputation": results["emailrep"].get("reputation"),
    "emailrep_suspicious": results["emailrep"].get("suspicious")
}
```
This is where you convert messy API JSON blobs into a small, structured, AI-friendly dataset. For example:
```
"email": email,
```
Just stores the original input.
```
"breach_count": len(hibp_breaches),
```
Counts number of breaches found. 
```
"breaches": hibp_breaches,
```
Stores only simplified breach names, like ["LinkedIn", "Adobe", "Dropbox"]

***


```
"hunter_status": results["hunter"].get("data", {}).get("status"),
```
1. results["hunter"]  accesses Hunter API response.
2. .get("data", {}) means if 'data' exists, use it. Otherwise use empty dict {}.
3. .get("status") Extracts: valid, invalid, accept_all, unknown

***


```
"deliverable": results["hunter"].get("data", {}).get("result"),
```
Indicates if email can receive mail. Possible values:
1. deliverable
2. undeliverable
3. risky
4. unknown

***


```
"emailrep_reputation": results["emailrep"].get("reputation"),
```
Returns reputation classification. Typical values:
1. low
2. medium
3. high

This gives behavioral trust scoring. You now combine the following into risk inference later:
1. breach exposure
2. deliverability
3. reputation

***



```
"emailrep_suspicious": results["emailrep"].get("suspicious")
```
Boolean flag:
1. True = suspicious behavior detected
2. False = normal profile

Very useful for phishing likelihood and fraud detection heuristics

***



```
print(summary_data)
```
Displays the cleaned dataset BEFORE AI processing. This confirms:

1. API layer works - no missing fields and no crashes
2. normalization works - data is reduced properly
3. LLM input is controlled - you are NOT feeding raw JSON anymore

Example output would be:
```
{
    'email': 'test@example.com',
    'breach_count': 3,
    'breaches': ['LinkedIn', 'Adobe'],
    'hunter_status': 'valid',
    'deliverable': 'deliverable',
    'emailrep_reputation': 'medium',
    'emailrep_suspicious': False
}
```

***


```
prompt = f"""
You are an OSINT analyst.

Provide:
1. Risk summary
2. Key findings
3. Confidence assessment

DATA:
{summary_data}
"""
```
This creates the “instruction package” for the model. 
1. You are an OSINT analyst - This tells the model to adopt an analyst mindset and to prioritize security interpretation.
2. Next we see the Task Definition:
```
Provide:
1. Risk summary
2. Key findings
3. Confidence assessment
```
This structures output. Without this responses become inconsistent, hard to parse and noisy.
3. Next is the Data Injection:
```
DATA:
{summary_data}
```
This passes structured input. 

***


```
print("[yellow]Sending to Ollama...[/yellow]")
```
This is your pipeline boundary. This moves from a Deterministic system to a Probabilistic system. Up until now we have seen Python logic, API calls and structured transformations. Now we need LLM inference to begin. 

***


```
llm_start = time.time()
```
LLM timer start. Marks the start of inference timing. This is so we can measure:
1. model speed
2. VM performance
3. model size impact

***


```
response = ollama.chat(
    model="phi3:mini",
    messages=[
        {
            "role": "user",
            "content": prompt
        }
    ]
)
```
This is the Ollama Chat call.
1. model="phi3:mini" - That selects the model to be used. 
2. messages=[...] - This is the messages format. In this script user = input prompt and assistant = model output.
3. "content": prompt - This sends This sends instructions and structured OSINT data.

***



```
print(f"[green]LLM completed in {time.time() - llm_start:.2f}s[/green]")
```
This measures inference speed, model efficiency and system bottlenecks. An example of the output would be:
```
LLM completed in 12.43s
```

***

```
print(response["message"]["content"])
```
This is the final output print. This extracts:
```
response = {
    "message": {
        "content": "AI-generated OSINT report"
    }
}
```
The reason we want this is because Because Ollama returns:
1. metadata
2. tokens
3. message wrapper
4. model info

And we just want 'content'

***

## Summary Pipeline
```
1. User input
        ↓
2. API collection (HIBP / Hunter / EmailRep)
        ↓
3. Data normalization (summary_data)
        ↓
4. Prompt construction
        ↓
5. Ollama inference
        ↓
6. OSINT report output
```

Some possible next steps would be:
1. real multi-agent router 
2. tool registry system
3. async API execution (massive speedup)
4. parallel OSINT workers
5. a proper "investigation graph engine"

To test it it run:
```
python3 main.py
```











































