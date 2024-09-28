# Peeling apart the Pythons: Structure of Nublado images

```{abstract}

The Nublado component of the Rubin Science Platform provides users with a Python notebook execution environment that includes the Science Pipelines stack.
At present, the Python installation provided by the Science Pipelines conda environment is used to run both JupyterLab and for the notebook kernel.
This approach is relatively simple, but it tightly couples the JupyterLab UI with the notebook kernel in a way that complicates maintenance of the UI and makes it unnecessarily fragile.
This tech note proposes a new base Docker image for Nublado pods and a strategy for separating the Python environment used for the JupyterLab UI from the Python environment used for the notebook execution kernel.

```

## Motivation

At the time of writing (September 2024), Nublado lab containers are built on top of stack containers produced by Jenkins.
These containers are named `docker.io/lsstsqre/centos:7-stack-lsst_distrib-<tag>`, using the tag conventions described in {sqr}`059`.
Currently, we are using the Python included in the conda environment provided by the stack for both the JupyterLab UI and for the notebook execution kernel.

This approach has the following issues:

1. We want to support execution kernels other than the Science Pipelines stack.
   Some possible examples are a lightweight Python-only container, a machine-learning-oriented container customized to the needs of a particular collaborting group, or a kernel maintained by a different mission.

2. In operations, we will need to support older kernels, such as a two-year-old version of the Science Pipelines execution environment, with whatever JupyterLab is current at that time.

3. Sharing a Python execution environment between the user's kernel and the JupyterLab server means that the user can customize their environment in ways that break the JupyterLab server.
   When this happens, their Nublado lab may no longer be able to start until they reset their environment.

4. The Science Pipelines conda environment adopts new versions of Python relatively slowly, but newer versions of Python are often helpful when developing the JpyterLab environment.
   For example, we recently wanted to use {py:meth}`pathlib.Path.walk` in the JupyterLab startup code, which requires Python 3.12, but the Science Pipelines conda environment was still using Python 3.11.
   It is likely that the Science Pipelines conda environment will become even more conservative once we enter operations.

The recent switch of the Science Pipelines container from CentOS to AlmaLinux provides a convenient opportunity to make this change.

The remainder of this technote addresses some implementation considerations that support our motivations.

## Nomenclature

Within the context of this technote, we are going to assume two Pythons: the "Jupyterlab Python" responsible for providing the UI and browser-specific components to the Lab user, and the "Payload Python", packaged as a Jupyter kernel, which provides the Python environment that executes the user's notebook and can be used from an interactive shell.
There will likely be non-core-Python components that must be common to both (see below), but in general the contents of these different Python environments will be largely disjoint.

We are assuming that the payload is a Python environment.
Non-Python payloads are a future possibility, but are outside the scope of this tech note.

## Arbitrary Payloads

In considering support for arbitrary Nublado payloads (i.e. different flavors of container), the following questions arise:

1. How should we accept a definition of the payload?  A pip `requirements.txt` file?  A conda environment YAML file?  A supplied Docker container?  A `Dockerfile`?  An installation shell script?  Something else?  What style of definition will be the most convenient for publishers of alternative payloads?

2. What else will be required for this payload?  A tag categorizing and sorting class, similar to the existing `RSPTag`?  A set of environment variables that must be supplied?  A configuration YAML file?  A custom spawner menu?  Something else?

3. How do we maintain compatibility with prior versions of the Science Pipelines stack?  We must be able to create maintenance rebuilds of historical images with updated libraries and integration APIs and have them launch and perform correctly.

### Payload Specification

Except in the case where someone else provides a Docker image, the payload must be specified in such a way that it can be automatically built by our build chain.
Ultimately, it will be installed from a `Dockerfile`
That could be a single `RUN` or `COPY` command or a `RUN` command kicking off a complex script.
Alternately, SQuaRE could supply the base image and the maintainer of the payload could install their payload on top of that base image, following a documented interface that allows JupyterLab in the base container to locate and start the payload-supplied kernel.

There are two fundamental ways of providing different payload and Jupyterlab Python environments.

