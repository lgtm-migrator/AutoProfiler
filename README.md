# AutoProfile

Profile your Pandas Dataframes! Autoprofile will automatically visualize your Pandas dataframes after every execution, no extra code necessary. 

There's more bugs than features right now so buyer beware!

![screenshot of autoprofile](examples/basic_profiler_sc.png)

## Short Demo

Autoprofile reads your current Jupyter notebook and produces profiles for the Pandas Dataframes in your memory along with some summary statistics and a data preview.

![demo of autoprofile](examples/demo.gif)


## Requirements

* JupyterLab >= 3.0
* Pandas 

# Install

Pip install coming soon :)

<!-- To install the extension, execute:

```bash
pip install AutoProfile
``` -->

## Development install

Note: You will need NodeJS to build the extension package.

The `jlpm` command is JupyterLab's pinned version of
[yarn](https://yarnpkg.com/) that is installed with JupyterLab. You may use
`yarn` or `npm` in lieu of `jlpm` below.

```bash
# Clone the repo to your local environment
# Change directory to the AutoProfile directory
# Install package in development mode
pip install -e .
# Link your development version of the extension with JupyterLab
jupyter labextension develop . --overwrite
# OR
jupyter labextension install .
# Rebuild extension Typescript source after making changes
npm run build
```

Jupyter labextension can be weird sometimes; nuking the conda env and restarting tends to fix it.

You can watch the source directory and run JupyterLab at the same time in different terminals to watch for changes in the extension's source and automatically rebuild the extension.

```bash
# Watch the source directory in one terminal, automatically rebuilding when needed
jlpm watch
# Run JupyterLab in another terminal
jupyter lab
```

With the watch command running, every saved change will immediately be built locally and available in your running JupyterLab. Refresh JupyterLab to load the change in your browser (you may need to wait several seconds for the extension to be rebuilt).

By default, the `jlpm build` command generates the source maps for this extension to make it easier to debug using the browser dev tools. To also generate source maps for the JupyterLab core extensions, you can run the following command:

```bash
jupyter lab build --minimize=False
```

### Development uninstall

```bash
pip uninstall AutoProfile
```

In development mode, you will also need to remove the symlink created by `jupyter labextension develop`
command. To find its location, you can run `jupyter labextension list` to figure out where the `labextensions`
folder is located. Then you can remove the symlink named `AutoProfile` within that folder.