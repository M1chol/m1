---
layout: post
title: "Tiny LLMs on the Edge - Jetson Nano"
author: "Michał Kozłowski"
tags: Tutorials
excerpt_separator: <!--more-->
---

I went out and tested the Jetson Nano's capabilities for running local LLMs. <!--more--> This was originally part of the [Jetson Orin Nano - Getting Started](https://m1chol.github.io/m1/2025-09-22/jetson) guide, but the topic kept growing, so I decided to split it into two parts. In this post, I cover the current state of tiny LLMs for edge devices, as well as instructions for setting up the Jetson with `ollama`.
 
### Downloading `ollama`
To run LLMs easily, I will be using an open-source back-end solution called `ollama`. It integrates a package manager for models with a simple CLI (and even an API - more on that in future posts). You can install it quickly using the official shell script:
```
curl -fsSL https://ollama.com/install.sh | sh
```
To be honest, it didn’t work for me. For some reason, the download speed peaked at around 150 KB/s, and then after about 30 minutes I got `stream error: stream ID 1; PROTOCOL_ERROR; received from peer`. That being said, please try it out - if it works for you, you can skip the manual installation explained below.

If you have the same problem as I had, you’ll need to install `ollama` manually. For that, you need a capable download manager. I recommend `aria2c`:
```
sudo apt-get install aria2
```

