---
title: "My Strix Halo"
date: 2026-06-27
draft: false
description: "128GB of unified memory for under $2k sounded like the deal of the year. Then I tried to actually use it for LLMs."
tags: ["strix-halo", "rocm", "llama-cpp", "local-llms"]
---

It started, like a few of my worst ideas, with an end-of-year email from HP.

A Z2 Mini G1a — AMD's Strix Halo in a small quiet box — on promotion. The 128GB configuration landed around $2,000. I did the only math that mattered at the time: 128GB of LPDDR5X was roughly $800 on its own at the end of 2025, so the CPU, the NPU, and the whole rest of the machine were basically free. What did I know.

## The dream

My frame of reference was an NVIDIA RTX 2080 — 8GB of VRAM and several happy years of `ollama` and ComfyUI. But I was always bumping the ceiling. Anything bigger than a 7B meant offloading to system RAM and watching tokens drip out in single digits per second.

The Strix Halo pitch is *unified memory*: the integrated Radeon shares the system's 128GB, so in theory you get a GPU with an enormous framebuffer without buying an enormous GPU. A local box that could run something real, not just toys. That was the hook.

## The wall

Then came the software.

Strix Halo's integrated GPU is **gfx1151** — RDNA 3.5, a brand-new ISA — and ROCm, AMD's compute stack, did not love it. I'd never owned an AMD GPU. On NVIDIA the path is paved: CUDA, cuDNN, everything links. On AMD, in late 2025, on silicon that fresh, the path was more of a *suggestion*.

Two problems stacked on each other. First, getting the memory to the GPU at all: the BIOS hands the iGPU a fixed chunk of RAM as a framebuffer, and the default was stingy. Cranking it toward 96GB so models could actually *live* in VRAM was its own afternoon of firmware menus and reboots. Second, even once the memory was there, ROCm's kernels for gfx1151 were either missing, broken, or only existed in a nightly build that hadn't been QA'd against anything I owned.

## The detour

I gave up on ROCm for a while and pointed `llama.cpp` at Vulkan. Vulkan is the great equalizer — if your GPU has a driver, `llama.cpp` can usually run on it. And it did. Models loaded, tokens came out.

But integrated GPUs are weird about memory. Vulkan would either over-allocate device-local memory and stall, or fall back to host-visible memory and crawl. The numbers moved, but they moved erratically, and getting a big model to sit cleanly in that unified pool felt like balancing a broom on my palm.

## The way through

So back to ROCm. This time I stopped trying to install it on the host.

I pulled the official ROCm Docker images — and when those didn't agree with gfx1151, the nightly ones — and ran `llama.cpp`'s ROCm (HIP) backend inside the container, passing the render devices through:

```bash
docker run -it --rm \
  --device=/dev/kfd --device=/dev/dri \
  --group-add video --group-add render \
  --shm-size 16G \
  -v "$PWD":/work -w /work \
  rocm/pytorch:nightly \
  ./llama-server -m model.gguf -ngl 999 --host 0.0.0.0
```

Containers are usually how I ship things; here they were how I *survived* things. If a nightly blew up, I threw it away and pulled the next. The host stayed clean. Unified memory feeding HIP, in a throwaway container, models finally landing where they were supposed to.

That was the setup that held.

## It wasn't always like this

Once the box could finally load a model, I thought the hard part was over. It wasn't.

The Qwen models fought me in a completely different way. Their tool-call format is *special*, and the worst of it was that they'd embed the tool calls *inside the thinking stream* — a half-formed reasoning trace that would suddenly turn into a function call mid-sentence. The engine would hand that to the harness, and the harness would choke. Nothing on either end could agree on what it was even looking at. It failed. A lot.

So I wrote my own middleware — a shim that sits between the engine and the harness and rewrites the stream on the fly, teasing the tool calls back out of the thinking and reshaping them into something everyone downstream can parse. A little duct tape to make everybody happy. :-)

There were stretches where that duct tape was the only thing between me and giving up. I genuinely got frustrated enough to price the machine out. I asked Google what I could get for it.

And here's the part I didn't expect: the quote was *good*. Good enough to stop me short. Because when I'd bought the thing I'd tacked on three years of HP onsite warranty, and that transferable warranty made the resale value pop. I'd been about to sell a machine that was, by the market's own math, worth keeping.

So I got stubborn instead. I decided I was going to make this machine work — warranty and all.

## The payoff

The first model I put to real work was `qwen-coder-next`. Not as a chatbot — as a colleague.

I had a project with a couple of genuinely nasty race conditions and some deadlock-prone concurrency paths, and I pointed the model at them and asked it to explain what it saw. It caught things I'd stopped being able to see because I'd been staring too long. That's the moment a local box earns its keep: when the loop is tight enough that you ask the model something, it answers in the time it takes to sip your coffee, and you ask again — no rate limit, no token meter, nothing leaving the room.

We're lucky with the weights right now. **Qwen3.6** and **Gemma 4** are both genuinely good, and both fit comfortably in 96GB of VRAM with room to spare. The choice isn't *"what can I squeeze in"* anymore; it's *"what do I actually like working with."*

## Right now

These days the numbers are something I'd have laughed at a year ago: **north of 25 tokens a second** on Qwen3.6 — MoE or dense, with MTP — and a steady **15 t/s** running DS4 with DeepSeek 4 Flash.

And the stack just clicks. I run @badlogic's **pi** with @antirez's **DS4** (Dwarfstar) on **DeepSeek 4 Flash**, and I couldn't be happier. Everything plays nicely, Flash flies on the unified memory, and for the first time since I bought the thing I'm spending my time on the *work* instead of on the *box*.

It paid off.

---

So — was it the deal of the year? In dollars, probably yes. In hours, I paid full price. I don't regret either.

I've got a quiet little brick on the desk that runs real models locally, doesn't phone home, and taught me more about the AMD compute stack in six months than a year of reading would have. The matrix is open, and a fair chunk of it lives in my 128GB now.
