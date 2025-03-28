# notes
Building a character-level large language model (chatgpt is a token-level llm, where tokens are word chunks): define a transformer, train it on tiny shakespeare dataset, generate infinite shakespeare. 

First, define tokenizer to convert the text as a string to a sequence of integers according to a vocabulary of elements. In the case of a character-level model, translate individual characters into integers. 

Second, define data loader to feed the integer sequences into the model: sample batch (efficiency reasons) of random chunks of sequences (with a length block_size, computational reasons) from the dataset and feed them to the network. In a chunk of block_size, the chunk contains block_size examples and block_size+1 characters. Let's explain with example: when block_size is 2, the chunk may look like (1,2,3). 
The following examples are in this chunk: 
- when input is (1), the target is 2
- when input is (1,2), the target is 3
So, train on examples with context 1 to context of block size. This enables the network to start sampling generation with even 1 character of context.  

Let's start with a character-level bigram language model: pass the inputs to a token-embedding table. A token-embedding is a vector of shape (vocab_size, vocab_size). If an input is passed to this table, it returns a vector that represents the raw logits for the next character in the sequence. These logits can be interpreted as unnormalized probabilities over the vocabulary, indicating the likelihood of each possible next character given the current one. These logits can be used to: 1. during training, evaluate the quality of the predictions (with respect to the targets) using cross-entropy loss (negative log likelihood); 2. during prediction, generate the next character in the sequence.

The results of the bigram language model are of course not that great. It's a very simple model where the tokens are not 'talking to each other': given the previous context, only the last character is considered to make a prediction about the next character. What's the simplest way to make tokens communicate with each other? Bag of words: take the average of the current and the previous tokens. How to do this efficiently? The mathematical trick in self-attention: at each position, compute weighted aggregations of past elements using matrix multiplication with a lower-triangular structure (tril). The values in this matrix determine the contribution of each past element to the current position. This forms the foundation of the self-attention block.

Let's implement self-attention for a single head, refining how we define the weighted aggregation matrix: 1. Initialize wei as a zero matrix of shape (block_size, block_size); 2. Mask future positions by setting all elements where tril (a lower-triangular mask) is zero to negative infinity; 3. Apply softmax along each row to normalize the weights, ensuring they sum to 1. At this stage, wei represents the attention scores, or affinities, between different tokens. Since we initially set wei to zero, the resulting softmax distribution is uniform, meaning each token equally weighs all allowed past tokens, effectively computing a simple average. However, different tokens should attend to others with varying importance. To allow tokens to determine relevance in a data-dependent way, each token generates three vectors: Query vector (Q): Represents what the token is looking for; Key vector (K): Represents what the token contains; Value vector (V): Contains the actual information to be aggregated. Self-attention computes affinities by performing a dot product between queries and keys, where each query interacts with all previous keys. The resulting scores update wei, replacing the uniform weights with meaningful relationships between tokens. To stabilize the computation, these scores are scaled by 1 / sqrt(d_k), where d_k is the dimension of the key vectors. In practice, queries, keys, and values are projected into a lower-dimensional space head_size. Finally, the updated wei is used to weight the value vectors (V). This weighted sum produces the final output representation, where each token now contains information from the past in a structured, context-aware way. This process is efficiently implemented using three linear layers (without bias), mapping tokens into query, key, and value spaces before performing matrix multiplications.

Some notes on self-attention: 
- Attention is a communication mechanism, can be seen as nodes in a graph looking at each other and aggregating information with a weighted sum from all nodes that point to them with data-dependent weights. In this case, the first node points to itself, the second node point to itself and the first node, the third node points to itself, the first and the second node and so on...
- There's no notion of space (position). Attention simply acts over a set of vectors. This is why we need to positionally encode tokens.
- Each example across the batch dimension if processed completely independent and never 'talk' to each other.
- In an "encoder" attention block the line that does masking (tril) is deleted, allowing all tokens to communicate. Now, it's called a "decoder" attention block because it has triangular masking, and it's usually used in autoregressive setting, like modeling.
- Self-attention means that the keys and values are produced from the same source as queries. In cross-attention, the queries get produced from x, but the keys and values come from another source. 
- Scaled attention additionally divides wei by sqrt(head_size). When input Q and K are unit variance, then wei will be unit variance too and softmax will stay diffuse and not saturate too much.

In multi-head attention, multiple attention mechanisms run in parallel, each learning to focus on different aspects of the input sequence. The results from all heads are then concatenated, allowing the model to capture a richer set of dependencies. The head size is typically defined as d_k = embedding_dim / num_heads, ensuring that the total computation remains manageable.

After self-attention, a fully connected layer is applied to further refine the representations learned by the attention mechanism. This follows a two-step process: first, tokens communicate with each other through attention, integrating contextual information. Then, each token independently processes this aggregated data through the feedforward layer, enabling deeper feature extraction.