Then go to the [latest Ollama GitHub release](https://github.com/ollama/ollama/releases/latest) and search for the file named `ollama-linux-arm64.tgz`. Right-click it and copy the URL. Next, `cd` into your downloads folder and execute this command (replace `<URL_HERE>` with the copied link): 
```
aria2c -x4 -s4 <URL_HERE>
```
When the download finishes, repeat the process for the `ollama-linux-arm64-jetpack6.tgz` file.

`aria2` sped up the download from 150 KB/s to about 600 KB/s and fixed the stream error! After the downloads complete, extract the packages first ensuring that the previous installation (if you tried the script) is completely removed.
```
sudo rm -rf /usr/lib/ollama
sudo tar -C /usr -xzf ollama-linux-amd64.tgz
sudo tar -C /usr -xzf ollama-linux-arm64-jetpack6.tgz
```
This will take a second, so please be patient.

After the packages are extracted, test if the installation was successful. To do this, run `ollama serve`, first exporting a special debug variable:
```
export OLLAMA_DEBUG=True
ollama serve
```
If you see lines containing:
```
CUDA driver version: 12.6
calling cuDeviceGetCount
device count 1
msg="detected GPUs" count=1 library=/usr/lib/aarch64-linux-gnu/nvidia/libcuda.so.1.1
```
this means that `ollama` has detected the Jetson’s GPU and will use it while generating model responses (see the full expected output [here](https://gist.github.com/M1chol/ca0c65bc571d9aee1d8f76708c8f4478)).

---
### Launching a model
If you haven’t already, start the `ollama` back-end:
```
ollama serve
```
and then in a second terminal window type:
```
ollama start <MODEL_NAME>
```
You can find all available models with their names on the [official Ollama page](https://ollama.com/search). But follow along if you want to see my recommendations :)

**Optional: Creating an `ollama` service**
If you want the `ollama` server to start alongside your system, you’ll need to create a service file. I recommend doing this if you’ve built a program that relies on `ollama`. If you only want to use it occasionally, skip this part, as it will consume resources and increase boot time.

Create a user and group for Ollama:
```bash
sudo useradd -r -s /bin/false -U -m -d /usr/share/ollama ollama
sudo usermod -a -G ollama $(whoami)
```

Create a service file in `/etc/systemd/system/ollama.service`:
```bash
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3
Environment="PATH=$PATH"

[Install]
WantedBy=multi-user.target
```

Then start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable ollama
```

---
### Deciding on what model to run
There are lots of criteria to consider when selecting a large language model, but you should mainly focus on:
- size of the model (in billions of parameters)  
- memory use  
- output speed (tokens/s)  
- thinking vs. non-thinking  

The first two are closely linked: the more parameters the model has, the more memory you’ll need to run it. On the `Jetson Nano`, we have about 8 GB RAM + SWAP. SWAP is a Linux term for a disk partition used as RAM backup. This theoretically means you could load up a 600 GB model (if SWAP size is large enough) and it would still “work.” But remember, reading from disk is MUCH slower than reading from RAM. In my case, it’s 68 GiB/s vs. 4 GiB/s, so RAM is about 17x faster than SSD. As you can guess, this heavily impacts output speed—which is our third point.

And finally: should you pick a thinking or a non‑thinking model? Thinking models have better problem‑solving capabilities but take longer to respond. They’re generally not great for a fast chat experience like ChatGPT, but they’re better for complex tasks such as math or programming.

**TL;DR**  
- The model should fit into available memory for best performance.  
- Choose a thinking model if you need maximum intelligence, even at the cost of responsiveness.  

[What is a thinking model?](https://medium.com/@sebuzdugan/thinking-llms-general-instruction-following-with-thought-generation-paper-explained-7cefb01edded)

---
## Performance of selected models
I tested several models on my hardware. In the table below you can see:
- tokens/s — output speed  
- CPU/GPU split — how much of the model is loaded onto RAM vs. SWAP  
- total memory used  
- multilingual capabilities — tested with Polish (my personal impressions)  
- general intelligence score (from [artificialanalysis.ai](https://artificialanalysis.ai/models/open-source))  

| Category          | Score                        |
| ----------------- | ---------------------------- |
| tokens/s          | more = better                |
| CPU/GPU           | more GPU% = better           |
| Total memory use  | less = better                |
| Intelligence score| more = better                |
| Multi-lingual     | good > simple > poor > none  |

> CPU/GPU stats were measured *after* memory optimization. See the next chapter for details.

| Model name     | Thinking | tokens/s | CPU/GPU | Total memory use | Intelligence score | Multi-lingual |
| -------------- | -------- | -------- | ------- | ---------------- | ------------------ | ------------- |
| gemma3:1b      | NO       | 12 -14   | 0%/100% | 1.9 GB           | 6                  | simple        |
| gemma3:4b      | NO       | 4 - 7    | 0%/100% | 6.3 GB           | 15                 | good          |
| gemma3n:e2b    | NO       | 4 - 7    | 0%/100% | 6.3 GB           | 8                  | good          |
| exaone4.0:1.2b | YES      | 11 - 15  | 15%/85% | 8.2 GB           | 27                 | none          |
| qwen3:1.7b     | YES      | 9 - 11   | 0%/100% | 2.3 GB           | 22                 | poor          |
| qwen3:4b       | YES      | 4 - 6    | 0%/100% | 4.1 GB           | 26                 | simple        |
| phi4-mini      | NO       | 5 - 6    | 0%/100% | 3.9 GB           | 16                 | simple        |
| deepseek-r1:8b | YES      | 3 - 4    | 0%/100% | 5.6 GB           | 19                 | poor          |

Special Polish-language-trained models:

| Model name       | tokens/s | CPU/GPU | Total memory use | Intelligence score |
| ---------------- | -------- | ------- | ---------------- | ------------------ |
| bielik-4.5b-v3.0 | 5–6      | 0%/100% | 6.0 GB           | unknown            |

If you want to replicate my tests, you can run `ollama ps` (while the model is loaded) to check the CPU/GPU split:  
- `100% GPU` = model fully in GPU memory  
- `100% CPU` = model fully in RAM  
- `48%/52% CPU/GPU` = model split between GPU and RAM  

To get the tokens/s statistic, run:
```
ollama run <MODEL_NAME> --verbose
```

---
## Memory optimization
Running LLMs takes a lot of RAM. I wouldn’t have been able to run `gemma3:4b` entirely on the GPU without clearing RAM by:
- disabling the desktop  
- turning off miscellaneous services  

To temporarily disable the desktop on Jetson, run:
```
sudo init 3
```
To re-enable it:
```
sudo init 5
```

> After disabling the desktop you’ll have to sign in via terminal. While in terminal mode you can switch sessions using `Alt+F2`, `Alt+F3`, etc.

Disable unnecessary services:
```
sudo systemctl disable nvargus-daemon.service
```

Check memory usage with:
```
free -m
```

---
## My Model Recommendations
From my tests, I got the best results with **gemma3**, with the 4‑billion‑parameter version fitting the Jetson perfectly. An added bonus is that **gemma3** is multi‑modal, so it can also accept images as input—not just text. **Gemma** was also the clear winner in the multilingual category, responding clearly in Polish and rarely making typos or grammatical mistakes.

In the thinking category, I would pick `qwen3` instead of `exaone`. **Exaone** was less reliable, and on some occasions it started looping or simply ignored my messages.

I’ll explore function‑calling capabilities of these models in a future post.