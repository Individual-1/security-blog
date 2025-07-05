---
slug: "thoughts-on-supply-chain-security"
title: "Thoughts on Supply Chain Security"
date: 2023-02-20
draft: false
---
Supply chain has become a hot topic in the last few years, with high-profile vulnerabilities such as log4j leaving companies scrambling to patch in an ad-hoc manner.

For many, this exercise served as a wake up call; companies were made aware of just how little they knew about their supply chains. Some have tried to address this by building as much software in-house as possible (which still leaves a degree of insider risk), but that scales poorly and comes with its own caveats. 

As such, usage of some degree of OSS and vendor code is inevitable, and is in fact becoming reality within even the most not-built-here of big tech. Because of this, it is important that engineers think about the implications of bringing in these unknown trust packages and how they may affect the rest of the build and deploy process, not only for OSS and vendor code, but for their own code as well.

Unfortunately, there is no single correct way to approach these problems, since so much of it depends on your tolerance for risk and what kind of attacker profile fits into your threat model. Instead of trying to dictate **the one true way**, I'll discuss some building blocks which have proven useful for controlling risk, such as methods for increased transparency and more granular controls to guarantee the integrity of our supply chain. Because this isn't a one size fits all approach, it's up to the individual to assess what mitigations suit their needs. To support that, we'll outline situations in which each may be applicable. What makes sense for a megacorp doesn't necessarily scale down for a personal project.

<!--more-->

## Understanding Risk

Before we dive into specific controls, it would be useful to discuss a bit what exactly the risks we face are and why thinking about them is important. We will frame these in the context of a Python project to provide a frame of reference, but the concepts apply to any language or provider.

### Toolchain and packaging risk

So given our theoretical Python project, what can go wrong? Lets start with the most basic example: Hello World. 

```
def main():
    print('Hello World')
```

Surely nothing could go wrong here, it is only two lines! Well, think about what is actually going on when you are running this program and how we package it for delivery. 

As an interpreted language, we'll need a Python interpreter to actually execute this program. Along with that comes some implementation of the Python standard library providing a number of common functionalities and helpers like the `print` function.

What if our interpreter is compromised? Or there is a vulnerability in the standard library? If we vet the interpreter once, how do we know we get the same binary later even if we try downloading the same version?

As you can see, even with the simplest of programs there are hidden supply-chain and environment risks. This doesn't mean you have to obsess over every single layer of your execution and build stack though. Consider your threat model and personal risk profile and use that to figure out where you're comfortable.

### First and third-party dependency risk

Let's add a bit more complexity and import some more packages into our program to see how it changes our risk profile.

```
from random import random

def main():
    print('Random float: {}'.format(random()))
```

We'll start with a dependency from the standard library. We import `random` and use it to generate a float which we then print out. Not too much going on here, but now we have pulled in a module which wouldn't have been included otherwise. As a part of the standard library there should presumably be some form or additional confidence in the quality and integrity of the code, but as a matter of course we should still question it. 

How do we know this code isn't vulnerable to some exploit? Or that it isn't itself backdoored? This ties a bit into our concerns from the first section as this is included along with the toolchain when you download Python, but is a separate module which the interpreter will import from the installation.

```
> pip install random2

from random2 import Random

'''
I don't actually know if this would work, random2 docs are limited and I didn't want to install the module just for this example
'''
def main():
    print('Random float: {}'.format(Random()))
```

This time we install a random number generator with pip and use that to generate a number. This is our first time working with code that wasn't written by us or a Python contributor so what does that mean for us? Is this a developer that we can trust? Do they have robust code review and contribution policies? And so on.

We also need to consider the dependency retrieval process. By default pip does not verify integrity of packages and may run arbitrary code as part of building source-only distributions. This combined with the usual network security issues means that just by pulling in an external dependency we're opening ourselves up to network integrity, package integrity, and untrusted code concerns.

### Source code storage

So far we have just been writing all our code into a file on our local filesystem. This is fine for local prototyping and testing, but as soon as we want to start working with someone else or maintain a history of changes then we start to have problems. There are a number of options for revision control systems, but not all are created equal or provide the same guarantees.

For example, what happens if our disk crashes? Or someone compromises our dev environment and makes their own changes to the code? It is important to keep in mind both the availability and resiliency aspects when using revision control.

### Manual build and deploy

The next step after we reach the minimum viable product phase is to actually get everything put together and out there. We'll start by doing it all manually, both the build and deploy.

Python doesn't have a compile step so let's imagine we have a C++ codebase instead. Because we're being agile we don't have time to write a Makefile or shell script to build things so we just keep typing out our command on the CLI.

```
> clang++ -Wall example.cc -o example.out
```

To deploy our binary we email it to our devops person, who then SSH's into our prod server and drops it in place.

We have already covered toolchain concerns in an earlier section (reproducible builds, untrusted toolchain, etc), so let's think about the deploy aspect. For starters, email isn't a particular secure method of transferring a file, and we're relying on a human to get it where it needs to go. Think about what might happen if an external attacker could tamper with your delivery mechanism (and subsequently the deployed binary), then extend that idea to the greater access an internal adversary might have.

We're also having someone log directly into our prod server and mess around, which should throw up another million red flags. What stops them from doing something other than deploying? Or what do we do if something goes wrong with the environment?

### Adding CI/CD
We certainly could keep rebuilding manually off a dev machine and then FTP'ing that over to our prod server forever, but we heard about some fancy tools that could do it all for us at the press of a button so let's  enter the 21st century and integrate into a CI/CD system.

