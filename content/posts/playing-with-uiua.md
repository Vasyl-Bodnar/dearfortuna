+++
author = "V"
title = "Playing with Uiua for Optimal Mindustry Mining"
date = "2026-07-08T18:00:00-04:00" 
description = "Using Uiua for some minor problems I have in Mindustry."
tags = ["Uiua", "Dynamic Programming", "Mindustry"]
style = "custom.css"
draft = true
+++

## Intro
I happened to be playing [Mindustry](https://mindustrygame.github.io) one evening.
But I kept getting annoyed by a minor issue, my drills and mining are probably suboptimal.

![Picture of resources in Mindustry with a small drill over a few tiles of them](/images/mindustry-drills.jpg)

If you can see the picture and can understand it, the drills cover two by two tiles, 
whereas the resources are a large uneven cluster.
It is unfortunately an innate and natural desire to make those drills cover all possible resources you can get.
Thankfully, I have an outlet for such unrefined desires in my life.
I could just make an optimal algorithm for it.

## The Question to Ask
First we need to state the problem properly so that the algorithm can be well-defined.
This is generally an easy problem to state, especially if we ignore some extra details 
(cluster overlap, the big problem). We have a matrix of ones and zeros as an input, e.g.
```c
[0 1 1 0 
 1 1 1 1 
 0 1 1 0]
```


## Working

## The First Hurdle
```uiua
+1 1_1_1
```
You can see, run, and play with the code [here in the Uiua online interpreter]().

One problem though
