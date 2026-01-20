---
layout: post
title: "SDET. Running Ollama with Open WebUI: First Steps and Impressions"
subtitle: Exploring Local LLMs with a User-Friendly Interface
date:   2025-09-15 23:15:00 -0700
categories: sdet
tags: [ollama, deepseek, gpt-oss]
author: Yurii Chukhrai
---

## Introduction

Exploring new AI tools always starts with curiosity. I wanted to understand what it’s like to run large language models locally and how to make the process smoother.
Instead of using only cloud services, I decided to experiment with [Ollama](https://ollama.com) on my machine, and then connect it with a user-friendly interface.
To see which model I can use for the future AI agent (GitHub code review).

This post describes the steps I took: installing the Ollama CLI, downloading models, testing them in the terminal, and finally adding **Open WebUI** for a better experience.

## Installing Ollama CLI

The journey began by downloading and installing the Ollama CLI from the [official website](https://ollama.com/download).
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama --version
```
`ollama version is 0.11.11`

After installation, it became straightforward to pull models for testing.
```bash
ollama pull deepseek-r1:70b
ollama pull deepseek-r1:14b
ollama pull deepseek-v2:16b
ollama pull gpt-oss:20b

# See which models are available
ollama list
```
output:

```text
NAME               ID              SIZE      MODIFIED
gpt-oss:20b        aa4295ac10c3    13 GB     1 moment later
deepseek-v2:16b    7c8c332f2df7    8.9 GB    1 moment later
deepseek-r1:14b    c333b7232bdb    9.0 GB    1 moment later
deepseek-r1:70b    d37b54d01a76    42 GB     1 moment later
```

> ⚠️ **NOTE:** Like you can see I am using time unit `moment` just to replace it for article, since the model were downloaded couple weeks ago :). \
> And Apple M1 Ultra has 128 GB RAM, so I can run even 70B model (42 GB), but it will be slow (not dramatically ~ 10/15 seconds).

With just a few commands, I had several models ready to try. Each one varies in size and speed, so testing them side by side gave me flexibility.

---

## Running a Model
To check if everything was working, I tried a quick run:
```bash
ollama run gpt-oss:20b "To be or not to be?"
```
The response confirmed that Ollama was up and running. While the command line works well for simple prompts, I quickly realized it wasn’t the best environment for longer interactions.

![Question#1](/resources/sdet/2025-09-15-ollama-and-open-web-ui/images/question_01.jpg){: .mx-auto.d-block :}

---

## Adding a User Interface: Open WebUI

That’s when I decided to bring in [Open WebUI](https://github.com/open-webui/open-webui). Running it in Docker and connecting it to the local Ollama server gave me a clean and easy-to-use interface.

```mermaid
graph TD
    subgraph Host_Machine [Host Machine]
        style Host_Machine fill:#f9f9f9,stroke:#333,stroke-width:2px
        
        User[User / Browser]
        
        subgraph Docker_Environment [Docker Environment]
            style Docker_Environment fill:#e1f5fe,stroke:#0277bd,stroke-width:2px
            
            WebUI[Open WebUI Container]
            style WebUI fill:#4fc3f7,stroke:#01579b,color:white
            
            WebUI_DB[(WebUI Data Volume)]
            style WebUI_DB fill:#b3e5fc,stroke:#0288d1
        end
        
        subgraph Ollama_Service [Ollama Service Native]
            style Ollama_Service fill:#fff3e0,stroke:#ef6c00,stroke-width:2px
            
            Ollama_API[Ollama API Server]
            style Ollama_API fill:#ffb74d,stroke:#e65100,color:white
            
            Model_Runner[Model Runner]
            style Model_Runner fill:#ffe0b2,stroke:#fb8c00
            
            Local_Models[(Local Models Storage)]
            style Local_Models fill:#ffcc80,stroke:#f57c00
        end
    end

    User -- "HTTP" --> WebUI
    WebUI -- "HTTP API" --> Ollama_API
    WebUI -.-> WebUI_DB
    
    Ollama_API -- "Load Model" --> Model_Runner
    Model_Runner -- "Read Weights" --> Local_Models
    Model_Runner -- "Inference GPU/Metal" --> Host_Machine
```
The work diagram of Ollama and OpenWeb UI
{: style="text-align:center;"}


```bash
docker run -d -p 3000:8080 \
  -e WEBUI_AUTH=False \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

Opening `http://localhost:3000` in the browser made all the difference.


![Question#2](/resources/sdet/2025-09-15-ollama-and-open-web-ui/images/question_02.jpg){: .mx-auto.d-block :}

The answer in the second attempt different (interesting why? - float on GPU?).

## What do you know about me?

My favorite question to test models is: "What do you know about me (Yurii Chukhrai)?".

### gpt-oss:20b
![Question#6](/resources/sdet/2025-09-15-ollama-and-open-web-ui/images/question_06.jpg){: .mx-auto.d-block :}

### deepseek-v2:16b
![Question#3](/resources/sdet/2025-09-15-ollama-and-open-web-ui/images/question_03.jpg){: .mx-auto.d-block :}

### deepseek-r1:70b
![Question#4](/resources/sdet/2025-09-15-ollama-and-open-web-ui/images/question_04.jpg){: .mx-auto.d-block :}

More power for hallucinations, so sad :). I have master degree in Applied Physics, but I am not scientist, unfortunately.

---

## First Impressions

Switching from the terminal to the web interface felt like moving from a sketchpad to a proper workspace. \
Now I had: \
A clear history of prompts and responses; \
A more natural way to experiment with different models; \
A smoother workflow for longer conversations; \
The setup was simple, and the payoff was immediate.

---

## Summary

This experiment showed me that running AI locally is not only possible but also practical (hallucinations generations). With Ollama providing direct access to models and Open WebUI making interactions easier, I ended up with a flexible and user-friendly environment.
It gave me both control and convenience — a combination that makes exploring open-source models much more enjoyable.
