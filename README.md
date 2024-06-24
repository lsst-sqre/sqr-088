[![Website](https://img.shields.io/badge/sqr--088-lsst.io-brightgreen.svg)](https://sqr-088.lsst.io)
[![CI](https://github.com/lsst-sqre/sqr-088/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-sqre/sqr-088/actions/workflows/ci.yaml)

# Peeling Apart The Pythons

## SQR-088

We have reached the point of untenability with respect to the tight coupling of the LSST DM Science Pipelines stack and the JupyterLab interface presented by the Notebook Aspect of a Phalanx environment.

This technote will explore our options to move forward, with four particular considerations in mind:

1. We need the ability to support arbitrary payloads, not solely the DM pipelines stack.  Maybe that's a vanilla Python environment, maybe it's an ML framework, maybe it's some completely different analysis pipeline.
2. How should we accept a definition of the payload?  A pip requirements.txt file? A conda environment YAML file? A supplied Docker container?  A Dockerfile?  An installation shell script? something else?
3. What else will be required for this payload?  Custom tag-categorization-and-sorting-class equivalent to the current RSPTag?  A set of environment variables that must be supplied?  A configuration YAML file?  A custom spawner menu?  Something else?
4.  It will be necessary to maintain compatibility with prior versions of the DM pipelines stack; not only must any new spawner be able to launch older images, but we must be able to create maintenance rebuilds of historical images with updated libraries and mechanisms and have them launch and perform correctly.

**Links:**

- Publication URL: https://sqr-088.lsst.io
- Alternative editions: https://sqr-088.lsst.io/v
- GitHub repository: https://github.com/lsst-sqre/sqr-088
- Build system: https://github.com/lsst-sqre/sqr-088/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

.. code-block:: bash

```sh
git clone https://github.com/lsst-sqre/sqr-088
cd sqr-088
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://sqr-088.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://sqr-088.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