1. Install the Jupyterlab machinery on top of a supplied container.
   This is the way we're currently doing it, and it can be adopted for a two-Python configuration.
   Currently, we install everything into the `rubin-env` conda environment, but we could instead create a new conda environment for the JupyterLab runtime, with whatever version of Python is desirable, and install whatever JupyterLab needs into it.
   This creates a larger environment than the current status quo (as will all of these approaches) but is conceptually straightforward.

   A drawback of this method is that we have to tailor our installation approach to fit the base container.
   We will want system-packaged command-line tools, for example, so that people have their choice of editors and so forth.
   If we use the system machinery of the upstream container, this will require maintaining different build processes for each possible choice of system machinery in upstream containers: RPM-based systems, dpkg-based systems, Alpine systems using `apk`, and so forth.

2. Install a standard container and then install the payload on top of that container instead.
   For instance, `docker.io/library/python:3.12` is the official Python container, built on top of a Debian stable container.
   This will work if the payload isn't tightly coupled to a specific Linux system flavor by, for example, using RPM internally.
   In practice, this seems like a good assumption.
   Open source packages are usually portable across flavors of Linux, and even commercial ones will usually work on either RHEL or Ubuntu.
   The one exception is Alpine or other musl-based Linux containers, which should be avoided for this approach.
   This approach also creates a larger container, but keeps the system layer fairly simple compared to using an upstream one.

The first method is closer to current practice.
However, if a payload is not already packaged as a container, the second approach is required anyway.
The most worrying part of the first approach is that accomodating new payloads is hard, since it may require substantial new tooling for the new variation of Linux packaging.

The second method means, at least theoretically, that we use a single lab-building framework and drop in new payloads.
New payloads would only require changing the build stages that implement the installation and runtime launch framework of that payload.
The problem there is that there is no universal method of installing payloads.
However, it seems difficult to imagine that one of the following five techniques would not be applicable.

1. The payload is pip-installable: there's a `requirements.txt` file, the installer points `pip` at it, and the payload is installed.

2. The payload is conda-installable: there's an environment `YAML` file, the installer points `conda` at it, and the payload is installed.

3. The payload is delivered as an installation shell script.
   The Science Pipelines stack can be installed this way with [lsstinstall][2].
   This may be the most common way for thorny environments to be delivered.
   For anything of sufficent size that has made the (bad) choice to not use a standard build/installation system, such a script will probably already exist.
   See, for instance, the huge proliferation in products that expect you to `curl https://some-url | sudo /bin/bash`.
   Less grotesquely, maybe it's something you can install with `./configure; make; make install`.
   This approach will only work if the installation script does not assume a particular underlying distribution type such as dpkg or RPM, and if it limits its actions to a specific directory in the filesystem.

4. The payload is supplied as a Dockerfile that constructs a container image.
   The appropriate pieces of the Dockerfile can be moved into the Nublado-launchable image construction as `COPY` and `RUN` statements, or as a shell script which does the same thing.
   This has the same disadvantages as the above: it must not assume a particular method of package installation, and mustlimit the payload-specific installation to a specified directory.

5. The payload is only available as a Docker image, without build instructions.
   This is even more alarming and certainly suggests the first method would be a better idea, but if the payload is similarly limited to a directory (for instance, the Science Pipelines stack lives almost entirely under `/opt/lsst/software/stack`), a multi-stage Docker build could transplant that directory hierarchy into the target container.
   In practice this is likely to be difficult.

The best approach appears to be the second method (install the payload on top of a standard Nublado container), using the first or second techniques if possible.

That lets us keep almost all of the image-building machinery the same between payloads and makes replacement of the payload straightforward since it's isolated from the rest of the container build.
If the payload is in one of the easy formats, the payload installation script may be nothing more than `pip install -r requirements.txt`.

That sets up the third point: what will be necessary in order to turn the installed payload into a JupyterLab-runnable kernel and a terminal environment with the payload contents configured and available?

### Payload configuration

Configuration of the payload environment has at least three components.

