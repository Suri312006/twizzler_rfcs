- Feature Name: Toolchain Cache
- Start Date: 2025-07/11
- RFC PR: [twizzler-rfcs/rfcs#0000](https://github.com/twizzler-operating-system/rfcs/pull/0000)
- Twizzler Issue: [twizzler-operating-system/twizzler#0000](https://github.com/twizzler-operating-system/twizzler/issues/0000)

# Summary
[summary]: #summary
<!-- One paragraph explanation of the feature. -->

I propose we utilize github releases as a cache to store compressed versions of
the twizzler toolchain. In addition to this, we would extend xtask in order to
perform some quiet toolchain management inorder to avoid having the toolchain
built locally, as well as some cli subcommands to manage edge cases, such as a cached
version not being availible. Certain Twizzler repositories, such as [rust](https://github.com/twizzler-operating-system/rust),[abi](https://github.com/twizzler-operating-system/abi), and
[mlibc](https://github.com/twizzler-operating-system/mlibc), on every update to main would require a rebuild of the toolchain and pushing
it onto github releases.


# Motivation
[motivation]: #motivation
<!-- Why are we doing this? What use cases does it support? What is the expected outcome? -->

The current twizzler toolchain bootstrapping process has quite a few downsides.
It takes a really long time to build, takes up a lot of space, and serves as a
roadblock for easy developer environments. Ideally this would make it so
onboarding new people would be a lot less cumbersome, as well as allowing for
possible docker / vagant developer environments, which would make onboarding
even easier. Additionally a toolchain cache would make CI run a lot faster, a
huge QOL improvement.
 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


<!-- Explain the proposal as if it was already included in the system and you were teaching it to another Twizzler programmer. That generally means: -->

<!-- - Introducing new named concepts. -->
<!-- - Explaining the feature largely in terms of examples. -->
<!-- - Explaining how Twizzler programmers should *think* about the feature, and how it should impact the way they use Twizzler. It should explain the impact as concretely as possible. -->
<!-- - If applicable, provide sample error messages, deprecation warnings, or migration guidance. -->
<!-- - If applicable, describe the differences between teaching this to existing Twizzler programmers and new Twizzler programmers. -->


Ideally this solution would work invisibly with regards to intervention from a developer.  

Here are a few error messages / prompts a Twizzler programmer would receive when having to interact with the system.

If there is a hash mismatch but the ideal toolchain already exists locally.
```
  Swapping toolchain to hash <hash>!
```

If there is a hash mismatch and the ideal toolchain exists as a Github release.
```
Active toolchain cannot be used to compile due to hash mismatch. Active hash: <Hash> , Ideal hash: <Hash>.
Toolchain exists remotely, would you like to install this toolchain alongside the active one? (Will replace active toolchain if no) [Y/n]

```

If there is no toolchain locally or remotely that can build the required toolchain.
```
WARN!!!
No cached toolchains can build twizzler!
Building will requite a local compilation of the twizzler toolchain.
This process will require around 40-50 Gb and will take a substantial amount of time.
Proceed? [Y/n]
```


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<!-- This is the technical portion of the RFC. Explain the design in sufficient detail that: -->

<!-- - Its interaction with other features is clear. -->
<!-- - It is reasonably clear how the feature would be implemented. -->
<!-- - Corner cases are dissected by example. -->
<!-- The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work. -->

The build workflow would look like:

1. There would be a github action on any sensitive repositories that would impact the toolchain, which would trigger a recompile on
    our current CI system.

2. The github action would prune any unnessesary files from the final output, and then compress with Zstd.

3. The final binary would be tagged with a hash of the `rust`, `abi`, and `mlibc` crates, and released on the twizzler Github repository.


The developer workflow would look like:


1. The first step in the current build process is `git submodule update --init --recursive`, but this would clone the `mlibc` and `rust` submodules
  inside the toolchain.
2. On initial pull, `cargo bootstrap` would download the cached toolchain instead of building from scratch. 

3. Whenever there is a build command in xtask, we first check if the hash of `mlibc`, `rust` and `abi` checked out locally match
the hash of the currently installed toolchain.
  - If the hash matches, build proceeds as normal.
  - If the hash doesnt match, we check if there exists a toolchain locally with that hash
    - If the local toolchain exists, we silently swap out the current toolchain with the one with proper local toolchain, and compile. 
    - If the local toolchain doesnt exist, we check if a github release exists with that hash.
      - If the release exists, we inform the user that to continue building, download of the new release is required, and we give them the
      option to install this new toolchain over the current toolchain, or we can install it along with the other toolchains.
      - If the release doesnt exists, we warn the user that building will require a full compiliation of the toolchain, and if the user agrees, we proceed to
      clone the `rust` and `mlibc` submodules, build traditionally, and then prune the resulting files.
      

# Drawbacks
[drawbacks]: #drawbacks

<!-- Why should we *not* do this? -->

It would take a bit of work in order to set this system up, as well as making sure it works as intended.
This also edges quite close to githubs maximum file-size limit so we might have to find a different solution for hosting.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- - Why is this design the best in the space of possible designs? -->
<!-- - What other designs have been considered and what is the rationale for not choosing them? -->
<!-- - What is the impact of not doing this? -->

The core of the problem is the current way of distributing the toolchain takes too long to build, and takes up so much space. Thus we need
some way of pruning the toolchain of useless build artifacts. It would make onboarding so much easier if the upfront storage / time cost was
smaller. I belive this is a sensible first step in the right direction, mainly because RedosOS solves the problem in a similar way. If we don't find
a solution to this problem, it would be an inconveince for new folks, as well as general development.

# Prior art
[prior-art]: #prior-art

<!-- Discuss prior art, both the good and the bad, in relation to this proposal. -->
<!-- A few examples of what this can include are: -->

<!-- - Does this feature exist in other operating systems and what experience have their community had? -->
<!-- - For community proposals: Is this done by some other community and what were their experiences with it? -->
<!-- - For other teams: What lessons can we learn from what other communities have done here? -->
<!-- - Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background. -->

<!-- This section is intended to encourage you as an author to think about -->
<!-- the lessons from other systems, provide readers of your RFC with a -->
<!-- fuller picture.  If there is no prior art, that is fine - your ideas -->
<!-- are interesting to us whether they are brand new or if it is an -->
<!-- adaptation from other operating systems. -->

<!-- Note that while precedent set by other operating systems is some -->
<!-- motivation, it does not on its own motivate an RFC. -->

RedoxOS does a similar thing as this proposed RFC, except that instead of using github releases, they have a build server to fetch the
patched binaries of LLVM, Rust, and GCC [source](https://doc.redox-os.org/book/build-phases.html). It looks like they utilize podman to
build RedoxOS, and then extracting the final image out and then run that with QEMU.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged? -->
<!-- - What parts of the design do you expect to resolve through the implementation of this feature before stabilization? -->
<!-- - What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->

A few things I hope to resolve with this RFC is the granularity of caching, as its possible for different developers to be 
working on different toolchain versions. The answer to this question would depend on the flexibility provided to us by github. 

Another question would be whether its possible to have our own build server now or in the immediate future?  

# Future possibilities
[future-possibilities]: #future-possibilities

<!-- Think about what the natural extension and evolution of your proposal -->
<!-- would be and how it would affect the operating system and project as a -->
<!-- whole in a holistic way. Try to use this section as a tool to more -->
<!-- fully consider all possible interactions with the project and system -->
<!-- in your proposal.  Also consider how this all fits into the roadmap -->
<!-- for the project and of the relevant sub-team. -->

<!-- This is also a good place to "dump ideas", if they are out of scope for the -->
<!-- RFC you are writing but otherwise related. -->

<!-- If you have tried and cannot think of any future possibilities, -->
<!-- you may simply state that you cannot think of anything. -->

<!-- Note that having something written down in the future-possibilities section -->
<!-- is not a reason to accept the current or a future RFC; such notes should be -->
<!-- in the section on motivation or rationale in this or subsequent RFCs. -->
<!-- The section merely provides additional information. -->

I think in the long run we could possibly move to a dedicated CI / build server so we have
more freedom over the amount of things we can cache and even faster CI runs. Other than that,
I cant seem to think of anything.
