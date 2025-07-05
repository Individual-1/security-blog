---
slug: "llms-and-security-1"
title: "LLMs and Security: The user's perspective "
date: 2023-03-29
draft: false
---
LLMs have been a hot topic these last few months with OpenAI's ChatGPT, Google's Bard, and Meta's LLaMA all being released (or leaked) for the general public to play with. Plenty has been said about the usefulness and applications of this technology, but less so on the security side of things.

In this post, I aim to provide a brief threat model of some common LLM use cases from the user's perspective and what considerations a user will need to take when utilizing or integrating with these models.

Future posts will cover other aspects such as integrators (first and third-party), and for running the model infrastructure itself.

<!--more-->

## Defining our target
The first thing we want to do here is to scope and define what exactly our system under test and principals of interest are. In this case, we will consider a generic LLM run on a corporation's hardware and accessible via API or web interface to a user providing prompts. This user will not be training the model themselves or feeding it large amounts of their own data as a training set (outside of what they provide via prompts). We will also consider the infrastructure in scope only to the degree that user data touches it.

As we are thinking from the user's perspective, our adversaries will be other users or the LLM infrastructure itself, as well as any integrations the LLM may have on the backend. We will not be considering attacks against the LLM from the LLM's perspective, that will be saved for future articles.

## Risk Surfaces

Given this target definition, let's think a bit about where risk may actually come from.

Our primary interaction point will be the prompt API or interface by which we submit queries and receive responses. This interface may require some form of authentication and authorization via provided credentials.

From the user's perspective, we feed in some data to this external interface, and get some magic out. This output will most likely be in plaintext, but for the sake of argument let's assume that it may be a binary blob as well.

There may be other surfaces secondary to the LLM's function such as a billing or financial incentive interface, but we'll consider those to be out of scope as the threat model is the same as any other financial services company (assuming the LLM is not ingesting this financial data).

## Threats

Starting with the user's data, let's trace the path a user query may take through the system. We'll also cover how cross-customer or malicious integrations may influence things and what we should be cognizant of.

First, we have the user's authentication and authorization to utilize the interface. This gives the hosting infrastructure the ability to tie a given query to some identifier (whether explicit via an authenticated principal, or implicit from anonymous assignment and request markers). We now have some potential ties between any questions or inputs we may be asking and ourselves, with only the company's privacy policy and compliance with legal requests preventing others from accessing it.

The primary concern here would be the attribution aspect, whereby if we are sending sensitive or proprietary data it may be associated with us and have secondary reprecussions (as with providing extra data to any company, like DNA companies and selling results or social media working with authorities). Some companies like Google takes steps to anonymize and redact any sensitive information coming in to reduce liability and risk to customers, but this kind of behavior is not a given and also difficult to validate from the outside.

Second, we have the query data itself. The contents of this message will be stored by the hosting infrastructure and potentially fed into the model and other analytics systems. We should consider that anything we send into the model will not remain private to us, whether intentionally or inadvertently like with ChatGPT's early authorization issues leading to queries being displayed to other users.

We should assume that anything we input into this model will be accessible to other parties, and consider what the implications would be if the content of our queries were leaked (loss of IP, PII, etc). We should also be cognizant of our input being used as further training data, which would then expose it to other users if it is returned as a response.

Last, is the data being returned as a response. Due to the black-box nature of a lot of LLM operations, we can't expect any real guarantees on the integrity, veracity, and safety of the data being returned. As a broad recommendation, users should consider the output of an LLM as untrusted data and potentially malicious and handle it accordingly. This is not only from the traditional malicious payload perspective whereby the returned info is designed to subvert our system protections, but also from the angle of the information being subtly wrong in a malicious manner.

In short, LLMs are prone to hallucinations and are trained on a largely untrusted dataset. It will never be possible to fully curate a dataset to ensure that everything is verifiable truth or safe, and even if it were possible the general tendency of the models to string unexpected things together could very easily bypass that.

## Conclusion

This has been a brief walkthrough of some threats the average user may face when working with an LLM. Because we heavily restricted the scope and interaction points we have fairly limited avenues for things to go wrong, but even with that simple interface it is important that we remain cognizant of just how many moving parts there are behind that simple prompt and what considerations we should take when making use of it.