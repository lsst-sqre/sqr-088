# Peeling apart the Pythons: Structure of Nublado images

```{abstract}

The Nublado component of the Rubin Science Platform provides users with a Python notebook execution environment that includes the Science Pipelines stack.
At present, the Python installation provided by the Science Pipelines Conda environment is used to run both JupyterLab and for the notebook kernel.
This approach is relatively simple, but it tightly coupled the service with the notebook kernel in a way that presents serious backwards compatibility problems and has been a source of fragility.
This tech note describes plans for a new base (lightweight) Docker image for Nublado pods and a strategy for separating the Python environment used for JupyterLab from the Python environment used for the notebook execution kernel.

```

## This Is An Archival Document

We merged the 2-python work to `main` in April 2025.
RSP sciplat-lab images `w_2024_14` and later are all 2-python versions.
This document captures the motivation and design choices made to get to the current design.

## Motivation

At the time of writing (October 2024), Nublado lab containers are built on top of science pipelines containers produced by the pipelines build system (Jenkins).
These containers are named `docker.io/lsstsqre/centos:7-stack-lsst_distrib-<tag>`, using the tag conventions described in {sqr}`059`.
Currently, they use the version of Python included in the Conda environment provided by the stack for both JupyterLab and for the notebook execution kernel.

This was a convenient approach in early versions of the service, but presents a number of issues going forward:

1. We want to support execution kernels other than the Science Pipelines stack.
   Some possible examples are a lightweight Python-only container, a machine-learning-oriented container customized to the needs of a particular science collaboration, or a kernel maintained by a non-Rubin mission who adopted our platform.

2. During operations, we have a complex backwards compatibility story.
   We are required to make available kernels based on up to three-year-old versions of the Rubin Science Pipelines.
   We do not, however, want to be locked into deploying three-year-old versions of the service.

3. Sharing a Python execution environment between the user's kernel and the JupyterLab server means that the user can customize their environment in ways that break the JupyterLab server, thus locking themselves out of the service.
   This has already happened to one user during the Preview phase.
   Although we provide them with the capability to reset their environment to resolve this, the reset creates unnecessary user support load and may lose other customizations.

4. The Science Pipelines Conda environment adopts new versions of Python relatively slowly, but newer versions of Python are often helpful for the JupyterLab service layer.
   For example, we recently wanted to use {py:meth}`pathlib.Path.walk` in the JupyterLab startup code, which requires Python 3.12, but the Science Pipelines Conda environment was still using Python 3.11.
   It is likely that the Science Pipelines Conda environment will become even more conservative once we enter operations.

We have been aware of these issues for some time.
Recent developments such as the recent switch of the Science Pipelines container from CentOS to AlmaLinux and the approach of Data Preview 1 presented a timely opportunity to replace this architecture with a new one that is better matched to our future needs.

## Nomenclature

Within the context of this tech note, we refer to two Python distributions: the "JupyterLab Python" responsible for providing the UI and browser-specific components to the Lab user, and the "payload Python", packaged as a Jupyter kernel, which provides the Python environment that executes the user's notebook and can be used by the user from an interactive shell for many purposes, including installing their own packages for use in their notebooks.
There will likely be non-core-Python components that must be common to both (see below), but in general the contents of these different Python environments can and should be largely disjoint.

For the context of this technote, the Nublado payload is explicitly a Python environment.
Non-Python payloads are a future possibility, but are outside the scope of this tech note as we have no immediate plans to support them.

## Proposed architecture

The proposed new architecture for Nublado lab containers consists of three layers:

1. The base layer is `docker.io/library/python`, the official Python container built on top of a Debian stable container.
   This tracks the latest stable Python release and will be updated for new Python releases on the same cadence as other Rubin Science Platform services (in other words, fairly aggressively, around the time of the first `.1` patch release).

2. On top of that base we install JupyterLab and its supporting packages, including any customizations to JupyterLab itself outside of the kernel payload.
   This uses the Python provided by the base layer.
   It will be published as `ghcr.io/lsst-sqre/nublado-jupyterlab-base` and serve as the base layer for all lab containers.

3. Finally, the payload for the execution kernel is layered on top of that base container.
   For the default container for the Rubin Science Platform, published as `sciplat-lab`, this means running [lsstinstall][1] on top of the `nublado-jupyterlab-base` container created in the previous step, and then performing some additional setup and configuration.
   This installs the `rubin-env` (and `rubin-env-rsp`) Conda environment including, among other things, a separate version of Python that has been tested with the Science Pipelines stack.

