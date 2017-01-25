---
title:  "SharePoint meets TensorFlow!"
date:   2017-01-26 10:00:00
categories: [TensorFlow,SharePoint,.net core]
tags: [TensorFlow,SharePoint,.net core]
---

## Before they met
I guess many of us have used [How Old website](https://how-old.net/). It was a demo site from Microsoft to showcase Vision APIs as part of cognitive service. Vision API now can also describe the image and provide even more information about the image content such as genders of people in picture if any, classification of racial/adult content and so on. This is all coming from the booming of neural networks deep learning with breakthroughs in the last few years. Deep learning is more targeted for computer vision problems but can be applied to many other domains as well.

There are many tools to do deep learning ranging from cloud APIs like Microsoft cognitive service and Azure ML to open source frameworks like TensorFlow, Cafee, and Theano. In this post I will consume a ready-made TensorFlow model from a SharePoint addin. I will not develop & train the network from scratch as this is a time consuming process and TensorFlow community already published some models for common practical problems. TensorFlow is a bit hard and verbose and there nice abstractions that can make it more approachable and we still have the option to develop our custom models if needed. But for our case here I would like to stand on the shoulders of some giants and make use of something tested and trained.

So why do not we just use Azure ML or cognitive services vision API:
+ Maybe we have an architecture that is not allowed to call cloud services for security reasons or so.
+ The problem we are solving might need a custom model that is not available from providers like Microsoft & Google.
+ Some solutions need very fast response so models can be deployed and consumed directly on mobile phones for example.
+ An extra tiny benefit of using .NET on linux as we will see.

## Problem Description