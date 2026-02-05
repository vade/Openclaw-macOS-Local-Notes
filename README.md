# Openclaw-macOS-Local-Notes

AKA Adventures in Babysitting, or, the realities of local inference serving with commodity hardware.
AKA me want robo now.

## Overview

My goal stiving for:
* Running a secure agent, on local infra so im not leaking prompts or details to 3rd party API providers.
* Running a secure agent in a way that minimizes potential private data leaks, or the consequences of prompt injection or other agent targetted attacks. 
* Efficient model serving with a capable model that has a modicum of personality and enough smarts to use tools and be helpful. I dont want to pay an arm and a leg running high wattage gaming GPUs all day. M Series is fast enough in theory, and light weight power wise.
* A capable agent with working toolusage, i want it to actualy be able to interact with its available resources in a robust and reliable manner.
* A secure mobile comms channel with the agent that i have trust and control over. 

The above is a big ask and the guide will cover the nuances therein.
Note - this is not an OpenClaw install guide. This is more of an opionated walkthough of trying to reach the above goals on a budget without losing your mind.

## Intro 

You need resources to run:

#### 1. Openclaw Gateway Software

* Runs a local HTTP server to manage the Agent and OpenClaw features
* Is the os environment the Agent will be interacting with (file system, where tools run)
* To meet our goals, should not house senstive data 
* We should be comfortble with this environment getting trashed by accident by the agent
* The service infrastructure to configure the Agent and supporting infrastructure
* The service that runs the agents and connects to the Inference Server
* The agents workspace, which is the primary place the agent stores its working memory, output.
* The single largest surface area of shit that can go sideways

#### 2. An Inference Server
* Runs the ML Model the agent will use to think
* Runs an HTTP server that should support the features OpenClaw needs
* To meet our goals, should be capable hardware with enough GPU to run a model that meets our requirements.
* This is replacing our remote 3rd party hosted APIs like OpenAI, Anthropic, AND their cloud compute.
* The inference server is local or remote web server that is responsible for running one or many models that the agent uses to do its thinking.
* The inference server is INFRASTRUCTURE that needs to be running all the time for the agent to function. 
* The inference server doesnt need access to file system stuff, it doesnt run tools directly, its just a resource that does the thinking (inference) for the agent. 
* The inference server needs to have a fast GPU and a decent amount of memory to run capable models. 
These two resources need to see one another for the entire system to work.

## 1. Openclaw Gateway

Ideally, the Openclaw Gateway is isolated from anything important, like your credit card history, your social security info, and a photo of your passport. 
So part of our goal is to be able to limit Openclaw and give it a place where it can mess up freely with the minimum side effects. 
To that end, were going to run OpenClaw in a virtualized macOS system. 

Running UTM macOS VM is "cheap" as its native, but it uses memory for a whole new OS and the apps, plus storage.
You realistically want hardware with 32 GB of ram to give UTM 16GB or so. Good news is storage requirements are minimal on top of the OS. 

### The Inference Server

Now the nuance is that there are many different inference servers, and many models, and a lot of nuance to get right to have this work.

More capable models are larger, which require more ram, and more GPU horsepower. Smaller models are faster, require less memory, but are not as capable. 

Most of the guide will cover inference server nuances and how to configure OpenClaw for a specific server.

### The setup

Depending on your resources you can handle this a few ways:

* I have a single Mac:
    - I have data I care about?
      -  Run OpenClaw virtualized via UTM and the Inference Server simultaneously.
    - I dont have data I care about?
      - Run OpenClaw without virtualization, and save on memory overhead while running Inference Server simultaneously.
      
* I have more than one Mac:
  - Run Inference Server on Fastest Mac
  - Run OpenClaw with or without virtualization (up to you) on the other
  - Ensure they can talk
 
For my setup, ive got a M2 Mac Mini w 32GB of ram which i use as my dev workstation ( Machine 1 ), an one newer M4 Pro w 48 Gb of ram that was shared with colleagues (Machine 2).

Machine 1:
* Running UTM.app with a macOS virtual machine and storage, with a shared drive for data exchange.
* Install Openclaw on the VM
* Configure the VM Networking to be bridge mode, which lets it see and interact with your local network. That is a risk, but doable. 
 
Machine 2: 
* Dedicated Inference Server able to potentially serve other models for local systems in the office. 

### Models, Inference Serving

