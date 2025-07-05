---
slug: "supply-chain-slsa"
title: "Supply Chain Security and SLSA"
date: 2023-03-26
draft: false
---

In [Thoughts on Supply Chain]({{< ref "supply-chain-thoughts.md" >}}) we talked a bit about how to think about supply-chain security in the abstract.

In this post, we'll anchor our discussion against a set of guidelines called [SLSA](https://slsa.dev/).

<!--more-->

As the [introductory page](https://slsa.dev/spec/v0.1/) puts it:
> SLSA is a set of standards and technical controls you can adopt to improve artifact integrity, and build towards completely resilient systems. 
> It’s not a single tool, but a step-by-step outline to prevent artifacts being tampered with and tampered artifacts from being used, 
> and at the higher levels, hardening up the platforms that make up a supply chain.

For our purposes, we care primarily about the different pillars (Build, Source, Deps) SLSA defines, as well as the level requirements for each. Every risk we defined maps to an SLSA pillar, and each control an incremental step towards the next level of SLSA compliance.

## SLSA Pillars

In this section we'll go through each pillar of the SLSA framework and call back to the example we used in the first post. We do this to give each theoretical risk we defined a concrete backing in SLSA guidance.

### Source

[SLSA Requirements](https://slsa.dev/spec/v0.1/requirements#source-requirements)
- Version controlled
- Verified history
- Retained indefinitely
- Two-person reviewed

The Source pillar focuses on risks to our source code storage and how we can maintain the integrity (both past and present) of the code base. 

In our example, we talked a bit about revision control in the [Source code storage]({{< ref "supply-chain-thoughts.md#source-code-storage" >}}) section. We mentioned a couple situations which could compromise the availability and integrity of the code via disk failure or system compromise, and suggested utilizing a resilient revision control system to help defend against the former and help mitigate the effects of the latter. However, we didn't come up with anything to prevent the compromised commit in the first place. Fortunately, SLSA has a requirement which will help bridge that gap.

But first, before we get to that recommendation, let's take a look at how we stack up against the other requirements.

For the first item "Version controlled", we get that by virtue of using a revision control system in the first place. If we had stuck with our old design of keeping the project on disk and emailing things around we would have failed this requirement, but by using git or Perforce or any other option we tick this box.

With the second item "Verified history", things become a bit less clear. This requirement asks that we timestamp and strongly authenticate some party associated with each changelist. This may be something which our revision control system provides, but depends heavily on our configuration. We'll need to be sure to configure it in such a way that we clearly specify which identities have been strongly authenticated on a per-commit basis. An example of where this may not hold is with default git configurations, where a user can specify any email and username to be associated with a commit.

For the third item "Retained indefinitely" we get into the configuration component again. We must ensure that we configure our revision control system to maintain all history for the required time period, and prevent deletion or rewrites. Some revision control systems such as git allow for mutable history by default, and would need further configuration to prevent such changes.

Finally we get to the requirement alluded to before. With "Two-person reviewed", we increase the bar to actually getting malicious code into the repository. This was not something we covered in the original post, leaving that gap. With this requirement in place we make it so that malicious compromise needs to have an insider supporting them or otherwise compromise multiple identities with permission to modify the repository.

### Build

[SLSA Requirements](https://slsa.dev/spec/v0.1/requirements#build-requirements)
- Scripted build
- Build service
- Build as code
- Ephemeral environment
- Isolated
- Parameterless
- Hermetic
- Reproducible

The Build pillar focuses on securing the build process and ensuring the safety of our generated artifact.

Our risk discussion in the [Manual build and deploy]({{< ref "supply-chain-thoughts.md#manual-build-and-deploy" >}}) and [Adding CI/CD]({{< ref "supply-chain-thoughts.md#adding-cicd" >}}) map most closely to this pillar. In those sections we went from typing out our compile command each time to automating the whole build with a continuous integration system. This represents a rather significant shift, and that is reflected in how many SLSA requirements will be covered in that gap.

Starting from the top with "Scripted build". This requirement asks that we make our builds consistently re-runnable by scripting out the build steps in a Makefile or some other method. We did not start with this as we were retyping our compile command by hand, but in moving to a build framework or CI/CD setup we get the reproducibility from having to tell our toolchain how to build our project.

The second requirement "Build service" is also a freebie when it comes to our CI/CD setup (unless we are running an instance of the CI/CD tooling on our workstation). This requirement just asks that we not run the build process on our local system, or at least not for the artifacts we'll be deploying.

Third we have "Build as code". This asks that we define our build steps and process as some form of text or other definition which is revision controlled. This allows us to verify the steps which are taken and control changes to them through defined means. Most CI/CD systems will support some form of this, and it is just a matter of using the right configuration with our desired toolchain to check this box.

Next, we have "Ephemeral environment". This requirement states that we should run the build process in an instance or environment solely for that build invocation. This should also be a capability of most CI/CD frameworks, but will depend on the user configuring them properly.

After that we have "Isolated". Somewhat related to the prior item, this requires that our build process must be self-contained and uninfluencable by other build instances or infrastructure. Again, this is going to vary from framework to framework and will depend on us configuring it properly.

Similar to the build definition requirement, we have "Parameterless". This requirement states that our build should be almost entirely defined in code and not be influenced by defineable variables or such outside of some basic items. Meeting this requirement will rely on us being careful not to rely on such parameterized functionalities when writing our build file definition.

Second to last is the "Hermetic" requirement. This asks that our build process be run in an entirely self-contained manner with all references being defined beforehand in an immutable manner. This is a tricky requirement to meet, and doing so will require configuration specific to whichever CI/CD framework is being used.

Lastly is the trickiest of the bunch, "Reproducible". With this requirement, we must ensure that two runs of the same build inputs should result in bit-exact outputs. This isn't something that is necessarily feasible with all (or even most) CI/CD frameworks or local build tooling, and requires thought if we actually want to meet it. On the local build side, this is where we could use something like Bazel to get more cleanly defined hermetic and reproducible builds.

### Provenance

[SLSA Requirements](https://slsa.dev/spec/v0.1/requirements#provenance-requirements)
- Available
- Authenticated
- Service generated
- Non-falsifiable
- Dependencies complete

Content Requirements
- Identifies artifact
- Identifies builder
- Identifies build instructions
- Identifies source code
- Identifies entry point
- Includes all build parameters
- Includes all transitive dependencies
- Includes reproducible info
- Includes metadata

Provenance is a bit of an outlier pillar in that it doesn't cover a specific part of the process but rather the auditability of the whole flow.

Accordingly, the [Toolchain and packaging risk]({{< ref "supply-chain-thoughts.md#toolchain-and-packaging-risk" >}}) and [First and third-party dependency risk]({{< ref "supply-chain-thoughts.md#first-and-third-party-dependency-risk" >}}) sections fall under this category. To sum them up without repeating everything, we discussed the issues with relying on different kinds of 3p packages, how to ensure we are getting what we expect, and what guarantees different mitigation options would get us.

The direct SLSA requirements are all fairly specific to provenance metadata, which is not something we covered at all so we'll gloss over those and just say we fail all of thos requirements. Of more interest to us are the content requirements for the provenance declarations, which we can partially cover with our mitigations. Given the number of content requirements, we'll focus only on those which we can comment on with our risk assessment, and refer the remainder for future analysis if desired.

The first requirement we can speak to is "Identifies artifact". This asks that we cryptographically identify the output artifact. For our purposes this is partially covered by our usage and discussion of signature verifying dependencies. We ensure we are getting what we want dependency-wise by storing and verifying a cryptographic signature for each dependency, which can be thought of as verifying the provenance of that dependency (although it says nothing about their build process or any of the other tenets we have been talking about).

The second requirement is "Identifies entry point". This isn't directly covered by our discussion, but is touched upon as part of our usage of version pinning and some version specifiers depending on the language. Some language ecosystems will include upstream information as part of a version tag, whether it is a git commit hash or other identifier which can be mapped back to a specific point in time. Not a perfect solution, but something we can get part of the benefit of.

The last requirement we can partially cover is "Includes all transitive dependencies". Some manifest formats for language toolchains will automatically include all transitive dependencies with version pinning and signature verification. This is the case for Golang, and may be available for other languages either via the standard package management tooling or via supplementary frameworks.

## Conclusion

Now that we have mapped some of our discussions back to a much more concretely defined framework, hopefully you have a better understanding of just how supply-chain security problems fit together and what an end-to-end solution may look like. There are certainly options and frameworks other than SLSA, but for our purposes it provides a good mix of guidance and definition without being too prescriptive such that we can define our own solution.