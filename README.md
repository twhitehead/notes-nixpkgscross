# Concerns with nixpkgs cross-compilation dependency propagation

I've tried to wrap my head the cross-compilation [description of how dependencies propagate in
nixpkgs](https://nixos.org/nixpkgs/manual/#ssec-stdenv-dependencies) a few times now, and finally sat down and
wrote out all the possibilities. In doing so, I concluded that the described algorithm is much more complex than it
needs to be and it makes some corner cases errors in both propagating dependency build `build,host,target` triplets
and occasionally dropping dependencies. This writeup is to (hopefully)

* explain the system,
* demonstrate the issues, and
* propose a solution.


# How it seems to me it must work to be correct

Derivations expressions give packages build dependencies. These depedencies must be present in order to build that
package (e.g., libraries, etc.). Each dependency can further stipulate other (propagated) dependencies that are
required to be present whenever it is required to be present. What is required outside of buildtime is a separate
issue (e.g., this may be determined by scanning the build results for any _/nix/store/..._ path strings).

To build a package it also nessesary to specify up to three architecture specific items (only the GNU compiler
collection generally requires the third)

* build: the architecture the program will be built on
* host: the architecture the built program will run on
* target: the architecture the built program will generate code for

Technically each of these is actually a system triplet (`cpu-vendor-os`), but to save typing we will just use an
abreviated `cpu` name in our examples. We will also refer to a complete `build,host,target` specification as a
build triplet, where `,` delimination is intended to reduce confusion with the `-` deliminated `cpu-vendor-os`
triplet.

Given a specification of this triplet for a package (we will use angle brackets for the package), say
`<build>,<host>,<target>=x86,arm,power`, the next question is what triplets should its dependencies be built with
(we will not use angle brackets for the dependencies). The `build` part is easy as we want to be able to do all
required building (the package and its dependencies) on a single machine, so everything must have a common `build`
value, implying `build = <build>` (`x86` in our example). Remembering this will turn out to be a key insight.

This leaves the `host` and `target` specifications. It turns out these depend on the nature of the dependency. A
build system dependency, like `gnumake`, generally needs to run on the `build` machine, and so it will require
`host = <build>` (`x86` in our example). A library dependency, like `libssl`, generally needs to run on the `host`
machine, so it needs to be built with `host = <host>` (`arm` in our example).

Considering all possibilities gives six potential useful matchings (i.e., `build <= host <= target`) of a
dependencies `host,target` specification to the packages `<build>-<host>-<target>` specification. Which matching is
used for a dependency is identified by which of the `deps${host}${target}` dependency lists the package writer
lists it in

* `depsBuildBuild` - built with `<build>,<build>,<build>` (e.g., `x86,x86,x86`)
* `depsBuildHost` - built with `<build>,<build>,<host>` (e.g., `x86,x86,arm`)
* `depsBuildTarget` - built with `<build>,<build>,<target>` (e.g., `x86,x86,power`)
* `depsHostHost` - built with `<build>,<host>,<host>` (e.g., `x86,arm,arm`)
* `depsHostTarget` - built with `<build>,<host>,<target>` (e.g., `x86,arm,power`)
* `depsTargetTarget` - built with `<build>,<target>,<target>` (e.g., `x86,power,power`)

The pattern here is that the `deps${host}${target}` name gives which of the `<build>,<host>,<target>` package
triplet specifications are used for the package's `host,target` specifications (i.e., `host = <${host}>` and
`target = <${target}>`). The historical `nativeBuildInputs` and `buildInputs`, which are by far the most common
situations, correspond to `depsBuildHost` and `depsHostTarget`, respectively.

Build dependencies can also be propagated by putting them in one of the `deps${host}${target}Propagated` lists (the
historical names for these are `propagatedNativeBuildInputs` and `propagatedBuldInputs`). When a package with
propagated dependencies is depended upon, both it, its propagated dependencies, their propagate dependencies, and
so on, must all be present to complete the build.  An example would be a package whose header files include a
dependencies header files. This dependency would have to be propagated to make it present too anytime it was a
dependency in a build.

