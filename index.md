# Peeling Apart The Pythons

```{abstract}

Up to the time of writing, we have been using the LSST Science Pipelines Stack (hereafter "stack") as the python underlying nublado, the service implementing the Rubin Science Platform Notebook Aspect.
At the time, this was an easy and efficient way to guarantee that a nublado user had a python environment that was consistent with the stack environment and to leverage work already done by the pipelines build system.

Now that we are operating a long-running service, we are encountering the limitations of this approach.
This technote outlines our plan to change nublado to use a python version for the service that is distinct from the in-stack python available to the user.

```

## Motivation

At the time of writing (June 2024), nublado lab containers are built every night by GitHub Actions jobs.
They consume a stack container produced by Jenkins, which is named `docker.io/lsstsqre/centos:7-stack-lsst_distrib-<tag>` where the tag follows conventions described in [sqr-059][1].

So far we have been using the python included inside the stack containers as an easy and efficient way to guarantee that a nublado user had a python environment that was consistent with the stack environment and to leverage work already done by the pipelines build system.

The following drivers are causing us to move on from the current approach.

1. We want nublado to be able to support arbitrary payloads, i.e. different types of containers besides stack containers. Examples would be: a lightweight python only container, a "special edition" container supplied by a collaborating group that is specific to a particular science user case (eg a ML-oriented container maintained by an initiative), or another mission's container.

2. In operations we have a complex backwards compatibility story that includes the requirement to provide a nublado container with a two-year-old version of the pipelines in the current service. This means we need to be able to keep the service current while the stack is running an older version of python.

3. We have seen operational fragility where a user can modify their environment in a way that prevents JupyterLab from starting.

4. We have occasionally wanted to move to a newer version of python than what is supported in the stack (for example we recently wanted to move to 3.12 in order to use the `pathlib.Path.walk()` while the stack was still on Python 3.11). It is a possibility during survey operations, especially in the latter half, pipelines will become more conservative about python upgrades at a time when Science Platform infrastructure is still aggressively developed.

We have been tracking these risks for a while, and the upcoming switch of the pipelines build from CentOs to AlmaLinux (which will already require some retooling on the services side) presents us with a convenient opportunity to do this change.

The remainder of this technote addresses some implementation considerations that support our motivations.

## Nomenclature

Within the context of this technote, we are going to assume two Pythons: the "Jupyterlab Python", responsible for providing the UI and browser-specific components to the Lab user, and the "Payload Python", packaged as a Jupyter kernel, which provides the Python environment that the user wants to work within.
There will likely be non-core-Python components that must be common to both (see below), but in general the contents of these different Python environments will be largely disjoint.

We are assuming that the payload is at its core a Python environment.
It is not impossible that we might someday consider a non-python payload, but that is outside the scope of this technote, and in any event, any Nublado-spawned container image will need at least one Python environment in order for the Jupyterlab UI to function.

## Arbitrary Payloads

In considering support for arbitrary nublado payloads (i.e. different flavors of container), the following questions arise:


1. How should we accept a definition of the payload?  A pip requirements.txt file? A conda environment YAML file? A supplied Docker container?  A Dockerfile?  An installation shell script? something else?

2. What else will be required for this payload?  Custom tag-categorization-and-sorting-class equivalent to the current RSPTag?  A set of environment variables that must be supplied?  A configuration YAML file?  A custom spawner menu?  Something else?

3.  It will be necessary to maintain compatibility with prior versions of the DM pipelines stack; not only must any new spawner be able to launch older images, but we must be able to create maintenance rebuilds of historical images with updated libraries and mechanisms and have them launch and perform correctly.

4. Methods for specifying the payload need to consider likely scenarios of what publishers of alternative payloads would find convenient to supply as manifests.


## Payload Specification

Obviously the payload must be specified in such a way that it can be automatically built by our build chain.
Ultimately it will be installed from a Dockerfile; whether that's a single `RUN` or `COPY` command, or a `RUN` command kicking off a complex script, or `FROM some-other-container AS upstream` is a matter of debate.

There are two fundamental ways of providing different payload and Jupyterlab python environments.

