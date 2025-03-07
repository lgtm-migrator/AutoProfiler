<p align="center"><a href="#"><img width=60% alt="" src="https://raw.githubusercontent.com/cmudig/AutoProfiler/main/.github/screenshots/Autoprofiler_social_tag.png"></a></p>

<p align="center">
    <a href="https://pypi.org/project/digautoprofiler/">
        <img alt="PyPi" src="https://img.shields.io/pypi/v/digautoprofiler.svg" align="center">
    </a>
     <a href="https://mybinder.org/v2/gh/cmudig/AutoProfiler/HEAD?labpath=examples%2FSF%20Housing%20Demo.ipynb">
        <img alt="Binder" src="https://mybinder.org/badge_logo.svg" align="center">
    </a>
     <a href="http://dig.cmu.edu/AutoProfiler">
        <img alt="Lite" src="https://gist.githubusercontent.com/willeppy/35cdc20a3fc26e393ce76f1df35bcdfc/raw/a7fca1d0a2d62c2b49f60c0217dffbd0fe404471/lite-badge-launch-small.svg" align="center">
    </a>
</p>

Profile your [Pandas](https://pandas.pydata.org) Dataframes! Autoprofiler will automatically visualize your Pandas dataframes after every execution, no extra code necessary.

Autoprofiler allows you to spend less time specifying charts and more time interacting with your data by automatically showing you profiling information like:

-   Distribution of each column
-   Sample values
-   Summary statistics


## Updates profiles as your data updates
![screenshot of Autoprofiler](https://raw.githubusercontent.com/cmudig/AutoProfiler/main/.github/screenshots/profiler_sc.png)

Autoprofiler reads your current Jupyter notebook and produces profiles for the Pandas Dataframes in your memory as they change.

https://user-images.githubusercontent.com/13400543/199877605-ba50f9c8-87e5-46c9-8207-1c6496bb3b18.mov


## Install

To instally locally use pip and then open a jupyter notebook and the extension will be running.
```bash
pip install -U digautoprofiler
```

## Try it out

To try out Autoprofiler in a hosted notebook, use one of the options below

|Jupyter Lite|Binder|
|:---:|:---:|
|[![Lite](https://gist.githubusercontent.com/willeppy/35cdc20a3fc26e393ce76f1df35bcdfc/raw/a7fca1d0a2d62c2b49f60c0217dffbd0fe404471/lite-badge-launch-small.svg)](http://dig.cmu.edu/AutoProfiler) | [![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/cmudig/AutoProfiler/HEAD?labpath=examples%2FSF%20Housing%20Demo.ipynb) |

## Development Install

For development install instructions, see [CONTRIBUTING.md](CONTRIBUTING.md).

If you're having install issues, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

## Acknowledgements

Big thanks to the Rill Data team! Much of our profiler UI code is adapted from [Rill Developer](https://github.com/rilldata/rill-developer).

## Let us know what you think! 📢

We would love to hear your feedback on how you are using AutoProfiler! Please fill out [this form](https://forms.gle/V3ejpXxMcQXqYJG48) or email Will at [willepp@cmu.edu](mailto:willepp@cmu.edu).