Setting up the Jupyter kernel is the most straightforward.
If the payload can be represented in a Python virtual environment, `python -m ipykernel install` is all that needs to be done.
If the payload is more complicated (as the DM pipelines stack is), then it turns out that the kernel can be a shell script that does necessary setup and execs a Python process as its last action.  If this is necessary it will be the payload provider's responsibility to supply this.

The next is setting up a terminal environment.
One way to do this is to set a sentinel environment variable at Jupyterlab launch, and then when the user requests a shell, in the user's `.bashrc` or moral equivalent, test that variable, and if it is set (indicating the user is running inside Jupyterlab), perform the necessary setup (environment activation, running an appropriate setup script, whatever) before yielding control to the user for interactive shell use.

The third is the most difficult.
In Nublado, we have a spawner form.
That form contains a selection of prepulled images, a drop-down box to select an uncached image, a selector to choose container size, and two checkboxes useful for debugging and unwedging inoperable environments.
I am not actually sure how generic that is, but I think erring on the side of "everyone will want something like this" is better than insisting it's DM-pipelines-specific.

The purpose of the form in the RSP environment is basically twofold: first, to let the user select how much they need in the way of resources, and second to reflect our release cadence.
We have, effectively, "daily", "weekly", and "release" images and would like some way to present those to the user and update them without human intervention as new images are built.

By having rules about how many of each we use, we can also decide which images to prepull.
Given our use cases, it is vital to prepull images, because they are so large that waiting for a spawn that needs to pull an image from a remote repository becomes an unpleasant experience.
If the images are cached to each node (which Nublado does), then for images displayed on the menu (rather than in the drop-down list of all images), the spawn is generally a matter of seconds rather than minutes.  We simply poll the image repository asking for the checksums of the appropriately-tagged images periodically.
Most of the time, neither the tags nor the content at each tag has changed, and in this case, there's nothing to do.
Only if a new tag matching our criteria is present, or if the checksum for a tag differs from what we have (Docker repositories, by design, let you move tags to different container contents), do we do any real work.

It is not entirely clear that the Rubin practice of builds-on-a-fairly-short-cadence, almost immediately reflected in the spawner, is what all users of Phalanx will want.
However, this form would work just as well with a single type of release and builds updated quarterly, although the mechanisms that populate it will be overcomplicated for the use case.

Assuming the spawner options form is fundamentally generic, then there needs to be some method of categorizing and sorting images.
Currently this is the `RSPTag` class, buried deep inside the Lab controller, which understands the tag conventions used by the DM pipelines container builds.  This scheme is overly baroque and is essentially a Rubin-DM-stack-specific idiosyncrasy.
We should update this class with a more generic name and, in the general case, only support semantic or calendrical versioning.
That said, we will need to retain the current sorting methods in order to retain backwards compatibility with our current installation.
We should, however, document that use of anything but semver and calver is not supported for any container other than the Rubin DM stack (and its counterpart for T&S, the SAL stack).

### Compatibility

Of course, for the RSP use case, if we switch to a new version of image construction, we must allow the spawner to continue to spawn older images.

There is a proof-of-concept implementation of such a thing in [the tickets/DM-44731 branch of sciplat-lab][3] which demonstrates this: effectively, we've created symbolic links or copies of various files such that they are present both at the old "everything under `/opt/lsst/software`" location and the newer "Jupyterlab machinery lives in `/usr/local/share/jupyterlab` but the DM pipelines stack is still in `/opt/lsst/software/stack`" locations.  The actual functionality to provide the compatibility layer is found in [the install-compat script run during `docker build`](https://github.com/lsst-sqre/sciplat-lab/blob/tickets/DM-44731/scripts/install-compat).

Minor modifications have been made to the spawner to allow the locations of certain items, like "what command is run inside a user container to start the machinery" to be parameterized in the spawner config, so in either the scenario where all the old-style images are so old we no longer care about supporting them, or the scenario where we're running a completely different payload and therefore have no reason to use the old paths, the spawner can work with only changes to the Phalanx configuration rather than changes to its code.

This also addresses the maintenance issue.
If we need to rebuild a current version of the container with an antiquated payload, we have the compatibility layer to assist.  The layout of all the non-payload components is free to change, as long as we have a way to link or copy the (fairly few) pieces back into their prior location.

