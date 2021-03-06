# ctapipe integration report 20171212

D. Neise & C. Alispach

## Preface

In the analysis call 20171128 SST1M decided to proceed integrating [digicampipe](https://github.com/calispac/digicampipe) into the nascent [ctapipe](https://github.com/cta-observatory/ctapipe).

   > We should get out as much as we can,
   and try to provide as much as we can to them.

## What can SST1M use from ctapipe?

According to ctapipe docu, *core is stable*

### ctapipe.core.Container

ctapipe.Containers are already used inside digicampipe e.g. [the R0CameraContainer](https://github.com/calispac/digicampipe/blob/master/digicampipe/io/containers.py#L77).

Still the question remained: were they used "the correct way"?

No issue opened here yet, [ctapipe#600](https://github.com/cta-observatory/ctapipe/issues/600) is related.

However, I want to comment regarding the "stable" predicate:
In [ctapipe#590](https://github.com/cta-observatory/ctapipe/issues/590) Karl Kosack said:

> Remember also that we are going to soon fully re-factor the Container class hierarchy, which means also refactoring the event source generators or FileReader classes

So the stability of this part of `ctapipe` is at the moment questionable.

### ctapipe.core.Component

`Component`s should do the work inside any `Tool` (see below).

![](component.png)

Looking at this image we expected `Components` to have a very specific *Interface*, like:
 - `Component`s are callables with `Container` as `input`
 - `Component`s **add** their results to their `input`
 - `Component`s return their `input`

This can be called a "Processor Interface" and is well known in other IACT pipelines.
But this was **not the case**. On the contrary, our questions revealed there is
opposition against having an *Interface* defined for `Component`s.

So what are Components?
`ctapipe.core.Component` can **configure itself** from a config file and
it provides **logging**.

However, in
[ctapipe#591](https://github.com/cta-observatory/ctapipe/issues/591)
we agreed that `Processors` will come at some
point, and they will be *thin wrappers* around the actual work-functions which
can be just simple functions, taking any input (from an event or other
configuration) and returning any output (into the pipeline). The `Processor`
wraps that simple function so that it can finally be used inside a more
formalized pipeline.



### ctapipe.core.Tool

A `Tool` is a command line application similar to `digicampipe.pipeline_crab.py`

The Tool takes care of parsing command line parameters and merging them with
an optional config-file in order to **configure** all `Component` playing a role
in an application, like
 - the event_source (where is the input)
 - all worker functions
 - the sink (what kind of output, where to write)

Converting an application, which already exists in digicampipe into a `ctapipe.Tool`
is in my opinion the most straight forward step to "put digicampipe into ctapipe".

# Converting an Application to a Tool what's missing?

[`pipeline_crab.py`](https://github.com/calispac/digicampipe/blob/master/pipeline_crab.py)serves as an example.

## event_source

Up to line ~100 we see a lot of configurations. This will hopefully be nicely handled by `Components`.

The first "turning wheel" of the application is the [event_source](https://github.com/calispac/digicampipe/blob/master/pipeline_crab.py#L95). So we need to get a file reader into
ctapipe.

"cocov" (Victor??) already opened a pull request long ago, trying to get the SST1M protozfits reader into ctapipe. It went "stale" despite it being nearly "done".
This PR was [revived and reworked](https://github.com/cta-observatory/ctapipe/pull/598), so the ctapipe tests pass, but is still not merged...

Subsequently and issue was opened asking how such camera specific readers should find their way into ctapipe:
[ctapipe#590](https://github.com/cta-observatory/ctapipe/issues/590)

The outcome requirements were basically this:
- have the high-level event sources inside ctapipe/io
- keep the low-level access libraries as separate packages (like pyhessio, zfits)
- don't require users to have the low-level libraries (make the import fail gracefully, so they are optional)
- in the CI system, make sure we do have all the low-level libraries so the testing can be made.

A translation of this list for SST1M's protozfits reader might be:

- [x] high-level event source in ctapipe.io, like [in #598 in ctapipe.io.zfits](https://github.com/dneise/ctapipe/blob/3a8df3561a49d8eb777cc1b9eab56fd3f9cd459d/ctapipe/io/zfits.py#L29)
- [x] external package, like [as this tar ball](http://www.isdc.unige.ch/~lyard/repo/ProtoZFitsReader-0.42.Python3.5.Linux.x86_64.tar.gz)
- [ ] Let high level event sources fail gracefully, if low level readers are missing. *To be defined*
- [ ] Continous integration (we just began, see below)
- [x] small test files.
- [x] Keep future refactoring in mind.

In order to have trust in software, this software must always bring a quick, sure and repeatable proof, that every element of the code works as it should.

For the protozfitsreader we just started to write tests, so verify it is working correctly. While writing these tests, reviews of the protozfitsreader revealed some bugs, which lead to an incompatibility with Python 3.6.

This work is not yet merged into the master, but one can have a look at the tests here: https://travis-ci.org/calispac/digicampipe/builds/315002869#L1000

Ctapipe wants to support the 2 latest version of python apparently, and digicampipe at the moment supports Python 3.5 only.

There are two reasons for this, I see at the moment:

 - The C++ protozfitsreader is difficult to build, so Etienne thankfully provided prebuild binaries. At the moment these are for Py3.5, but he can also provide them for Py3.6 if needed.

 - According to Etienne there were tests being done with Py3.6 once, and the results were not correct. I believe this is related to
 [digicampipe isse#44](https://github.com/calispac/digicampipe/issues/44),
 which will be solved by merging this [digicam PR#45](https://github.com/calispac/digicampipe/pull/45)

More tests are always better, but this seems to be a great starting position.

## Instrument Information

In addiotion to the actual events any worker function needs also access to "metadata" like, the geometry of the telescope and camera. In digicampipe this metadata
was "copied" event-wise into the event stream by the `event_source`, like here:

```python
def zfits_event_source(url, geometry):
    patch_matrix = compute_patch_matrix(geometry)
    cluster_7_matrix = compute_cluster_matrix_7(geometry)
    cluster_19_matrix = compute_cluster_matrix_19(geometry)

    for event in ZFile(url):
        data = DataContainer()
        data.r0.event_id = event.id
        data.r0.tel.adc_samples = event.adc_samples

        data.inst.geom = geometry
        data.inst.cluster_matrix_7 = cluster_7_matrix
        data.inst.cluster_matrix_19 = cluster_19_matrix
        data.inst.patch_matrix = patch_matrix
        yield data
```

One sees here how `geometry` (and static depenendcies of it) are simply "hooked"
into each event (DataContainer) and then pushed out towards the worker functions.

This way, the workers find **inside the data thing** all the information they need
to do their job.

This is how Cyril has done it in digicampipe, and I think that is very nice and useful.
However this is not how it is done in ctapipe. In fact I have no idea how they
plan to do it in ctapipe, there is a discussion about it in [ctapipe issue#600](https://github.com/cta-observatory/ctapipe/issues/600)

I think having each worker function, reach out to a database and get the meta data it needs is asking too much of a worker function.

## worker functions

As I was still working on the tests of the protozfitsreader until this morning
I have not event started to look at any worker functions.

## the sink

Also nothing is done
