# llm.metal
This is a fork of [Andrej Karpathy's llm.c](https://github.com/karpathy/llm.c) but ported to run using Metal Compute Shaders (not MPS except for GEMM) instead of CUDA. 

Right now only the forward pass is implemented. Things are still very much a work in progress and catching up to where things are on the CUDA side of things. For my system (M2 Max 12-core 96GB) the CPU code with OpenMP enabled is running at about ~460ms per batch (64 sequence len backward/update commented out) and the metal version is around 72ms per step. There is still a lot of low-hanging fruit in terms of optimization, too. I anticipate it should be possible to get things to about 40ms per step (forward only) and around 80-90ms for forward/backward/update. This is about what pytorch seems to be from what I can tell. Pytorch's MPS backend is using MPS Graph framework I believe.

## Quick Start (Metal)
Right now only Apple Silicon devices are supported.
```
# If you don't have Xcode build tools installed already... You most likely will though?
xcode-select --install
pip install -r requirements.txt
python prepro_tinyshakespeare.py
python train_gpt2.py
make train_gpt2_metal
./train_gpt2_metal
```

## Some Notes on Metal
Metal is a modern, fully-programmable graphics-API first and foremost. Many of the conveniences of CUDA are not present when dealing with Metal. There's quite a bit more boilerplate necessary to set things up and dispatch work to the GPU. Another thing is that we have to go through the Obj-C API to interact with Metal. There are cpp and other language wrappers out there that interact with the objc dynamic runtime to provide bindings. The approach I took was to instead wrap only the bits I needed to set up and dispatch compute kernels. This is all exposed in a pure C header file with no Obj-C code. There are quite a few differences between Metal and CUDA especially on the host side of things. One big difference is in how we call our kernels. We have to manually bind each argument to the argument table with an index that matches up with the attribute index defined in the actual kernel code. There is an API for this in metal_compute.h. I want to avoid using C++ or complex macros. On one hand I'm pretty sure I could cook up a clever templated solution to this in cpp but I'd rather A, have this code stay simple and close to llm.c as possible and B, have things compile super fast--introducing cpp is off the table. When it comes to the shader code, it's very similar to CUDA on some level. The terminology and builtins different though.

## CUDA to Metal Glossary
CUDA -> Metal

Thread -> Thread

Warp ~> SIMD group

Thread Block -> Threadgroup

Grid ~> Similar but Metal (on M-series at least) supports non-uniform threadgroups.

I'll defer to [the documentation](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf) for the definitive spec. One thing that's cool is that there are a lot more builtins for thread indices. Like you can get the absolute thread index in grid directly and non-uniform threadgroup sizes means you don't need to worry about boundary conditions unless you dispatch your kernel via the other API (that I'm not using in metal_compute.h). There are also builtins your kernel can use for totals like threadgroup size, simd size, etc. to determine more about your execution context. 

## More on this Fork
Similar to llm.c, this fork is meant as a place for learning and experimentation. 
**Where things stand:**
- Right now only the forward pass is implemented. 
- The test_gpt2*.c file isn't there. I created llm_cpu.c to compare GPU to CPU results and have a CHECK_TENSOR #define that will run each layer through the GPU, copy the activation to another buffer and then re-run the same layer on CPU and write the result to the activations buffer. Then I'm comparing the two activations within 1e-2 tolerance similar to the test code in the original repo. Doing it in this way ensures that each layer receives input that is coming from a "known-good" source since the last thing to write to the activations buffer was the CPU not our potentially broken GPU kernel.