## Proposed Implementation

We think something like [the proof-of-concept implementation][3] is the right direction.  This one is based on the current official Python 3.12 container, which is Debian-based.  It does not produce exactly the same stack container that its counterpart in the lsst-distrib-as-base-container model would, in that the rubin-env Conda environment is freshly installed each time the container runs [lsstinstall][2] and therefore will get newer versions of dependent packages (the decision not to hard-pin versions was taken by the Science Pipelines developers; the author disagrees with their choice, but above my pay grade, I suppose).

There is a potential problem--I am not sure of its gravity--in that Jupyterlab extensions are usually supplied as a frontend UI set of components, and a Python backend server component.
It is not clear to me that the two of these should always be in the Jupyterlab environment rather than one in the Jupyterlab Python and one in the Payload Python, or whether they should be installed in both, and if the answer is "both", then how does the frontend know how to talk to the correct backend?  
Will the backend, potentially running on an entirely different Python version, necessarily even be able to communicate with the frontend?
It is entirely possible I am overthinking this; it appears that installing all the extension-ish stuff and its dependencies into the Jupyterlab Python is working for us now.

Behaviorally this container appears functionally identical to the single-stack container in terms of what payload code it will run.
There is a change visible in the UI, in that I do not know how to hide the Python environment running Jupyterlab from the generated image.

The other major drawback is that the container image is almost 15GB rather than 10GB.
No attempt at optimization of packages that are no longer needed in the stack environment has been made, so there's probably some wiggle room.  And we could always give up on PDF export of notebooks, which introduces a surprising amount of complexity and storage space requirements into the UI Python environment.
It can probably be reduced to "not grossly larger than the single-stack container" with some work.

This approach has also been used with our (early July, 2024) proof-of-concept implementation of a SPHEREx Nublado deployment, done as part of a technology demonstration.
There, a payload container (from https://github.com/lsst-sqre/spherex-lab ) running the public (that is, non-export-controlled) parts of the SPHEREx analysis environment (supplied as a Conda environment specification) is demonstrated.

## Further enhancements

A further change to the spawner would help us move towards our goal of fully separating the Jupyterlab and payload environments:

Package the kernel definition, kernel helper shell script (if any), and script to set up and start JupyterLab in a ConfigMap.
Then the third point above collapses down to: "if you want to run your payload on a Phalanx Nublado instance, bring a configmap with these two or three items, with the following keys and content for each that you provide."

That would be accompanied by a fork or copy of the sciplat-lab repository, with at least the script currently called `install-dm-stack` (which should probably become something like `install-payload-enviroment`) replaced.
Changes will likely be needed to the dependency packages as well, in addition to any changes to the UI libraries and extensions (not all environments will need all of `plotly`, `matplotlib`, `bokeh`, and `holoviews`, for instance).
The rest of the image assembly and parameterization is already handled by the Makefile (which applies the image tag to a template and generates a Dockerfile; after the image is built it can then be uploaded to one or more image repositories), although producers of other payloads will want to change the defaults in the Makefile to produce images destined for a different repository.

It is certainly possible to imagine adding more parameters to the makefile which told it where to find various build stages and swapped them in, so that a single repository could be used to build many targets.
This would take careful design work.

We should replace the `RSPTag` class with a more generically-named one.
The "daily", "weekly", and "release" categories and the complex tag parsing would need to hang around for the DM Stack and T&S SAL use cases, but for other use we should provide semver or calver sorting within the Tag class and document that those are the only supported methods of container tag construction.
However, this work does not need to be done immediately.
Initially, we could simply point anyone interested in providing their own payloads to [sqr-059][1] and point out that only ever having `Release` versions with three-component versions gives them, effectively, semantic versioning or time-based sorting depending on how they structure their tags.

[2]: https://ls.st/lsstinstall "lsstinstall"
[3]: https://github.com/lsst-sqre/sciplat-lab/tree/tickets/DM-44731 "Proof of concept for two-Python sciplat-lab container"