The resulting container therefore has, in the typical case, two separate installations of Python.
One, provided by the official Python container, is used to run JupyterLab and cannot be customized by the user.
The other, provided by the `rubin-env` Conda environment, is used to run the execution kernel and as the user's Python environment for interactive shell use, but is not used to run the JupyterLab process itself.

When adding other payloads, we will use the same approach: a Docker build for the payload lab containers, layered on top of `nublado-jupyterlab-base`.

This approach has been tested in the [Nublado repository][2] and the resulting images appear to work correctly.

### Changed behavior

There are a few note-worthy differences between this approach and the previous containers that we believe are acceptable:

- The `rubin-env` Conda environment is freshly installed for each new container build, rather than being inherited from a base Science Pipelines image.
  It will therefore pick up newer versions of package dependencies, since the Science Pipelines Conda environment does not pin package versions.
  This creates some additional risk of surprise bugs or incompatibilities.

- The container image is almost 15GB (uncompressed), rather than 10GB with a shared Python installation.
  This is prior to any attempt at optimization, such as removal of packages that are no longer needed in the payload, so this discrepancy may get smaller with optimization.
  A lot of the space usage comes from supporting PDF export of notebooks, which introduces a surprising amount of complexity and large software package installations into the JupyterLab environment.

- The Python environment running JupyterLab shows up as an option if the user goes to the UI to change kernels, even though we would prefer the user not use that environment for execution kernels.

### Potential problems

JupyterLab extensions are usually supplied as a combination of frontend UI components and a Python client component.
In some cases, the latter may need to be available in the payload so that the executed cell can talk to the JupyterLab UI extension.

This creates the possibility that the code in the payload will diverge from the code in the frontend in a way that causes the former to no longer be able to talk to the latter, particularly when running older payloads
We do not currently have a solution to this, but since we have not problems with the extensions we have been using so far, we have grounds to hope that such an incompatibility won't occur.

### Transition to the new architecture

For the Rubin Science Platform use case, we need to be able to continue to spawn older images.
This requires some care, since the new design moves some files to different locations in the Docker image, but the Nublado controller expects to use the same entry points for all lab images.

The approach we've taken is to create symbolic links or copies of various files so that they are present both at the old "everything under `/opt/lsst/software`" location and the newer "JupyterLab machinery lives in `/usr/local/share/jupyterlab` but the Science Pipelines stack is still in `/opt/lsst/software/stack`" locations.
Minor modifications have been made to the spawner to allow the locations of critical files, such as the command run inside a user container to start the lab, to be parameterized in the Nublado controller configuration.

This configuration cannot be set per image, so we will need to create symlinks or copies in the old paths for some time even after switching to the new image build system.
For now, the Nublado controller continues to use the old paths.
Once all of the old images are so old that we no longer need to be able to use them, the Nublado controller configuration can be switched to the new paths and the backward-compatibility symlinks and copies can be removed.

A similar backward-compatibility layer can also be used to support old payloads once the Nublado controller has switched to the new paths.
When rebuilding an image for an old payload on top of a current JupyterLab container, a backward-compatibility link and copy step can make the old paths available in the new location expected by the Nublado controller.

## Source organization

As part of this work, we also intend to reorganize the source to the various Nublado components to make it easier to find and maintain as a unified product.

