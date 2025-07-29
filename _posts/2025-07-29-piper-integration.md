---
layout: post
title: "Text to speech in python"
author: "Michał Kozłowski"
tags: Tutorials
excerpt_separator: <!--more-->
---

I'm working on integrating a speech synthesizer to my home assistant project. I decided to use piper library in python. <!--more--> I created a simple example of how to use [piper](https://github.com/OHF-Voice/piper1-gpl) python API to generate Polish audio from text. If you are interested in different languages the library supports over 40 of them (Check out [those examples](https://rhasspy.github.io/piper-samples/)). This library does not need much computational power so it should work on edge devices like `Raspberry PI` or even better Nvidia's `Orin NANO`. Latter supports `cuda` so will work even better.

## Installation

[Installation](https://github.com/OHF-Voice/piper1-gpl/blob/main/docs/CLI.md) is very straight forward, you can download `piper-tts` pypi package using pip.

```
python -m venv .venv
source .venv/bin/activate
pip install piper-tts
```

then download the model

```
python3 -m piper.download_voices pl_PL-darkman-medium
```

you can quickly test everything

```
python3 -m piper -m pl_PL-darkman-medium -f test.wav -- 'To jest test!'
```

## Streaming

There is no working example provided in the documentation, so I had to improvise. I want the assistant to be as responsive as possible, so streaming the response is important. I don’t want to wait until all of the text has been fully generated. The code provided below waits for the user to type in the message, and then reads it out to speakers.

```python
import numpy as np
import sounddevice as sd
from piper.voice import PiperVoice, SynthesisConfig

# Setup
model = "./pl_PL-darkman-medium.onnx"
print(f"Loading voice model from: {model}")
voice = PiperVoice.load(model)
print("Voice loaded successfully.")

syn_config = SynthesisConfig(
            length_scale=1.2,  # Normal speed
            noise_scale=0.8,   # Moderate variation
            noise_w_scale=0.8, # Moderate speaking variation
            volume=0.5
        )

stream = sd.OutputStream(
    samplerate=voice.config.sample_rate,
    channels=1,
    dtype="int16",
)


stream.start()

text = input("> ").strip()
while text != "quit":
    for audio_chunk in voice.synthesize(text, syn_config=syn_config):
        raw_audio_bytes = audio_chunk.audio_int16_bytes
        # Convert the raw bytes to a NumPy array
        int_data = np.frombuffer(raw_audio_bytes, dtype=np.int16)
        stream.write(int_data)
    text = input("> ").strip()

stream.stop()
stream.close()
print("Program stopped")
```

## Performance

I haven't yet tested the performance on `RPI` or `Orin`  but I will update this post as soon as I get to it. On my PC on the other hand without using `cuda`the response is instant. I'm satisfied with this result, other then quality of the voice there is no room for improvement.
