---
author: Francis Phan
pubDatetime: 2024-01-22T05:24:46.539Z
title: Vector Search with Vertex AI (4-part series, intro)
featured: false
draft: false
tags:
  - ai
  - machine-learning
  - llm
  - generative-ai
  - vector-search
description: A 4-part series on how to use Vertex AI to build a vector search engine.
---

The advent of LLMs has seen it used in a wide range of applications, most notably ChatGPT or GitHub
Copilot. LLMs are also good for another thing: generating high-fidelity vectors, or embeddings.
These vectors can then be used to build a search engine using a distance-based search algorithm,
ranging from the cute simple cosine similarity to the relatively more complex KNN.

If you're not sure what I'm talking about, don't worry. I'll start by explaining the basics, and
then we'll get into the fun stuff. More experienced readers can skip ahead to the next section.

## What is vector search?

Imagine you look at a 2D map of the world. By using the latitude and longitude, you can tell
where a place is relative to another place. For example, Auckland is closer to Sydney than it is
to London.

Examples of 2D vectors include flat maps and those you did in high school to first year university
calculus. And then, up one level and you have 3D. You can now describe the position of an object in space
using three numbers. By adding more dimensions, you can now describe how far or close an object is
to another object in a higher dimensional space. This is the basis of vector search.

The interesting thing is that as you add more dimensions, you can describe distances between
objects more accurately. This is an overkill for a point in 2D or 3D space, but when you have
very unstructured data, like text, it becomes very useful.

In fact, my last name, Phan, results in 730 dimensions when I use Google Gemini to generate a vector
for it.

Indeed, vectors don't have to be generated by LLM. For structured and semi-structured data, you can
use enumeration and serialisation to manually generate vectors. For example, in the below example,
you can set an enum for the make of a car (like 1 for Toyota) and the engine type (like 3 for Electric).

```json
{
  "make": "Toyota",
  "year": 2018,
  "engine_type": "ELECTRIC"
}
```

This can be converted to a vector like so:

```json
[1, 2018, 3]
```

You can also train traditional machine learning models to generate vectors, instead of
having to write explicit code to do so. Still, this is not very useful for unstructured data like
text. This is where LLMs come in.

After this process, you can then use a distance-based search algorithm to compare and contrast
vectors. The most common one is cosine similarity, which is a measure of the angle between two
vectors. The closer the angle is to 0, the more similar the vectors are.

## Our plan

Please note that this is a four-part series given the complexity. I am also writing in my spare time,
so please expect a new post every month or so.

- Part 1: We will start first by examining the dataset we will use to classify what is structured
  and what is unstructured data. For the unstructured data, we will use Google Gemini to generate
  embeddings for it.

- Part 2: We will train a simple KNN model to classify the structured data, then put both structured
  data and unstructured data together into a single dataset again.

- Part 3: We will then use Vertex AI to train a KNN model to search the combined dataset.

- Part 4: We'll build a simple web app to search the dataset. This will be done using React, FastAPI,
  and Cloud SQL.

## Assumptions I'm making

I'm assuming that you have the following:

- A Google Cloud account with billing enabled. You can get $300 free credit when you sign up.
- A Google Cloud project. You can create one [here](https://console.cloud.google.com/projectcreate).
- A Mac, Linux or WSL2-enabled Windows machine, with Python 3.9+ and Google Cloud CLI installed.
- Basic knowledge of Python, cloud computing, relational and non-relational databases.

That's it! I'll see you in the next post, where we'll start by examining the dataset we will use.