TODO:
- [x] Set up Metal.
- [x] Get CPU code into a place where it can be used as an "oracle" for testing.
- [x] Write a simple shader that does something.
- [x] Bring over cuda version of train_gpt and comment out all the CUDA bits.
- [x] Figure out allocation strategy for Metal buffers.
- [x] Port encoder layer.
- [x] Port layernorm.
- [x] Port matmul. (ended up using MPS Gemm)
- [x] Port attention.
- [x] Port rest of layers... all downhill from here hopefully!!
- [x] Test everything against CPU to ensure we get the right activations.
- [x] Tidy things up and remove old unused code.
- [x] Grab some of the new stuff like tokenizer decoder.
- [x] Test end-to-end to see if model output looks "right".
- [ ] Ensure all the grad buffers are set up properly and things are ready to implement backward pass.
- [ ] Make sure we have ground-truth grads or backward pass functions to compare against.
- [ ] Implement cross-entropy backward pass.
- [ ] Implement softmax backward pass. Might try fused classifier approach like main repo instead!
- [ ] Matmul backward
- [ ] Layernorm backward
- [ ] Residual backward
- [ ] Gelu backward
- [ ] Attention backward
- [ ] Encoder backward
- [ ] Revisit softmax kernel. Need to use tiled reductions kernel instead of naive one.
- [ ] Try to re-use MPSMatrix and MPSMatrixMultiplication instead of creating a new one each time. I doubt this is a big bottle-neck but should be easy to try.
- [ ] Figure out how to better manage Obj-C memory. Need to somehow drain the autorelease pool and force stuff to free up. Can avoid recreating certain things but not others (like command buffers/encoders)
---
[ORIGINAL README]
---

#  llm.c

LLM training in simple, pure C/CUDA. There is no need for 245MB of PyTorch or 107MB of cPython. Training GPT-2 (CPU, fp32) is ~1,000 lines of clean code in the single file [train_gpt2.c](train_gpt2.c), and training it on GPU is ~2,000 lines (adds CUDA kernels) in [train_gpt2.cu](train_gpt2.cu). The code compiles and runs instantly, it exactly matches the PyTorch reference implementation, and it ~matches the speed of (compiled) PyTorch (fp32, no flash attention). I chose GPT-2 as the first working example because it is the grand-daddy of LLMs, the first time the modern stack was put together.

Currently, we are working on:

- optimize the CUDA implementation further to match/exceed PyTorch speed
- lower the precision from fp32 to mixed precision training
- add multi-gpu training, starting with DDP
- reproduce the GPT-2 training run (add data, evals)
- more modern architectures, Llama 2, Gemma, Mistral, etc.

I'd like this repo to only maintain C and CUDA code. Ports of this repo to other languages are very welcome, but should be done in separate repos, and then I am happy to link to them below in the "notable forks" section, just like I did in [llama2.c notable forks](https://github.com/karpathy/llama2.c/tree/master?tab=readme-ov-file#notable-forks).

## quick start (GPU)

The "I don't care about anything I just want to train and I have a GPU" section. Run:

```bash
pip install -r requirements.txt
python prepro_tinyshakespeare.py
python train_gpt2.py
make train_gpt2cu
./train_gpt2cu
```