The [Nublado repository](https://github.com/lsst-sqre/nublado) will hold the general infrastructure for the Nublado Phalanx service.
This is the code that is generic to any instantiation of the service.
It includes the JupyterHub and JupyterLab infrastructure, including any JupyterLab extensions that are common across payloads.
This repository is maintained as a verical monorepo (see {sqr}`075`).

Specifically, the Nublado repository will contain:

1. The Docker build and plugins for the Nubaldo version of JupyterHub.
2. The Nublado lab controller (see {sqr}`066`).
3. A Python library for making requests to the Nubaldo version of JupyterHub and JupyterLab.
4. Any JupyterLab extensions included in the Nublado version of JupyterLab (but see the note below).
5. Any code used in the JupyterLab base container to customize the JupyterLab start-up process that is independent of any payload.
6. The Docker build for the JupyterLab base container, including those extensions but not including any payload or Docker builds for payload containers.
7. The file server used to implement the WebDAV component of Nubaldo (see {sqr}`078`).
8. Other utilities or supporting containers for the Nublado JupyterLab or JupyterHub, independent of any payload.
   Examples include the `inithome` container to create user home directories on demand or a utility to delete old images from a Docker image repository.
9. Documentation for Nublado as a whole.

For point 4, note that the GitHub build tools for JupyterLab extensions currently require those extensions be at the top level of a repository.
We will therefore keep them in a separate repository until this limitation of the build tools has been fixed.

Anything related to a specific payload will not be kept in the Nublado repository.
Specifically, the pieces required for the Notebook Aspect of the Rubin Science Platform, as independent from a generic Nublado installation, will be maintained in a separate repository.
This ensures that the separation of concerns between Nublado and a specific payload will be exercised internally at Rubin.
This includes:

1. Any programs or libraries installed as part of the payload rather than inside JupyterLab itself.
   As an exception, generic clients for JupyterLab extensions maintained as part of Nublado can be maintained the Nublado repository along with the extension.
2. Installation and Docker builds for any specific payload container.
3. Any additional start-up code for the payload kernel, as opposed to the JupyterLab service.
4. Any additional software intalled for the benefit of the shell environment of the payload container, unless it's part of the generic Nublado infrastructure.

## Testing experience

This approach has also been used with our (early July, 2024) proof-of-concept implementation of a pathfinder SPHEREx Nublado deployment, done as part of a technology demonstration.
We successfully deployed a payload container (from [lsst-sqre/spherex-lab](https://github.com/lsst-sqre/spherex-lab)) running the public (that is, non-export-controlled) parts of the SPHEREx analysis environment and using this two-Python model.
The SPHEREx environment was installed via a Conda environment.

## Future work

- Implement a new Docker image tag policy class that can be used instead of `RSPTag` when the payload provider uses a different versioning convention.
  We will probably only want to support semver and calver.
  Until this is done, other payloads that do not want to use the full {sqr}`059` version semantics should use tags that match the syntax of release version numbers as documented there.

- Support selecting between different payloads in the spawner form.
  Currently, Nublado assumes that all images will have the same payload and use the same configuration for paths, environment variables, and so forth.
  Supporting multiple payloads at the same time within one Rubin Science Platform deployment will require rethinking the spawner form UI and the Nublado controller configuration.

## Appendix

The notes capture some of the discussion that led to the architecture described above and is unlikely to be of interest to the general reader.

### Analysis

To support arbitrary Nublado payloads, we have to answer a few questions:

1. What base container will we use for the Docker image?

2. How should we accept a definition of the payload?
   A pip `requirements.txt` file?
   A Conda environment YAML file?
   A supplied Docker container?
   A `Dockerfile`?
   An installation shell script?
   Something else?
   What style of definition will be the most convenient for publishers of alternative payloads?

3. What else will be required for this payload?
   A tag categorizing and sorting class, similar to the existing `RSPTag`?
   A set of environment variables that must be supplied?
   A configuration YAML file?
   A custom spawner menu?
   Something else?

4. How do we maintain compatibility with prior versions of the Science Pipelines stack?
   We must be able to create maintenance rebuilds of historical images with updated libraries and integration APIs and have them launch and perform correctly.

#### Choice of base container

There are two fundamental ways of providing different payload and Jupyterlab Python environments.

1. Install the Jupyterlab machinery on top of a supplied container.
   This is the way we're currently doing it, and it can be adopted for a two-Python configuration.
   Currently, we install everything into the `rubin-env` Conda environment, but we could instead create a new Conda environment for the JupyterLab runtime, with whatever version of Python is desirable, and install whatever JupyterLab needs into it.
   This creates a larger environment than the current status quo (as will all of these approaches) but is conceptually straightforward.

   A drawback of this method is that we have to tailor our installation approach to fit the base container.
   We will want system-packaged command-line tools, for example, so that people have their choice of editors and so forth.
   If we use the system machinery of the upstream container, this will require maintaining different build processes for each possible choice of system machinery in upstream containers: RPM-based systems, dpkg-based systems, Alpine systems using `apk`, and so forth.

2. Install a standard container and then install the payload on top of that container instead.
   For instance, `docker.io/library/python:3.12` is the official Python container, built on top of a Debian stable container.
   This was the approach we chose.

   This will work if the payload isn't tightly coupled to a specific Linux system flavor by, for example, using RPM internally.
   Open source packages are usually portable across flavors of Linux, and even commercial ones will usually work on either RHEL or Ubuntu.
   The one exception is Alpine or other musl-based Linux containers, which should be avoided for this approach.
   This approach also creates a larger container, but keeps the system layer fairly simple compared to using an upstream one.

The first method is closer to current practice.
However, if a payload is not already packaged as a container, the second approach is required anyway.
The most worrying part of the first approach is that accomodating new payloads is hard, since it may require substantial new tooling for the new variation of Linux packaging.

The second method hopefully will mean that we can use a single base container and build payload containers on top of it additively.
New payloads would only require changing the build stages that implement the installation and runtime launch framework of that payload.

#### Installing payloads

Unfortunately, there is no universal method of installing payloads, but in practice they should fall into one of the following five categories:

1. The payload is pip-installable: there's a `requirements.txt` file, the installer points `pip` at it, and the payload is installed.

2. The payload is Conda-installable: there's an environment YAML file, the installer points `conda` at it, and the payload is installed.

3. The payload is delivered as an installation shell script.
   The Science Pipelines stack can be installed this way with [lsstinstall][1].
   This may be the most common way for thorny environments to be delivered.
   For anything of sufficent size that has made the choice to not use a standard build/installation system, such a script will probably already exist.
   This approach will only work if the installation script does not assume a particular underlying distribution type such as dpkg or RPM, and if it limits its actions to a specific directory in the filesystem or otherwise avoids interfering with the JupyterLab environment in the base container.

4. The payload is supplied as a Dockerfile that constructs a container image.
   The appropriate pieces of the Dockerfile can be moved into the Nublado-launchable image construction as `COPY` and `RUN` statements, or as a shell script which does the same thing.
   This has the same disadvantages as the above: it must not assume a particular method of package installation, and must limit the payload-specific installation to a specified directory.

5. The payload is only available as a Docker image, without build instructions.
   This is even more alarming and certainly suggests the first method would be a better idea, but if the payload is similarly limited to a directory (for instance, the Science Pipelines stack lives almost entirely under `/opt/lsst/software/stack`), a multi-stage Docker build could transplant that directory hierarchy into the target container.
   In practice this is likely to be difficult to support.

Any of the first four approaches can in theory be used in a Docker image build layered on top of a generic JupyterLab container, as long as the payload does not make too strong of assumptions about the operating system used by the base container.
That makes replacement of the payload straightforward since it's isolated from the rest of the container build.

#### Payload configuration

Configuration of the payload environment has at least three components.

Setting up the Jupyter kernel is the most straightforward.
If the payload can be represented in a Python virtual environment, `python -m ipykernel install` is all that needs to be done.
If the payload is more complicated (the Science Pipelines stack, for example, is), the kernel can be a shell script that does necessary setup and execs a Python process as its last action.
If this is necessary, it will be the responsibility of the payload container build to supply this script.

The next component is setting up a terminal environment.
One way to do this is to set a sentinel environment variable at JupyterLab launch.
Then, when the user requests a shell, test that variable in the user's `.bashrc` or equivalent.
If it is set, indicating the user is running inside JupyterLab, perform the necessary setup, such as environment activation or running an appropriate setup script, before yielding control to the user for interactive shell use.

The third component is the most difficult: the user interface for selecting images.

##### Image selection

In Nublado, we use a JupyterHub spawner form for this purpose.
That form allows the user to select from a list of prepulled images or uncached images, a selector to choose container size, and two checkboxes useful for debugging and unwedging inoperable environments.
This general pattern is probably appropriate for most payload containers, but the details may differ.
The most likely place where they will differ is image selection.

Currently, image selection leans heavily on the Science Platform versioning rules documented in {sqr}`059`.
This divides the version space into daily, weekly, pre-release, and release images and maps tags to a useful human-readable description of the image.
All of this is done automatically based on a Docker tag convention.
The spawner menu updates dynamically based on the list of available images, without requiring any human intervention.

This same tag convention is used to decide which images to prepull.
Given the expected size of typical Nublado lab images, it is vital to prepull commonly-used images.
Spawning an uncached image is likely to take several minutes, which becomes an unpleasant experience for the user.
The image selection menu in the spawner form is organized according to the prepull rules: selection of images that are prepulled is encouraged, and uncached images are relegated to a drop-down list of additional available images with a warning that spawning a lab with one of those images is likely to be slow.

It is not clear whether this specific use of tags, prepull configuration, and image types used for Science Pipelines payloads will match what other payloads will want.
The form is somewhat flexible, in that dailies and weeklies can be disabled in favor of only showing releases, but the rules for constructing the Docker image tags are very strict and are likely not to match the conventions of other projects.

We may therefore need to add some new method of categorizing and sorting images.
Currently, image classification is done by the `RSPTag` class within the Nublado lab controller implementation, which understands the Science Pipelines conventions documented in {sqr}`059`.
We may need a second possible implementation of tag versioning with simpler sorting and prepulling properties.
One possible option would be a tag class that expects [semver][3] or [calver][4] versions and prepulls the most recent N full (i.e., not alpha, beta, or pre-release) releases.

[1]: https://ls.st/lsstinstall "lsstinstall"
[2]: https://github.com/lsst-sqre/nublado/tree/main/sciplat-lab "Proof of concept for two-Python sciplat-lab container"
[3]: https://semver.org/ "semver"
[4]: https://calver.org/ "calver"
