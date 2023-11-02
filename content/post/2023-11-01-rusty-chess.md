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

## Future Work

The current iteration of rusty chess relies on a fully connected deep neural network to score board states. This is a simple approach but it is not necessarily the best architecture for capturing the positional relationships between pieces. In the future I would like to experiment with a convolutional neural network that may better learn the scoring function. 

I'm additionally interested to experiment with optimizations to better prioritize the search space e.g. by using a transposition table to avoid re-searching states that have already been visited and implementing [quiescence search](https://www.chessprogramming.org/Quiescence_Search) to avoid the horizon effect.

## An Embedded Demo
<div style="background-color: white">
    <iframe src="https://garethgeorge.github.io/rustychess/" width="100%" height="1000px" style="border: none;"></iframe>
</div>