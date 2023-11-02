+++
title = "Rusty Chess"
date = "2023-11-01"
description = "Building a Chess AI using deep neural networks, rust, and WASM."
tags = [
    "rust",
    "ai",
    "wasm",
]
+++

Rusty Chess is a browser native Chess AI built in Rust, compiled to WASM, and leveraging a Neural Network scoring function.

## Some Quick Links

 * Rusty Chess Demo [demo](https://garethgeorge.github.io/rustychess/)
 * On GitHub [repo](https://github.com/garethgeorge/rustychess/)

## Why Chess?

For a while now I've been looking for a good project and excuse to play with what I view as some of the most interesting emerging technologies and, in revisiting an early interest of mine, I found an excuse to play with three of these:

 * Rust
 * WASM
 * Neural Networks

My journey building Chess AI's started early in college as an excuse to learn about optimizing C++ and representations of data in memory (e.g. bitboards, etc). In this latest revision, I've pivoted from my earlier attempts that focused on search speed to achieve intelligence by searching vast numbers of board states to instead look at more intelligent scoring of each state using deep neural networks. The total number of states visited is vastly reduced (search depth is as low as 3 or 4 plys) and yet a remarkably intelligent gameplay is achievable with a neural network scoring function trained on huge datasets of human gameplay. As a secondary goal, I wanted my AI to be able to run directly in a browser with no server side component that would require maintenance. 

While stockfish and other AI engines exist this was a fantastic learning project and enabled me to combine a number of very interesting technologies in a toy application. And while my AI definitely does not beat state of the art engines e.g. stockfish, it certainly crushes me in common game play!

## An Embedded Demo
<div style="background-color: white">
    <iframe src="https://garethgeorge.github.io/rustychess/" width="100%" height="1000px" style="border: none;"></iframe>
</div>

## Training a Chess model

A brief discussion of the construction and trianing of a neural network scoring function for chess (from the perspective of a non-expert!).

### The Lichess Dataset

Training rustychess starts with the [lichess datasets](https://database.lichess.org/). Lichess provides fantastic (and huge!) datasets of human gameplay freely available for download. Games are provided as zstd compressed streams of gameplay encoded in [pgn notation](https://en.wikipedia.org/wiki/Portable_Game_Notation). Additionally, about 6 percent of games are annotated with stockfish ratings for each move. This annotated subset of games is what I chose to train my a neural network on. E.g. the network will try to become a heuristic for the scores provided by stockfish. With this approach we clearly won't be able to beat stochfish but we can hopefully create a "super good enough" heuristic that can beat me (and most human players!). 

Lichess publishes an absolutely huge amount of data, I chose to generate a dataset of about 100 million positions and their evaluations as a good jumping off point ([code](https://github.com/garethgeorge/rustychess/blob/master/modeltraining/lib/game.py#L7-L32)) and I stored these states as serialized python objects in a leveldb database for scanning and random access at training time (choice to use a database is simply because it's too large a dataset to fit in my workstation's memory).

### Representing Boards

Once a dataset was obtained, the first model decision needed is how to represent board states to a neural network. I chose to represent boards as a 8x8x14 tensor of booleans. That is:

 * 8x8 board positions (64 squares)
 * 14 channels (one for each piece type and color, plus one for the current player)

For each square at most one bit of the 14 channels will be set if a piece is in the square. Using this boolean encoding makes for an input that is very easy for the neural network to decode (e.g. as opposed to more complicated schemes such as a value 0-7 for each piece type). I additionally extend the input vector with 6 additional channels that encode the player's turn and castling rights.

### Training the Model

I chose to build a model in pytorch, the general architecture I chose was a fully connected deep neural network with three hidden layers of differing sizes. I'm quite sure that this structure isn't optimal, but with this approach I was able to achieve very respectible training losses and good performance in gameplay. At the moment I'm using hidden layers of sizes:
 
  * Input: 902
  * Hidden 1: 451 (exactly half the input)
  * Hidden 2: 32
  * Hidden 3: 32
  * Output: linear layer with a single output

Each layer is separated by a relu activation function and the final layer is a linear layer with a single output. The model is trained using a mean squared error loss function and the Adam optimizer. I trained on an RTX 3080 and typically for a small number of epochs (e.g. 2-4) due to the huge dataset and relatively small size of the model. 

Code constructing the model is below (see [github here](https://github.com/garethgeorge/rustychess/blob/d9faa5df38c718c7b89b311bc372acf2743d43d2/modeltraining/training.py#L64C1-L88C1)):

```python
class EvaluationModel(pl.LightningModule):
  def __init__(self,learning_rate=1e-3,batch_size=1024,layer_shapes=[lib.game.tensor_len, lib.game.tensor_len, lib.game.tensor_len, lib.game.tensor_len]):
    super().__init__()
    self.batch_size = batch_size
    self.learning_rate = learning_rate
    layers = []
    prev_shape = lib.game.tensor_len
    for i in range(len(layer_shapes)):
      layers.append((f"linear-{i}", nn.Linear(prev_shape, layer_shapes[i])))
      layers.append((f"relu-{i}", nn.ReLU()))
      prev_shape = layer_shapes[i]
    layers.append((f"linear-{len(layer_shapes)}", nn.Linear(prev_shape, 1)))
    self.seq = nn.Sequential(OrderedDict(layers))
    print(self.seq)

  def forward(self, x):
    return self.seq(x)

  def training_step(self, batch, batch_idx):
    x, y = batch['binary'], batch['eval']
    y_hat = self(x)
    loss = F.l1_loss(y_hat, y)
    self.log("train_loss", loss)
    return loss
```


## Rusty ML

Given that evaluating the model is by far the most expensive part of move search, I suspect a perfectly decent chess AI could have been built natively in python leveraging the pytorch libraries (and in fact there are plenty of examples online of this approach). However, I wanted to experiment with compiling the model to WASM. To support this, I investigated my options for embedding the model in a compiled WASM binary and ultimately settled on Rust as a great language choice for warking with WASM and on [candle](https://github.com/huggingface/candle) as a library in the Rust ecosystem which allows me to easily implement a neural network.

Once I had a decent handle on candle's interfaces I was able to write some rust code that could load my model's weights, exported from pytorch in the [safetensor](https://huggingface.co/docs/safetensors/index) format, in rust and evaluate it on the CPU. This Rust code generally looks familiar to the pytorch code above (though with obvious Rust-y differences):

```rust
struct Model {
    layers: Vec<Linear>,
}

impl Model {
    fn forward(&self, input: &Tensor) -> candle_core::Result<Tensor> {
        let mut x = input.clone();
        for layer in &self.layers[0..self.layers.len() - 1] {
            x = layer.forward(&x)?;
            x = x.relu()?;
        }
        x = self.layers[self.layers.len() - 1].forward(&x)?;
        return Ok(x);
    }
}

fn load(safetensors: &[u8], prefix: &str) {
    let device = Device::Cpu;
    let tensors = candle_core::safetensors::load_buffer(safetensors, &device)
        .context("Failed to load safetensors")?;

    let mut keys = tensors.keys().collect::<Vec<_>>();
    keys.sort();

    let mut model = Model { layers: Vec::new() };

    for key in keys {
        if !key.starts_with(prefix) || !key.ends_with(".weight") {
            continue;
        }

        let Some(weight) = tensors.get(key) else {
            return Err(anyhow::anyhow!("Failed to find weight for key {}", key));
        };
        let Some(bias) = tensors.get(&key.replace(".weight", ".bias")) else {
            return Err(anyhow::anyhow!("Failed to find bias for key {}", key));
        };
        model
            .layers
            .push(Linear::new(weight.clone(), Some(bias.clone())));
    }
...
}
```

this code makes some simplifying assumptions (e.g. that every layer is the same type) which work nicely with our very simple model architecture (e.g. all layers are fully connected and separated by ReLU). For more complicated models, the choice to reimplement the model in Rust grows a bit in complexity as both the model implementation and serialization/deserialization code must be matched carefully to the pytorch model.

With this work done, I was able to embed the model weights in the rust binary using the `include_bytes!` macro. The result is a very satisfying binary that is fully self contained and weighs in at ~7MB when compiled to the WASM target.

## Future Work

The current iteration of rusty chess relies on a fully connected deep neural network to score board states. This is a simple approach but it is not necessarily the best architecture for capturing the positional relationships between pieces. In the future I would like to experiment with a convolutional neural network that may better learn the scoring function. 

I'm additionally interested to experiment with optimizations to better prioritize the search space e.g. by using a transposition table to avoid re-searching states that have already been visited and implementing [quiescence search](https://www.chessprogramming.org/Quiescence_Search) to avoid the horizon effect.
