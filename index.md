# Peeling Apart The Pythons

```{abstract}
We have reached the point of untenability with respect to the tight coupling of the LSST DM Science Pipelines stack and the JupyterLab interface presented by the Notebook Aspect of a Phalanx environment.

This technote will explore our options to move forward, with four particular considerations in mind:

1. We need the ability to support arbitrary payloads, not solely the DM pipelines stack.  Maybe that's a vanilla Python environment, maybe it's an ML framework, maybe it's some completely different analysis pipeline.
2. How should we accept a definition of the payload?  A pip requirements.txt file? A conda environment YAML file? A supplied Docker container?  A Dockerfile?  An installation shell script? something else?
3. What else will be required for this payload?  Custom tag-categorization-and-sorting-class equivalent to the current RSPTag?  A set of environment variables that must be supplied?  A configuration YAML file?  A custom spawner menu?  Something else?
4.  It will be necessary to maintain compatibility with prior versions of the DM pipelines stack; not only must any new spawner be able to launch older images, but we must be able to create maintenance rebuilds of historical images with updated libraries and mechanisms and have them launch and perform correctly.
```

## Add content here

See the [Documenteer documentation](https://documenteer.lsst.io/technotes/index.html) for tips on how to write and configure your new technote.