The results of the model with multi-head attention are still not that great. The neural networks is starting to get pretty deep and deep neural network suffer from optimization issues. There are two optimizations that drastically help with the depth issue: 1. Introducing residual connections: the data is first transformed, then the original features are added back to the transformed output. With this addition, the network can compute targets using only simple element-wise addition. As seen in micrograd, addition distributes gradients equally during backpropagation, creating a gradient highway—allowing gradients from the loss to flow smoothly through every addition node back to the input. At initialization, residual blocks contribute very little, behaving almost like an identity function. Early in training, their impact is minimal, but as optimization progresses, they gradually activate, refining features without disrupting the existing representations. This smooth transition significantly improves optimization, making deep networks easier to train; 2. Introducing layer normalization: layer normalization is very similar to batch normalization. Remember from makemore that batch normalization ensures each neuron has a unit Gaussian distribution across the batch dimension. Layer normalization normalizes across its feature dimensions. 

In the next step, the model is scaled up to push the performance. To prevent overfitting, dropout can be added. Dropout is a regularization technique that randomly de-activates a subset of neurons during every forward and backward pass.  

Add a position-embedding table to not only encode the identity of the token, but also the position. 



Small side-note about cuda: when cuda is used, move the data and the model parameters to the gpu. Then, the calculations happen on the gpu and they can be run a lot faster. 
# nanoGPT

![nanoGPT](assets/nanogpt.jpg)