We won't call out any particular provider or software stack here, and instead just lay out some common properties/assumptions and work from that.

Our theoretical CI/CD provider:

- Is a third-party provider
- Has access to our source repository
- Has some credential storage or retrieval mechanism (for repo and deploy environment access)
- Executes arbitrary code as defined by some configuration file in our repo (ostensibly to build our project)

Given these parameters, let's think through what our risk profile looks like here. 

We have all of the usual 3p risks here except amplified due to us trusting them with our sensitive source code and access to our production environment. Even if we don't use a 3p provider and run a hosted instance of a CI/CD toolchain we still face a lot of the same issues, just exposed to a different audience.

So if someone has compromised our CI/CD infrastructure, what can they do? With source access we face potential info leaks or malicious source modification. If we configured credentials for repo/env access then we risk those getting leaked as well. Lastly, if an attacker has compromised enough to make changes to the build or deploy configurations then arbitrary code execution or other similar attacks become very real possibilities.

## Controls

With that (non-comprehensive) exercise out of the way, let's talk a bit about what we can do to mitigate some of these risks.

### Package Version Locking

As discussed in the [First and third-party dependency risk] section, one major risk of 3p dependencies is the fetching of an upstream package of unknown provenance. Even if we reviewed the library code and knew it was safe, that only matters for the particular version we looked at. A common supply-chain attack is to compromise the upstream dependency provider and upload malicious updates to dependencies, causing users to download the compromised version.

So how do we ensure that we get the same version each time? Most toolchains have some form of version locking whereby you can specify a version to download. This ensures (sorta) that you always get the same known revision.

This control is low-effort and easy to implement, and is accordingly suitable for projects of all sizes.

### Package Integrity Check

Even if we lock the package version, what happens if someone compromises the upstream and just replaces a known version of a dependency? Or maybe our transport layer gets man-in-the-middled and our known module version replaced with a malicious one on the fly.

This is where checksumming and integrity checks come into play. Some toolchains support checksums or signature checking for packages, giving us a mechanism for verifying that we have indeed downloaded what we expected.

If supported by your environment, this control is also low-effort and easy to implement. It is suitable for projects of all sizes.

### Dependency Vendoring

Version pinning and checksumming give us good baseline guarantees but don't do a lot of good if we still want to be up and running if something does go wrong.

If you can manage the maintenance and process overhead, then one solution to this is to vendor your dependencies. This means keeping a known good copy of the code or compiled library you rely on in your project, such that you always have a safe revision to use. Some toolchains can even handle all of this for you by automatically downloading and storing all dependencies in your repo as part of the build process, making it even easier.

Of course this isn't foolproof if someone else can get control of your environment and just swap out your local copy, but it does help mitigate some of the remote compromise concerns.

As alluded to, this also does have some tradeoffs you need to consider such as how difficult it is to adapt the vendored code to your project, or extra work required when updating the dependency.

Given these caveats, this mitigation is not necessarily suitable to all projects but can be useful if you want that additional layer of protection.

### Software Bill of Materials (SBOM)

Every mitigation we have discussed so far has been to protect ourselves against malicious dependencies by ensuring we always use a known good version, but what happens if someone finds a vulnerability in our "known good" copy and we need to patch it? In theory we should be able to go into our project's lockfile and start bumping up dependency versions to patched revisions, but how do we know we got them all?

This is where the idea of an SBOM comes into play. A Software Bill of Materials serves as an inventory of a given component's relationships with its dependencies. In theory if you have comprehensive SBOMs for your project and all of its dependencies have similarly comprehensive SBOMs then you can just crawl the tree to find usages of the vulnerable library and know exactly what you need to update.

Of course, this comes with pretty significant up-front setup time if starting from scratch. As time passes and more OSS projects embrace the SBOM concept it may become more straightforward to jump in and get started, but for now it will take a fair amount of dedicated effort.

With that in mind, I would suggest this mainly for large, well-resourced projects which can afford the effort.

### Reproducible Builds

With everything we have done so far we have developed a pretty strong baseline for our project, and we can start thinking about some add-on efforts.

The first of these is the concept of reproducible builds. Reproducible builds are the idea that you can run a build with a given set of parameters and always get the same output, hence the reproducible. By "same output" we don't mean just functionally similar (like if we changed out the optimization level), but a step further and getting a bit-exact reproduction (less any metadata).

This capability may not seem too useful on the face of it, but being able to generate the same output forever means we can verify a given build once and continue to have a known good even if we need to re-build it. As projects scale up and the need for consistency and reliability increase, the value of reproducible builds becomes readily apparent.

In most build systems reproducible builds are non-trivial to set up. Unless you really need this behavior, or are starting a greenfield project and think you may get to this point then don't worry about it.

### Hermetic Builds

If we think of reproducible builds as applying to the output of our project, then we can apply the same idea to our build environment to get the concept of hermetic builds. A hermetic build is a build in which we can generate and retrieve the same toolchain revision and any other supporting tooling as defined by our project to ensure that we get the same environment every time.

These tend to go hand in hand with reproducible builds out of necessity (different toolchains revisions will generate different outputs), and is the other piece of the puzzle to really getting consistent control and guarantees around our build process.

This is something that can be done manually with some difficulty if you can capture and regenerate a comprehensive snapshot of your environment, or via some tools such as Google's Blaze/Bazel.

As with reproducible builds, don't worry about this if you don't need it. It is nice to have for casual projects and development, but not necessarily to the degree of refactoring everything into a build system which supports it.