How do we figure out the `build,host,target` triplets for propagated pacakges? To answer this, consider a package
with a dependency that has a propagated dependency. Which of the package's `deps${host}${target}` lists the
dependency appears in tells us how the dependency's `host` and `target` values are taken from the package's
`<build>,<host>,<target>` triplet. Likewise, which of the dependency's `deps${host}${target}Propagated` lists the
propagated dependency appears in tells us how the propageted dependency's `host` and `target` values are taken from
the `<build>,<host>,<target> triplet inferred for the dependency. And so on for any of its propagated dependencies.

That is, there is no flexibility here. The determination of each packages `build,host,target` triplet is fully
specified given that `build` is fixed by the fact that all building needs to be do able on one machine and all the
`host` and `target` dependency to parent package mappings have been specified. All that can be done is to apply the
specified mappings recursively and arrive at the required `build,host,target` triplets.


# How it is decribed as working

The nixpkgs manual [gives a different algorithm](https://nixos.org/nixpkgs/manual/#ssec-stdenv-dependencies) that
seems much more complex. In this explanation there is a `mapOffset` function that transfers a packages
`<build>,<host>,<target>` values to its dependencies' `host,target` values.

```
mapOffset(h, t, i) = i + (if i <= 0 then h else t - 1)
``` 

where

* `h` is for the package `<host>` value
* `t` is for the package `<target>` value
* `i` is for the dependency's `${host}` or `${target}` mapping request

and these variables have one of three possible values

* -1 = build
* 0 = host
* +1 = target

For `h` and `t` the three possible values refer to the `<build>,<host>,<target>` triplet specificaiton of the
top-level package being built regardless of how far down the dependency stack we are. For `i` they refer to which
of the three possible values `${host}` or `${target}` has in the `dep${host}${target}` list the dependency appeared
in. Whether it is the `${host}` or `${target}` value is clear in the use of `mapOffset`.

Putting in the three possible `i` values reveals how a dependency maps to its parent package triplet

* package's `<build>`: `mapOffset(h,t,-1) = -1 + h = h - 1` (oops)
* package's `<host>`: `mapOffset(h,t,0) = 0 + h = h` (correct)
* package's `<target>`: `mapOffset(h,t,+1) = 1 + t - 1 = t` (correct)

As noted, this makes sense for `<host>` and `<target>`. The dependency gets what it wants. It goes strange on
`<build>` though. In this case the algorithm just appears to guess whatever value proceeded whatever part of the
top-level-package-being-built's triplet that the parent's `<host>` referes to is the desired `<build>`.

There is no need to guess though. As mentioned earlier, we want to be able to do all required building (a package
and all its recursive dependencies) on a single machine. This requires `build = <build> = -1` all the way up to the
top-level package being build (`x86` in our example). The implication of all this is that algorithm described in
the nixpkgs manual will go astray anytime a depenency requests that parent's `<build>` value and `<host> - 1` is
not `-1`.


# An example of things going wrong

From the above analysis, we know how to setup an example where things go worng. Consider a complier A being built
as follows

A: x86-arm-power

- we are building it on an x86 machine
- it is going to run on an arm machine
- it is going to produce code for a power machine

The compiler A provides a library B to the programs it compiles that dynamically generates code at the runtime of
the compiled program to run (e.g., an OpenCL CPU implementation, etc.). As per the above, the programs A compiles are
compiled for the `power` machine. This implies library B must be built using

B: x86-power-power

- we are building it on the x86 machine
- it is going to run on the the power machine
- it is going to produce code for the power machine

Library B, however, is actually built upon tools provided by a package C that need to be run when building against
B (e.g., a specialized pre-processor, etc.). This implies B needs to propagate the C dependency and

C: x86-x86-power

- we are building it on the x86 machine
- it is going to run on the x86 machine
- it is going to produce code for the power machine (technically not used in this case)

The dependencies for these are then as follows

A: depsTargetTarget = [B]
B: depsBuildHostPropagated = [C]

If we evaluate these using the `mapOffset` deduction system, we get

```
dep(1,1, A,B)              -- B selects A's Target and Target values (`power,power`)
propagated-dep(-1,0, B,C)  -- C selects B's Build and Host values (`x86,power`)
1-1=0 in {-1,0,1}
1+0=1 in {-1,0,1}
---
propagated-dep(mapOffset(1,1, -1),mapOffset(1,1, 0), A,C)
  = propagated-dep(0,1, A,C)