This setup requires carefuly chosing a model, a server, and confguring Openclaw correctly and ensuring the server supports features we need.

Our Inference Server should support:
* OpenAI Responses API, as it more modern than Completions API and lets the model stream responses, support tool calling and structured output, multimodal payloads, and keep things seperate from main chat context.
* Reliable Tool calling. Tool calling format is model dependent (amusingly some models want JSON, some XML, some other formats)
* Streaming support
* Reasoning support
* Optimized Inference
* Not buggy




Our Model needs to
* Fit in our GPU memory budget, room overhead for chat context
* Reliably use tools
* Be capable enough to reason about tool usage and follow instructions.


| Server | Completions API | Responses API | Backend | Distributed Inference | Notes | 
|--------|:-----:|:-----:| :-----:| :-----: | :-- |
| Ollama  | ✔️ | ️✔️ | ollama |  ❌ | ⚠️ Larger contexts / chat history was so slow we hit a 5 minute hard coded timeout in OpenClaws Node `fetch()` defaults that OpenClaw doesnt override in config. Setting OpenClaw `agents. reply.timeout` config option had no effect.<br/><br/>Otherwise works very if you are patient and dont need large contexts. <br/><br/>Had the most successful sequences of tool usage, but felt too slow to use as daily driver.| 
| LM Studio  | ✔️ | ✔️ | Llama CPP / MLX  | ❌ | ⚠️ [Tool usage bug](https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/189) - Does not fully support all models tool usage formats.<br/><br/>Very Fast with MLX.<br/><br/>Had best overall setup and usage experience, despite intermittent tool usage failure which messed up agent flow and OpenClaw Gateway. Want LM Studio to work |
| Local AI |✔️ | ❌  | Too many too list. |✔️ Swarm P2P IP|️  ⚠️[Buggy installer](https://discussions.apple.com/thread/253714860?answerId=257037956022&sortBy=rank#257037956022$0) - wont load on macOS, (not code signed) terrible macOS UI, doesnt work: `Error installing backend "mlx": not a valid backend: run file not found "/Users/developer/.localai/backends/mlx/run.sh"`. Buggiest software I have used in ages.
| EXO Labs  | ✔️ | ❌ | MLX  | ✔️ via JACCL via RDMA Thundernolt 5 | Very Fast inference esp with two machines. <br/>Not stable (could not get 24 hours uptime with a single node). <br/>Sometimes stuck hitting 100% GPU usage bug in macOS 26.2. Lack of Responses API means tool usage and reasoning is suboptimal | 
| vLLM-MLX | ✔️ |  ❌ | MLX | ❌ | ⚠️ No Responses API means models arent as performant with their context or tool usage |
| LLamaCPP Server | ✔️ |  ❌ | LLama CPP | ❌ | ⚠️ No Responses API means models arent as performant with their context or tool usage.<br/><br/>Doesnt support multi modal|
| MLX LM Server | ✔️ |  ❌ | MLX | maybe via mx.distributed config ? | ⚠️ No Responses API means models arent as performant with their context or tool usage.<br/><br/>Doesnt support multi modal
| MLX Openai Server  | ✔️ | ️❌ | MLX | ❌  | ⚠️ Doesnt use the model request param (it serves one model).<br/><br/>No Responses API means models arent as performant with their context or tool usage.<br/><br>Slow, even though using MLX?<br/><br> Not user friendly, requires deep knowlege of model specifics to get command line arguments correct.<br/><br/>Crashed kernel. |

### Models 


* Kimi K2 (only massive memory Mac Studios)
* GPT OSS
* Step 3.5 Flash
* Qwen 3 Coder Next
* Qwen 3 Next 
* Qwen 3 VL (this model has both vision and text backbone which is useful)
* GLM 4.7 Flash (text only, very capbable but MLX implementation currently has tool calling bug)
* GLM 4.6v FLash (Text and vision)


Testing GLM 4.7 local provider via macOS inference. 

Issues im tracking related to observed challenges.

https://github.com/openclaw/openclaw/issues/7725
https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/189
https://github.com/cubist38/mlx-openai-server/issues/181#issuecomment-3845354720
https://github.com/cubist38/mlx-openai-server/issues/189#issuecomment-3850780634
https://github.com/ml-explore/mlx-lm/pull/792
https://github.com/ggml-org/llama.cpp/issues/19138
