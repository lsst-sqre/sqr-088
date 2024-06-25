# Peeling Apart The Pythons

```{abstract}
We have reached the point of untenability with respect to the tight coupling of the LSST DM Science Pipelines stack and the JupyterLab interface presented by the Notebook Aspect of a Phalanx environment.

This technote will explore our options to move forward, with four particular considerations in mind:

1. We need the ability to support arbitrary payloads, not solely the DM pipelines stack.  Maybe that's a vanilla Python environment, maybe it's an ML framework, maybe it's some completely different analysis pipeline.
2. How should we accept a definition of the payload?  A pip requirements.txt file? A conda environment YAML file? A supplied Docker container?  A Dockerfile?  An installation shell script? something else?
3. What else will be required for this payload?  Custom tag-categorization-and-sorting-class equivalent to the current RSPTag?  A set of environment variables that must be supplied?  A configuration YAML file?  A custom spawner menu?  Something else?
4.  It will be necessary to maintain compatibility with prior versions of the DM pipelines stack; not only must any new spawner be able to launch older images, but we must be able to create maintenance rebuilds of historical images with updated libraries and mechanisms and have them launch and perform correctly.
```

## Current State of the World (late June, 2024)

The Rubin Science Platform lab containers are built every night by GitHub Actions jobs.
They consume a container produced by Jenkins, which is named `docker.io/lsstsqre/centos:7-stack-lsst_distrib-<tag>` where the tag follows conventions described in [sqr-059] [1].

### Pros of this approach

This approach brings with it certain benefits.

1. First and foremost is that it produces the smallest plausible container images.  "Smallest" is of course relative: the DM pipelines stack itself is very large, and so even this image is nearly 10GB (uncompressed; Docker Hub suggests that "a bit more than 3" is an accurate compression ratio for image transmission purposes).
2. The DM Pipelines Stack version present is always exactly the official one for that date.  This is complicated by the fact that the conda-installed parts of rubin-env are fixed for the lifetime of a release, but the EUPS-installed pieces are indeed rebuilt nightly.  This state of affairs is a consequence of:
3. The production of the "stack container" is itself an extremely complicated layer-cake.  Effectively, the base conda environment for a given release version of the `rubin-env` environment is created, and itself containerized, when a new release version is cut, and then reused for subsequent builds.  This means that the stack you get differs from the stack you would have if you just ran [lsstinstall](https://ls.st/lsstinstall), because the second will get current-at-runtime versions of `rubin-env` packages and their dependencies.  Building the sciplat-lab image on top of the nightly container build sidesteps all those complexities.

### Cons of this approach

As you might expect, this also has several drawbacks.

1. The most important is that the Python environment of the Jupyterlab machinery that furnishes the notebook UI to the user must be a superset of the DM pipelines stack environment.  This is problematic for many reasons.  A good example at the time of writing is that it would be convenient at Lab startup to be able to use the Python `pathlib.Path.walk()` method; however, this feature is not available in Python prior to 3.12, and the DM Stack is at version 3.11 due to some of its dependencies not being able to function with 3.12 or later.  This general class of problem percolates down to other packages as well, including iPython and Jupyter components.  It would be extremely convenient to decouple the Lab launching and presentation mechanisms from the internals of the DM pipelines stack.  It becomes much more of a nightmare than necessary to ensure that the Lab machinery and the stack play well together.
2. It is by definition impossible to add new payloads.  If the only Python in the image is that of the DM stack then there is no way to add payloads that are not subsets of the DM-stack-plus-Jupyterlab-Machinery conglomerate.
3. With only one Python and only one Conda environment, it becomes distressingly easy for user-installed software to keep the Lab from starting, because the user-installed software, which worked with the image current when written, becomes incompatible with later Lab evolution.  If the Jupyterlab process cannot fully start, the user has no way to rectify the situation.  This is precisely why the Rubin spawner has the "Relocate User Environment" checkbox.
4. Currently, the DM Stack container is built atop CentOS 7.  CentOS 7 was released in 2014, ceased receiving full updates in August 2020, and will receive (by the time you read this, will have received) its last maintenance update at the end of June 2024.  CentOS itself has been discontinued.  The next release of the rubin-env Conda environment, 9.0.0, will be based on AlmaLinux 9.  Since SQuaRE will have to do significant retooling to accomodate that change, doing the split now may be easier than delaying.

### Summary

In sort, we (SQuaRE) have decided that the cons now outweigh the pros, and we plan to develop a way to allow the Jupyterlab python environment to exist completely independently of the DM stack python environment.  The rest of this technote will address the four bullet points in the abstract, with discussion of various approaches we could take to each of them.

## Arbitrary Payloads

Not all containers of interest to Rubin Observatory notebook users require the full majesty of the DM pipelines stack.  For example, it is completely irrelevant to the use at Telescope and Site environments of Jupyterlab as an instrument control interface.  Most of the notebooks people would like to run via Noteburst require little, if any, of the DM stack.

People have asked for capabilities that would be tricky to provide within the framework of the DM stack; for instance, Tensorflow or other machine learning frameworks.  There have been occasional requests to support R or FORTRAN, both of which seem like support nightmares to this author.

The primary use case seems to be "I'd lke to run an astronomical pipeline (probably, but not always, analytic--cf. the instrument control case) that is something other than the DM pipelines stack."  If we have a good way to split apart the Lab machinery from the pipeline presented as one of the Lab kernels, this immediately becomes feasible.

[1]: https://sqr-059.lsst.io/ "RSP Notebook container tag conventions"