The above lines (1) download the [tinyshakespeare](https://raw.githubusercontent.com/karpathy/char-rnn/master/data/tinyshakespeare/input.txt) dataset, tokenize it with the GPT-2 Tokenizer, (2) download and save the GPT-2 (124M) weights, (3) init from them in C/CUDA and train for one epoch on tineshakespeare with AdamW (using batch size 4, context length 1024, total of 74 steps), evaluate validation loss, and sample some text.

## quick start (CPU)

The "I am so GPU poor that I don't even have one" section. No worries, run:

```bash
pip install -r requirements.txt
python prepro_tinyshakespeare.py
python train_gpt2.py
make train_gpt2
OMP_NUM_THREADS=8 ./train_gpt2
```

The above lines (1) download the [tinyshakespeare](https://raw.githubusercontent.com/karpathy/char-rnn/master/data/tinyshakespeare/input.txt) dataset, tokenize it with the GPT-2 Tokenizer, (2) download and save the GPT-2 (124M) weights, (3) init from them in C and train for 40 steps on tineshakespeare with AdamW (using batch size 4, context length only 64), evaluate validation loss, and sample some text. Honestly, unless you have a beefy CPU (and can crank up the number of OMP threads in the launch command), you're not going to get that far on CPU training LLMs, but it might be a good demo/reference.

## training: more detail

Download and tokenize a dataset. The [tinyshakespeare](https://raw.githubusercontent.com/karpathy/char-rnn/master/data/tinyshakespeare/input.txt) dataset is the fastest to download and tokenize:

```bash
python prepro_tinyshakespeare.py
```

This prints:

```
Saved 32768 tokens to data/tiny_shakespeare_val.bin
Saved 305260 tokens to data/tiny_shakespeare_train.bin
```

The .bin files are raw byte streams of int32 numbers indicating the token ids with the GPT-2 tokenizer. Alternatively you could also tokenize the [TinyStories](https://huggingface.co/datasets/roneneldan/TinyStories) dataset with `prepro_tinystories.py`.

In principle we'd be ready to train the model right here. However the baseline CPU/fp32 reference code is so inefficient that it's not practical to train these models from scratch yet. Instead, we initialize with the GPT-2 weights released by OpenAI and just do finetuning. For that, we have to download the GPT-2 weights and save them as a checkpoint we can load in C:

```bash
python train_gpt2.py
```

You'll recognize this code from nanoGPT as a simple GPT-2 reference implementation in PyTorch. This script will download the GPT-2 (124M) model, overfit a single batch of data for 10 iterations, run a few steps of generation, and most importantly it will save three files: 1) the `gpt2_124M.bin` file that contains the raw model weights for loading in C, 2) the `gpt2_124M_debug_state.bin`, which also contains more debug state: the inputs, targets, logits and loss (useful for debugging and unit testing), and finally 3) the `gpt2_tokenizer.bin` which stores the vocabulary for the GPT-2 tokenizer, translating token ids to byte sequences of UTF-8 encoded string pieces. We can now initialize with these model weights and continue training in raw C. First compile the code:

```bash
make train_gpt2
```

You can have a look inside the `Makefile` and its comments. It will try to autodetect if OpenMP is available on your system, which is very helpful for speeding up the code at very low cost of code complexity. Some people seem to experience problems compiling on Ubuntu, have a look at [Issue 19](https://github.com/karpathy/llm.c/issues/19), TLDR you'd want to modify the `CFLAGS`:

```
# try this first
CFLAGS="-Ofast -fno-finite-math-only -Wno-unused-result -march=native" make train_gpt2
# try this second
CFLAGS="-O3 -Wno-unused-result -march=native" make train_gpt2
```

Once `train_gpt2` is compiled, you can run it:

```bash
OMP_NUM_THREADS=8 ./train_gpt2
```

You should tune the number of threads depending on how many cores your CPU has. The program will load the model weights, the tokens, it will run a finetuning loop for a few iterations with Adam lr 1e-4, and then generate a sample from the model. The file is (I think) very readable and you should have a look. Simply, there are implementations for the forward and backward pass of all the layers, and they get strung together into a large, manual, forward/backward/update loop. The output looks like this on my MacBook Pro (Apple Silicon M3 Max):

```
[GPT-2]
max_seq_len: 1024
vocab_size: 50257
num_layers: 12
num_heads: 12
channels: 768
num_parameters: 124439808
train dataset num_batches: 1192
val dataset num_batches: 128
num_activations: 73323776
val loss 5.252026
step 0: train loss 5.356189 (took 1452.121000 ms)
step 1: train loss 4.301069 (took 1288.673000 ms)
step 2: train loss 4.623322 (took 1369.394000 ms)
step 3: train loss 4.600470 (took 1290.761000 ms)
... (trunctated) ...
step 39: train loss 3.970751 (took 1323.779000 ms)
val loss 4.107781
generating:
---
Come Running Away,
Greater conquer
With the Imperial blood
the heaviest host of the gods
into this wondrous world beyond.
I will not back thee, for how sweet after birth
Netflix against repounder,
will not
flourish against the earlocks of
Allay
---
```

I like how Netflix comes up, it's clear that the shadow of the training past is still lurking in the model. I did not attempt to tune the finetuning hyperparameters so it's quite likely this can be improved quite a bit. I also noticed that slightly different platforms (e.g. MacOS / Linux) will (sadly) give very slightly different results, so perhaps don't expect to get the exact numbers or generation above. Also note that if you are seeing token ids instead of text in the generation, it might be because your code is out of date, as Tokenizer decoding was added April 14, 2024. `git pull` the updates, and then re-run `python train_gpt2.py`, which will now also save the tokenizer, which C can read and then use to print text instead of token ids.

## test

I am also attaching a simple unit test for making sure our C code agrees with the PyTorch code. Compile and run with:

```bash
make test_gpt2
./test_gpt2
```

This now loads the `gpt2_124M_debug_state.bin` file, runs a forward pass, compares the logits and loss with the PyTorch reference implementation, then it does 10 iterations of training with Adam and makes sure the losses match PyTorch.

## tutorial

I attached a very small tutorial here, in [doc/layernorm/layernorm.md](doc/layernorm/layernorm.md). It's a simple, step-by-step guide to implementing a single layer of the GPT-2 model, the layernorm layer. This is a good starting point to understand how the layers are implemented in C.

## CUDA

The full training loop is also implemented in pure CUDA in one file, but optimizations of the kernels are ongoing. Currently, we roughly match the speed of PyTorch. The way we organize code is that we have a growing collection of kernels of increasing complexity in the `dev/cuda` folder, see [dev/cuda/README.md](dev/cuda/README.md). We then copy paste the best kernels into the main training loop in the single training file `train_gpt2cu.cu`.

**Correctness**. First, we can do 10 iterations of training and verify that our code exactly matches and preproduces the numbers from PyTorch:

```bash
make test_gpt2cu
./test_gpt2cu
```

This prints `overall okay: 1`. So the forward activations, backward gradients, and the individual loss values for 10 iterations all match exactly.

**Training**. To train GPT-2 in a single file of CUDA, run the train script:

```bash
make train_gpt2cu
./train_gpt2cu
```

This will load the tiny_shakespeare dataset validation and training splits. At the default settings of B=4, T=1024, there are 8 validation batches and 74 training batches. The script is currently configured to do a single epoch of finetuning with learning rate 1e-4, and along the way it evaluates the validation performance and generates samples, e.g.:

```
step 1/74: train loss 4.367631 (80.639749 ms)
step 2/74: train loss 4.031242 (77.378867 ms)
step 3/74: train loss 4.034144 (77.315861 ms)
step 4/74: train loss 3.859865 (77.357575 ms)
...
step 72/74: train loss 3.085081 (78.850895 ms)
step 73/74: train loss 3.668018 (78.197064 ms)
step 74/74: train loss 3.467508 (78.009975 ms)
val loss 3.516490
generating:
---
?Where will you go?
I take you wherefore I can, myself, and must.
I cast off my beak, that I may look him up on the point;
For on his rock shall he be opencast.

<|endoftext|>My little nephew:
Keep on with me, my
```

This runs on my A100 in about ~10 seconds. This training loop in the PyTorch script is about 80ms/iteration, so we are slightly better than PyTorch here. However, this is measured with PyTorch that is a bit stale (I'm on 2.1.0) and we're not yet including FlashAttention or the PyTorch scaled_dot_product_attention fused operation.

We can compare to naive PyTorch like this, where we turn on `torch.compile` and the use of TensorCores, which use tf32 type:

```bash
python train_gpt2.py --write_tensors 0 --sequence_length 1024 --batch_size 4 --compile 1 --tensorcores 1
```

The compilation (first iteration) is ~27 seconds, but after that on my A100 this currently runs at ~80ms/iteration.

## experiments / sweeps

Now that the basic argparse and logging functionality is there in the .cu script, we can do our first learning rate sweeps. This is fairly manual right now, but just to document one example process to sweep learning rates on a machine with 4 GPUs on TinyStories. Run a shell script `sweep.sh` (after you of course `chmod u+x sweep.sh`):

```bash
#!/bin/bash

learning_rates=(3e-5 1e-4 3e-4 1e-3)

for i in {0..3}; do
    export CUDA_VISIBLE_DEVICES=$i
    screen -dmS "tr$i" bash -c "./train_gpt2cu -i data/TinyStories -v 250 -s 250 -g 144 -l ${learning_rates[$i]} -o stories$i.log"
done

# you can bring these down with
# screen -ls | grep -E "tr[0-3]" | cut -d. -f1 | xargs -I {} screen -X -S {} quit
```

This example opens up 4 screen sessions and runs the four commands with different LRs. This writes the log files `stories$i.log` with all the losses, which you can plot as you wish in Python. Here's a quick example script to plot the losses in a Jupyter notebook, obviously can become more sophisticated later:

```python
import matplotlib.pyplot as plt
%matplotlib inline

def parse_log(logfile):
  # look for lines like e.g. "s:100 tel:1.6952", step 100, val 1.6952
    val_steps, val_losses = [], []
    with open(logfile, "r") as f:
        lines = f.readlines()
    for line in lines:
        if "tel" in line:
            parts = line.split()
            step = parts[0].split(":")[1]
            loss = parts[1].split(":")[1]
            val_steps.append(int(step))
            val_losses.append(float(loss))
    return val_steps, val_losses

results = [parse_log(f"stories{i}.log") for i in range(0, 4)]
for i, (val_steps, val_losses) in enumerate(results):
    plt.plot(val_steps, val_losses, label="run {}".format(i))
plt.xlabel("steps")
plt.ylabel("loss")
plt.legend()
```

## repo philosophy

A few more words on what I want this repo to be:

First, I want `llm.c` to be a place for education. E.g. our `dev/cuda` folder is a place for a library of kernels for all the layers that are manually hand-written and very well documented, starting from very simple kernels all the way to more complex / faster kernels. If you have a new kernel with various different tradeoffs, please feel free to contribute it here.

That said, I also want `llm.c` to be very fast too, even practically useful to train networks. E.g. to start, we should be able to reproduce the big GPT-2 (1.6B) training run. This requires that we incorporate whatever fastest kernels there are, including the use of libraries such as cuBLAS, cuBLASLt, CUTLASS, cuDNN, etc. I also think doing so serves an educational purpose to establish an expert upper bound, and a unit of measurement, e.g. you could say that your manually written kernels are 80% of cuBLAS speed, etc. Then you can choose to do a super fast run, or you can choose to "drag and drop" whatever manual kernels you wish to use, and run with those.

However, as a constraint, I want to keep the mainline `llm.c` in the root folder simple and readable. If there is a PR that e.g. improves performance by 2% but it "costs" 500 lines of complex C code, and maybe an exotic 3rd party dependency, I may reject the PR because the complexity is not worth it. In that sense I'd be ok to only be at e.g. 90% of PyTorch speed, if it means we can remain at ~2,000 readable lines of code with minimal exotic dependencies. As a concrete example - making cuBLAS for matmuls the default in the root training loop is a no-brainer: it makes the mainline code much faster, it is a single line of interpretable code, and it is a very common dependency. On the side of this, we can have manual implementations that can compete with cuBLAS in `dev/cuda`.

Lastly, I will be a lot more sensitive to complexity in the root folder of the project, which contains the main / default files of the project. In comparison, the `dev/` folder is a bit more of a scratch space for us to develop a library of kernels or classes and share useful or related or educational code, and some of this code could be ok to be (locally) complex.

## notable forks

- Mojo
  - [llm.🔥](https://github.com/dorjeduck/llm.mojo) by @[dorjeduck](https://github.com/dorjeduck): a Mojo port of this project

- C#
  - [llm.cs](https://github.com/azret/llm.cs) by @[azret](https://github.com/azret): a C# port of this project

## discussions

Ways of organizing development:

- Experiencing a concrete issue with the repo? Use [Issues](https://github.com/karpathy/llm.c/issues).
- Have some code to contribute? Open a [PR](https://github.com/karpathy/llm.c/pulls)
- Chat about the repo, ask questions, etc.? Look at [Discussions](https://github.com/karpathy/llm.c/discussions).
- Something faster? I created a new `#llmc` channel on my [Zero to Hero Discord channel](https://discord.gg/3zy8kqD9Cp).

## license

MIT