The simplest, fastest repository for training/finetuning medium-sized GPTs. It is a rewrite of [minGPT](https://github.com/karpathy/minGPT) that prioritizes teeth over education. Still under active development, but currently the file `train.py` reproduces GPT-2 (124M) on OpenWebText, running on a single 8XA100 40GB node in about 4 days of training. The code itself is plain and readable: `train.py` is a ~300-line boilerplate training loop and `model.py` a ~300-line GPT model definition, which can optionally load the GPT-2 weights from OpenAI. That's it.

![repro124m](assets/gpt2_124M_loss.png)

Because the code is so simple, it is very easy to hack to your needs, train new models from scratch, or finetune pretrained checkpoints (e.g. biggest one currently available as a starting point would be the GPT-2 1.3B model from OpenAI).

## install

```
pip install torch numpy transformers datasets tiktoken wandb tqdm
```

Dependencies:

- [pytorch](https://pytorch.org) <3
- [numpy](https://numpy.org/install/) <3
-  `transformers` for huggingface transformers <3 (to load GPT-2 checkpoints)
-  `datasets` for huggingface datasets <3 (if you want to download + preprocess OpenWebText)
-  `tiktoken` for OpenAI's fast BPE code <3
-  `wandb` for optional logging <3
-  `tqdm` for progress bars <3

## quick start

If you are not a deep learning professional and you just want to feel the magic and get your feet wet, the fastest way to get started is to train a character-level GPT on the works of Shakespeare. First, we download it as a single (1MB) file and turn it from raw text into one large stream of integers:

```sh
python data/shakespeare_char/prepare.py
```

This creates a `train.bin` and `val.bin` in that data directory. Now it is time to train your GPT. The size of it very much depends on the computational resources of your system:

**I have a GPU**. Great, we can quickly train a baby GPT with the settings provided in the [config/train_shakespeare_char.py](config/train_shakespeare_char.py) config file:

```sh
python train.py config/train_shakespeare_char.py
```

If you peek inside it, you'll see that we're training a GPT with a context size of up to 256 characters, 384 feature channels, and it is a 6-layer Transformer with 6 heads in each layer. On one A100 GPU this training run takes about 3 minutes and the best validation loss is 1.4697. Based on the configuration, the model checkpoints are being written into the `--out_dir` directory `out-shakespeare-char`. So once the training finishes we can sample from the best model by pointing the sampling script at this directory:

```sh
python sample.py --out_dir=out-shakespeare-char
```

This generates a few samples, for example:

```
ANGELO:
And cowards it be strawn to my bed,
And thrust the gates of my threats,
Because he that ale away, and hang'd
An one with him.

DUKE VINCENTIO:
I thank your eyes against it.

DUKE VINCENTIO:
Then will answer him to save the malm:
And what have you tyrannous shall do this?

DUKE VINCENTIO:
If you have done evils of all disposition
To end his power, the day of thrust for a common men
That I leave, to fight with over-liking
Hasting in a roseman.
```

lol  `¯\_(ツ)_/¯`. Not bad for a character-level model after 3 minutes of training on a GPU. Better results are quite likely obtainable by instead finetuning a pretrained GPT-2 model on this dataset (see finetuning section later).

**I only have a macbook** (or other cheap computer). No worries, we can still train a GPT but we want to dial things down a notch. I recommend getting the bleeding edge PyTorch nightly ([select it here](https://pytorch.org/get-started/locally/) when installing) as it is currently quite likely to make your code more efficient. But even without it, a simple train run could look as follows:

```sh
python train.py config/train_shakespeare_char.py --device=cpu --compile=False --eval_iters=20 --log_interval=1 --block_size=64 --batch_size=12 --n_layer=4 --n_head=4 --n_embd=128 --max_iters=2000 --lr_decay_iters=2000 --dropout=0.0
```

Here, since we are running on CPU instead of GPU we must set both `--device=cpu` and also turn off PyTorch 2.0 compile with `--compile=False`. Then when we evaluate we get a bit more noisy but faster estimate (`--eval_iters=20`, down from 200), our context size is only 64 characters instead of 256, and the batch size only 12 examples per iteration, not 64. We'll also use a much smaller Transformer (4 layers, 4 heads, 128 embedding size), and decrease the number of iterations to 2000 (and correspondingly usually decay the learning rate to around max_iters with `--lr_decay_iters`). Because our network is so small we also ease down on regularization (`--dropout=0.0`). This still runs in about ~3 minutes, but gets us a loss of only 1.88 and therefore also worse samples, but it's still good fun:

```sh
python sample.py --out_dir=out-shakespeare-char --device=cpu
```
Generates samples like this:

```
GLEORKEN VINGHARD III:
Whell's the couse, the came light gacks,
And the for mought you in Aut fries the not high shee
bot thou the sought bechive in that to doth groan you,
No relving thee post mose the wear
```

Not bad for ~3 minutes on a CPU, for a hint of the right character gestalt. If you're willing to wait longer, feel free to tune the hyperparameters, increase the size of the network, the context length (`--block_size`), the length of training, etc.

Finally, on Apple Silicon Macbooks and with a recent PyTorch version make sure to add `--device=mps` (short for "Metal Performance Shaders"); PyTorch then uses the on-chip GPU that can *significantly* accelerate training (2-3X) and allow you to use larger networks. See [Issue 28](https://github.com/karpathy/nanoGPT/issues/28) for more.

## reproducing GPT-2

A more serious deep learning professional may be more interested in reproducing GPT-2 results. So here we go - we first tokenize the dataset, in this case the [OpenWebText](https://openwebtext2.readthedocs.io/en/latest/), an open reproduction of OpenAI's (private) WebText:

```sh
python data/openwebtext/prepare.py
```

This downloads and tokenizes the [OpenWebText](https://huggingface.co/datasets/openwebtext) dataset. It will create a `train.bin` and `val.bin` which holds the GPT2 BPE token ids in one sequence, stored as raw uint16 bytes. Then we're ready to kick off training. To reproduce GPT-2 (124M) you'll want at least an 8X A100 40GB node and run:

```sh
torchrun --standalone --nproc_per_node=8 train.py config/train_gpt2.py
```

This will run for about 4 days using PyTorch Distributed Data Parallel (DDP) and go down to loss of ~2.85. Now, a GPT-2 model just evaluated on OWT gets a val loss of about 3.11, but if you finetune it it will come down to ~2.85 territory (due to an apparent domain gap), making the two models ~match.

If you're in a cluster environment and you are blessed with multiple GPU nodes you can make GPU go brrrr e.g. across 2 nodes like:

```sh
# Run on the first (master) node with example IP 123.456.123.456:
torchrun --nproc_per_node=8 --nnodes=2 --node_rank=0 --master_addr=123.456.123.456 --master_port=1234 train.py
# Run on the worker node:
torchrun --nproc_per_node=8 --nnodes=2 --node_rank=1 --master_addr=123.456.123.456 --master_port=1234 train.py
```

It is a good idea to benchmark your interconnect (e.g. iperf3). In particular, if you don't have Infiniband then also prepend `NCCL_IB_DISABLE=1` to the above launches. Your multinode training will work, but most likely _crawl_. By default checkpoints are periodically written to the `--out_dir`. We can sample from the model by simply `python sample.py`.

Finally, to train on a single GPU simply run the `python train.py` script. Have a look at all of its args, the script tries to be very readable, hackable and transparent. You'll most likely want to tune a number of those variables depending on your needs.

## baselines

OpenAI GPT-2 checkpoints allow us to get some baselines in place for openwebtext. We can get the numbers as follows:

```sh
$ python train.py config/eval_gpt2.py
$ python train.py config/eval_gpt2_medium.py
$ python train.py config/eval_gpt2_large.py
$ python train.py config/eval_gpt2_xl.py
```

and observe the following losses on train and val:

| model | params | train loss | val loss |
| ------| ------ | ---------- | -------- |
| gpt2 | 124M         | 3.11  | 3.12     |
| gpt2-medium | 350M  | 2.85  | 2.84     |
| gpt2-large | 774M   | 2.66  | 2.67     |
| gpt2-xl | 1558M     | 2.56  | 2.54     |

However, we have to note that GPT-2 was trained on (closed, never released) WebText, while OpenWebText is just a best-effort open reproduction of this dataset. This means there is a dataset domain gap. Indeed, taking the GPT-2 (124M) checkpoint and finetuning on OWT directly for a while reaches loss down to ~2.85. This then becomes the more appropriate baseline w.r.t. reproduction.

## finetuning

Finetuning is no different than training, we just make sure to initialize from a pretrained model and train with a smaller learning rate. For an example of how to finetune a GPT on new text go to `data/shakespeare` and run `prepare.py` to download the tiny shakespeare dataset and render it into a `train.bin` and `val.bin`, using the OpenAI BPE tokenizer from GPT-2. Unlike OpenWebText this will run in seconds. Finetuning can take very little time, e.g. on a single GPU just a few minutes. Run an example finetuning like:

```sh
python train.py config/finetune_shakespeare.py
```

This will load the config parameter overrides in `config/finetune_shakespeare.py` (I didn't tune them much though). Basically, we initialize from a GPT2 checkpoint with `init_from` and train as normal, except shorter and with a small learning rate. If you're running out of memory try decreasing the model size (they are `{'gpt2', 'gpt2-medium', 'gpt2-large', 'gpt2-xl'}`) or possibly decreasing the `block_size` (context length). The best checkpoint (lowest validation loss) will be in the `out_dir` directory, e.g. in `out-shakespeare` by default, per the config file. You can then run the code in `sample.py --out_dir=out-shakespeare`:

```
THEODORE:
Thou shalt sell me to the highest bidder: if I die,
I sell thee to the first; if I go mad,
I sell thee to the second; if I
lie, I sell thee to the third; if I slay,
I sell thee to the fourth: so buy or sell,
I tell thee again, thou shalt not sell my
possession.

JULIET:
And if thou steal, thou shalt not sell thyself.

THEODORE:
I do not steal; I sell the stolen goods.

THEODORE:
Thou know'st not what thou sell'st; thou, a woman,
Thou art ever a victim, a thing of no worth:
Thou hast no right, no right, but to be sold.
```

Whoa there, GPT, entering some dark place over there. I didn't really tune the hyperparameters in the config too much, feel free to try!

## sampling / inference

Use the script `sample.py` to sample either from pre-trained GPT-2 models released by OpenAI, or from a model you trained yourself. For example, here is a way to sample from the largest available `gpt2-xl` model:

```sh
python sample.py \
    --init_from=gpt2-xl \
    --start="What is the answer to life, the universe, and everything?" \
    --num_samples=5 --max_new_tokens=100
```

If you'd like to sample from a model you trained, use the `--out_dir` to point the code appropriately. You can also prompt the model with some text from a file, e.g. ```python sample.py --start=FILE:prompt.txt```.

## efficiency notes

For simple model benchmarking and profiling, `bench.py` might be useful. It's identical to what happens in the meat of the training loop of `train.py`, but omits much of the other complexities.

Note that the code by default uses [PyTorch 2.0](https://pytorch.org/get-started/pytorch-2.0/). At the time of writing (Dec 29, 2022) this makes `torch.compile()` available in the nightly release. The improvement from the one line of code is noticeable, e.g. cutting down iteration time from ~250ms / iter to 135ms / iter. Nice work PyTorch team!

## todos

- Investigate and add FSDP instead of DDP
- Eval zero-shot perplexities on standard evals (e.g. LAMBADA? HELM? etc.)
- Finetune the finetuning script, I think the hyperparams are not great
- Schedule for linear batch size increase during training
- Incorporate other embeddings (rotary, alibi)
- Separate out the optim buffers from model params in checkpoints I think
- Additional logging around network health (e.g. gradient clip events, magnitudes)
- Few more investigations around better init etc.

## troubleshooting

Note that by default this repo uses PyTorch 2.0 (i.e. `torch.compile`). This is fairly new and experimental, and not yet available on all platforms (e.g. Windows). If you're running into related error messages try to disable this by adding `--compile=False` flag. This will slow down the code but at least it will run.

For some context on this repository, GPT, and language modeling it might be helpful to watch my [Zero To Hero series](https://karpathy.ai/zero-to-hero.html). Specifically, the [GPT video](https://www.youtube.com/watch?v=kCc8FmEb1nY) is popular if you have some prior language modeling context.

For more questions/discussions feel free to stop by **#nanoGPT** on Discord:

[![](https://dcbadge.vercel.app/api/server/3zy8kqD9Cp?compact=true&style=flat)](https://discord.gg/3zy8kqD9Cp)

## acknowledgements

All nanoGPT experiments are powered by GPUs on [Lambda labs](https://lambdalabs.com), my favorite Cloud GPU provider. Thank you Lambda labs for sponsoring nanoGPT!