---
dep(0,1, A,C)              -- wrongly implies C select's A's Host and Target values (`arm,power`)
```

which isn't going to work as we are going to windup with a version of C built for the `arm` that is not going to
provide the built-time tools we require on our `x86` build machine to integrate library C into the package A.


# Another example of where things go wrong

The other corner cases occur when dependency B is part of `depBuildBuild`, `depBuildHost`, or `depBuildTarget`. In
these cases when dependency C selects B's `<build>` value, the `mapOffset` generates a `-2` value which is taken to
mean that the dependency should be dropped (see the nixpkgs description). This doesn't make sense either
though. Consider

A: x86-arm-power

- we are building the program on the x86 build machine
- it will run on the arm machine
- it produces code for a power machine

This program is built using a program B that is a typical compiler build dependency

B: x86-x86-arm

- we are building the complier on the x86 build machine
- it will run on the x86 build machine (were A is being built)
- it will produce code for the arm machine (were A is going to run)

The compiler has a meta-programming feature provided by another program C whereby it runs some code at build time
to produce the final code that has to be built at build time (e.g., typical of highly optimized math libraries)

C: x86-x86-x86

- we are building the meta-programming program on the x86 build machine
- it will run on the x86 build machine (where A is being built)
- to produce code that will also be run on the x86 build machine (where A is being built)

This gives

A: depsBuildHost = [B]
B: depsBuildHostPropagated = [C]

Evaluating these with the `mapOffset` deduction system we get

```
dep(-1,0, A,B)             -- B selects A's Build and Host values (`x86,arm`)
propagated-dep(-1,0, B,C)  -- C selects B's Build and Host values (`x86,x86`)
-1-1=-2 not in {-1,0,1}
```

so dependency C is simply dropped and not made available at A's build despite being a critical component required
by B to build A's code.


# Thoughts

Upon reflection of the nixpkgs document, I think part way through design of this system two orthogonal items were
conflated: what a package's built triplet is, and whether that package is required for a build. In reality, these
are two distinct things that are entirely specified in the set of user-provided pacakge derivations. That is

* a package is required for a build if it is part of the dependency closure, and
* dependency's build triplets is entirely constrained by which `deps${host}${target}` or
  `deps${host}${target}Propagated` list it appears in.

We cannot remove any dependencies based on details of the triplet specification to which they were built.  Only the
derivation writer knows how and why a package is required (e.g., it could be for a textual header file that is
entirely independent of the triplet to which it was built). Likewise, if the package writer said the `host` or
`target` of dependencies triplet requires the `<build>` value of the parent's triplet, then we have to take them at
their word. How that `<build>` value maps to the top-level-package-being-built's triplet cannot be allowed to
result in it getting a value other than the parent's `<build>` value.

Possibly the insight that was lost deep in the `mapOffset` algorithm was (again) that there is only one valid
`build` value: the architecture the `nix-daemon` is running on. There is no specification of how a dependency's
`build` value it to be taken from the package's triplet as it isn't. Adjusting `mapOffset` accordingly gives

```
mapOffset(h, t, -1) = -1 = <build>
mapOffset(h, t, 0) = h = <host>
mapOffset(h, t, +1) = t = <target>
```

This change also means `mapOffset` can no longer stray outside the intial set of `{-1, 0, +1}`. This means it no
longer drops dependencies, so both problems are actually solved. Also, as it now just propagates the requested
dependency (`-1` means propagate `<build>`, `0` means propagate `<host>`, and `+1` means propagate `<target>`) it
is fully unified with apply-the-user-specified-propagations-recursively understanding of how things should work
too.

Given how much easier it is to understand the statement that `build` is fixed and we recursively propagate the
specified `host` and `target` mappings then to infer this from the [natural deduction
explanation](https://nixos.org/nixpkgs/manual/#ssec-stdenv-dependencies) with the above `mapOffset` definition, I
feel strongly the entire deduction system explanation should just be dropped in favour of the proceeding sentance.

I realize it probably took a lot of work to work out, and I hope this doesn't feel like I'm attacking the briliance
of the cross-build system. It is truly fantastic. I love it. I dug into it to understand it, and I'm only posting
this in hopes of addressing an issue I think exists in some corner in hopes of making it even a bit better.