1. Install the Jupyterlab machinery atop a supplied container.  This is the way we're currently doing it.  There's no particular reason that we need a single-stack, single python solution, even starting from `centos:7-stack...`.  Although we currently install everything into `rubin-env` within the container, we could simply create a new `conda` environment for the Jupyterlab runtime, and `conda` or `pip` install what we needed into it, including, if we like, a newer version of Python.  This creates a larger environment than the current status quo (as will all of these approaches) but is conceptually straightforward.  A drawback of this method is that we have to tailor our installation approaches to the base container.  We will certainly want system-packaged command-line tools, for example, so that people have their choice of editors and so forth.  If we are forced to use the system machinery of the upstream container, this will mean largely rewriting the build process to accomodate (at least) RPM-based systems, dpkg-based systems, and Alpine systems using apk.  In addition, Canonical is now pushing `snap` which is technically vendor-agnostic, although since Canonical controls the snap store, it's really just another attempt at vendor lock-in.
2. Install a standard container and then install the payload on top of that container instead.  For instance, `docker.io/library/python:3.12` is the official Python container, and it's built on top of Debian.  This will work if the payload isn't terribly tightly coupled to a given Linux packaging system flavor (if, for instance, the DM pipelines stack made use of RPM internally, it would be quite hard to make work on a Debian-derived system).  This may be a nonissue in practice: Open Source packages are usually portable across flavors of Linux, and even commercial ones will usually work on either RHEL or Ubuntu.  This approach also creates a larger container, but keeps the system layer fairly simple compared to using an upstream one.

The first method is closer to a known quantity.
However, if a payload is not already packaged as a container, then it ends up being a variation on the second method.
The most worrying thing about this approach is that it makes accomodating new payloads harder, because significant portions of the non-payload pieces need to be rewritten for each payload.

The second method means, at least theoretically, that we could keep one lab-building framework, and drop in new payloads by simply updating the build stage or stages that implement the installation (and runtime launch framework) of that payload.
The problem there is that there is no universal method of doing so.
However, it seems difficult to imagine that one of the following five techniques is not applicable.

1. The payload is pip-installable: there's a `requirements.txt` file, the installer points `pip` at it, and the payload is installed.
2. The payload is conda-installable: there's an environment `YAML` file, the installer points `conda` at it, and the payload is installed.
3. The payload is delivered as an installation script.  The DM pipelines stack can also be installed this way with [lsstinstall][2] and this may be the most common way for thorny environments to be delivered.  For anything of sufficent size, that has made the (bad) choice to not use a standard build/installation system, this will probably already exist.  See, for instance, the huge proliferation in products that expect you to `curl https://some-url | sudo /bin/bash`.  Less grotesquely, maybe it's something you can install with `./configure; make; make install`.  In any event this approach will only work if the installation script does not assume a particular underlying distribution type (e.g. dpkg vs. RPM) and if it contains its action within a specific subtree on the filesystem.
4. The payload is supplied as a Dockerfile that constructs a container image.  The appropriate pieces of the Dockerfile can be moved into the nublado-launchable image construction (either as `COPY` and `RUN` statements, or as a shell script which does the same thing), and the payload is installed.  This has the same disadvantages as the above: it must not assume a particular method of package installation, and must keep the huge majority of the payload-specific installation within a specified subtree.
5. The payload is only available as a Docker image, without build instructions.  This is even more alarming and certainly suggests the first method would be a better idea, but allowing the assumption that the payload is similarly limited to a filesystem subtree (for instance, the DM pipelines stack lives almost entirely under `/opt/lsst/software/stack`), a multi-stage Docker build could in principle transplant that directory hierarchy into the target container.  In practice this is likely to be quite difficult.

My preference is the second method, using the first or second variants if possible.
That lets us keep almost all of the image-building machinery the same between payloads, and by encapsulating the payload installation in a script and image layer of its own, replacement of the payload becomes quite straightforward.
If the payload is in one of the easy formats, this script might literally be nothing more than `pip install -r requirements.txt`.
However, specific image-building considerations mean that it may not be a single script and stage: system-level packages may need to be installed as root, while we would prefer that both the installation and runtime operation of the payload happen as a less-privileged user.

