---
layout: post
title: "Getting started with AI image generation"
author: "Michał Kozłowski"
excerpt_separator: <!--more-->
---

## Generating images with Stable Diffiusion
Do you want to generate images with AI, but you have no idea how to start? Read this.
<!--more--> 
## Before we begin
This is post is targeted at begginers, If you worked before with Stable Diffusion I recommend other sources.

## What is stable diffiusion
Stable diffiusion is a deep learning text to image model. Under the hood it is trained to denoise images, but if you reverse the process you can get images from the noise (this is called latent diffusion). If you want to know more about it check the [wiki](https://en.wikipedia.org/wiki/Latent_diffusion_model). To generate images on your machine you need to have a strong GPU. The most important aspect is `VRAM` or virtual ram. I would say that 6 GiB is **miminimum**. If you don't know what specs is your GPU search the internet. The more VRAM the betther that is because to run a model you need to completly load it to the memory.Without getting into details lets justs say that some of models are pretty big. 

## How to generate images
OK, so you have a right GPU, what's next? You need the right tools. If you did some research, you probably found those two:
1. [Automatic1111](https://github.com/AUTOMATIC1111/stable-diffusion-webui) - stable diffusion webui
2. [ComfyUI](https://github.com/comfyanonymous/ComfyUI) - Node based generation

As great as those tools are I wouldn't call them hassle free. So if you want to go the easy way chek out this Krita plugin:

3. [Krita plug-in for stable diffusion](https://github.com/Acly/krita-ai-diffusion)

You have a trade-off here, between controll and ease of use. By far krita plug-in is easiest to set up and generating images is just a press of a button. But if you want to have more controll over the image go for `Automatic1111`. I really wouldn't recommend `ComfyUI` for begginers but noone is stopping you, and you get most freedom with this one.

**Installing**
I won't get into much detail. If you want to install any of those follow install instructions provided in `README` files. But for all of those be sure to install proper version of python beforehand.

## First image
Before you generate an image you need a prompt. you usually have to prompts
1. Positive prompt
2. Negetive prompt

I think this is self explenatory, positive is what you want and negitive what you don't. Lets try `cinematic landscape, great moutain` for positive, and negetive you can leave empty for now. After you click generate wait for the image to show up. In `Automatic1111` and `ComfyUI` you also have controll of `seed`. By default each generation you do sets new random seed, this results in very different image each time. But if you for example found a great camera angle, or a good pose for your character, you can freeze the `seed`. Next images you generate will be very similar. 

## How to generate good images
Let's start from the most important thing, the model you use. I belive every one of those tools come preinstalled with `v1-5-pruned-emaonly.safetensors` which is something... but if you want to get good results you need a good model. The best site to search for Stable diffiusion models is [Civit.ai](https://civitai.com/). I usually go for:
- [Juggernaut XL](https://civitai.com/models/133005/juggernaut-xl)
- [Realistic Vision](https://civitai.com/models/4201?modelVersionId=501240)

Even if you want to generate stylized images (I'll get to that later) I advise you to get realistic good model first. Here is the `v1-5-pruned-emaonly.safetensors` vs `Juggernaut XL` (same settings) prompt: `cinematic landscape, great moutain`.

![img]({{ "assets/images/stable-diff1.jpg" | relative_url }})

You can see the diference between models.

**Main settings**
When generating images you have two settings that are the most important.
1. Sampling rate
2. CFG Scale

Sampling rate controlls how many times the model will "enchance" the image. More steps equals more quality, that is before the image becomes distorted. Some models can handel more steps before destorting the image. Also don't forget that more steps means more time to generate. For me the sweet spot is around 45 - 75 depending on usecase.

![img]({{ "assets/images/stable-diff2.jpg" | relative_url }})

CFG Scale is a parameter that conrolls how closely a model will follow your prompt. Usually you need to set it once, for every model. Usually in the 4 - 7 range rarerly more.

![img]({{ "assets/images/stable-diff3.jpg" | relative_url }})

## Images with style
If you want something more artistic or specific, you need to check LoRA's. Lora (Low-Rank Adaptation) is a training technique for fine-tuning Stable Diffusion models. It is basicly a way for a model to understand new concepts or styles. All you need to do is download the additional small model and put it in corresponding folder. Here are some examples:
- [Lora for porche 911](https://civitai.com/models/647663/porsche-911-gts-2024-flux)  - teaches model concept of specific car
-  [Il était une FOIS](https://civitai.com/models/631617/style-lora-il-etait-une-fois) - Artistic style of the French educational animation.

## And that's all
Not really. But you have very basic understending of core concepts. You sucessfully created your first image, and know some basic settings for image generations.

[full image from thumbnail](https://civitai.com/images/49681585)