That sets up the third point: what will be necessary in order to turn the installed payload into a Jupyterlab-runnable kernel and a terminal environment with the payload contents configured and available?

## Payload configuration

Configuration of the payload environment has at least three components.

Setting up the Jupyter kernel is the most straightforward.
If the payload can be represented in a Python virtual environment, `python -m ipykernel install` is all that needs to be done.
If the payload is more complicated (as the DM pipelines stack is), then it turns out that the kernel can be a shell script that does necessary setup and execs a python process as its last action.  If this is necessary it will be the payload provider's responsibility to supply this.

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

## Compatibility

Of course, for the RSP use case, if we switch to a new version of image construction, we must allow the spawner to continue to spawn older images.

There is a proof-of-concept implementation of such a thing in [the tickets/DM-44731 branch of sciplat-lab][3] which demonstrates this: effectively, we've created symbolic links or copies of various files such that they are present both at the old "everything under `/opt/lsst/software`" location and the newer "Jupyterlab machinery lives in `/usr/local/share/jupyterlab` but the DM pipelines stack is still in `/opt/lsst/software/stack`" locations.  The actual functionality to provide the compatibility layer is found in [the install-compat script run during `docker build`](https://github.com/lsst-sqre/sciplat-lab/blob/tickets/DM-44731/scripts/install-compat).

Minor modifications have been made to the spawner to allow the locations of certain items, like "what command is run inside a user container to start the machinery" to be parameterized in the spawner config, so in either the scenario where all the old-style images are so old we no longer care about supporting them, or the scenario where we're running a completely different payload and therefore have no reason to use the old paths, the spawner can work with only changes to the Phalanx configuration rather than changes to its code.

This also addresses the maintenance issue.
If we need to rebuild a current version of the container with an antiquated payload, we have the compatibility layer to assist.  The layout of all the non-payload components is free to change, as long as we have a way to link or copy the (fairly few) pieces back into their prior location.

## Proposed Implementation

We think something like [the proof-of-concept implementation][3] is the right direction.  This one is based on the current official Python 3.12 container, which is Debian-based.  It does not produce exactly the same stack container that its counterpart in the lsst-distrib-as-base-container model would, in that the rubin-env Conda environment is freshly installed each time the container runs [lsstinstall][2] and therefore will get newer versions of dependent packages (the decision not to hard-pin versions was taken by the Science Pipelines developers; the author disagrees with their choice, but above my pay grade, I suppose).

There is a potential problem--I am not sure of its gravity--in that Jupyterlab extensions are usually supplied as a frontend UI set of components, and a Python backend server component.
It is not clear to me that the two of these should always be in the Jupyterlab environment rather than one in the Jupyterlab Python and one in the Payload Python, or whether they should be installed in both, and if the answer is "both", then how does the frontend know how to talk to the correct backend?  
Will the backend, potentially running on an entirely different python version, necessarily even be able to communicate with the frontend?
It is entirely possible I am overthinking this; it appears that installing all the extension-ish stuff and its dependencies into the Jupyterlab Python is working for us now.

Behaviorally this container appears functionally identical to the single-stack container in terms of what payload code it will run.
There is a change visible in the UI, in that I do not know how to hide the Python environment running Jupyterlab from the generated image.

The other major drawback is that the container image is almost 15GB rather than 10GB.
No attempt at optimization of packages that are no longer needed in the stack environment has been made, so there's probably some wiggle room.  And we could always give up on PDF export of notebooks, which introduces a surprising amount of complexity and storage space requirements into the UI Python environment.
It can probably be reduced to "not grossly larger than the single-stack container" with some work.

This approach has also been used with our (early July, 2024) proof-of-concept implementation of a SPHEREx nublado deployment, done as part of a technology demonstration.
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

[1]: https://sqr-059.lsst.io/ "RSP Notebook container tag conventions"
[2]: https://ls.st/lsstinstall "lsstinstall"
[3]: https://github.com/lsst-sqre/sciplat-lab/tree/tickets/DM-44731 "Proof of concept for two-Python sciplat-lab container"
