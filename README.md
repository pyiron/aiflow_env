# Installation

This repository holds the [pixi](https://pixi.sh) environment for developing
[aiflow](https://github.com/pyiron/aiflow) and
[aiflow_nodes](https://github.com/pyiron/aiflow_nodes) together. Both are
cloned into the root of this repository as independent git repositories (not
submodules), so you can branch, commit and push in each of them as usual. They
are listed in `.gitignore`.

```
aiflow_env/
├── README.md
├── pixi.toml
├── pixi.lock
├── aiflow/          # git clone, installed editable into the environment
└── aiflow_nodes/    # git clone, installed editable into the environment
```

The environment supports `linux-64` and `osx-arm64` (Apple Silicon). Intel
Macs (`osx-64`) are not supported, since conda-forge has no recent TensorFlow
build for them. On macOS, GPAW is not installed, because the only osx-arm64
build needs a newer numpy than pyiron_atomistics allows. All other packages
are the same, and only the GPAW calculator nodes in aiflow_nodes are
unavailable.

## 1. Install pixi

```bash
curl -fsSL https://pixi.sh/install.sh | sh
```

Restart your shell (or `source ~/.bashrc`) so `pixi` is on your `PATH`, then
check it works:

```bash
pixi --version
```

See the [pixi installation docs](https://pixi.sh/latest/installation/) for
other options.

## 2. Clone aiflow_env

```bash
git clone git@github.com:pyiron/aiflow_env.git
cd aiflow_env
```

All following commands run from this directory. `git pull` here later picks up
changes to `pixi.toml` and `pixi.lock`; rerun `pixi install` afterwards.

## 3. Clone aiflow_nodes

```bash
git clone git@github.com:pyiron/aiflow_nodes.git
```

You can also work with your own fork instead.

```bash
git clone git@github.com:<your-user>/aiflow_nodes.git aiflow_nodes
```

## 4. Clone aiflow

```bash
git clone git@github.com:pyiron/aiflow.git
```

Or, from your own fork:

```bash
git clone git@github.com:<your-user>/aiflow.git aiflow
```

Either way the clones must be named `aiflow` and `aiflow_nodes` (the explicit
target directory above takes care of renamed forks), since `pixi.toml` refers
to them by these paths.

`pixi.toml` installs aiflow from `./aiflow` (editable, with the `all` extras)
and aiflow_nodes from `./aiflow_nodes` (editable), so the clones must exist
before the next step. Their conda-forge dependencies come from their own conda
recipes, which `pixi.toml` pulls in through its `[dev]` table (see
[Dependencies](#dependencies)).

## 5. Install the pixi environment

```bash
pixi install
```

This resolves the environment from `pixi.lock` and installs it into `.pixi/`.
The first run downloads several GB (LAMMPS, GPAW, TensorFlow, ...) and takes a
while.

## 6. Build the aiflow GUI

The `aiflow.gui` widget is written in TypeScript (`aiflow/frontend`) and is
compiled with npm. Node.js is part of the pixi environment, so build it with:

```bash
pixi run gui-build
```

This runs `npm install` and `npm run build` in `aiflow/frontend` and writes
`widget.js`, `widget.css` and `ngl.js` to `aiflow/src/aiflow/gui/static`.
Without this step the GUI shows an empty widget.

When working on the frontend, `pixi run gui-dev` rebuilds on every change
(restart the Jupyter kernel to pick it up) and `pixi run gui-test` runs the
jest tests.

## 7. Activate the pixi environment

```bash
pixi shell
```

This starts a subshell with the environment active; leave it with `exit`.
Activation also sets `GLIBC_TUNABLES=glibc.rtld.execstack=2`, needed for the
conda-forge TensorFlow build on glibc >= 2.41 (has no effect on macOS).

To run a single command without a subshell use `pixi run <command>`, e.g.
`pixi run lab` to start JupyterLab.

Check the installation:

```bash
python -c "import aiflow_nodes, aiflow; print('ok')"
```

## 8. Install the pre-commit hooks

`pre-commit` is part of the pixi environment. With the environment active,
install the hooks in each repository:

```bash
cd aiflow && pre-commit install && cd ..
cd aiflow_nodes && pre-commit install && cd ..
```

- **aiflow** installs a `pre-commit` hook (ruff lint and format) and a
  `pre-push` hook that runs the fast unit tests with `python -m pytest`.
  The push hook uses the active Python, so push from inside `pixi shell`.
- **aiflow_nodes** installs a `pre-commit` hook (ruff lint and format).

To run all hooks over the whole repository once, e.g. after installing:

```bash
pre-commit run --all-files
```

Skip the hooks for a single commit or push with `--no-verify`.

# Updating

All commands run from the root of this repository.

## Update the environment definition

```bash
git pull
pixi install
```

This picks up changes to `pixi.toml` and `pixi.lock`. `pixi shell` and
`pixi run` also bring the environment in sync with the lock file
automatically, so `pixi install` mainly saves you the wait later.

## Update aiflow and aiflow_nodes

Pull in each clone as usual:

```bash
git -C aiflow pull
git -C aiflow_nodes pull
```

Both are installed editable, so Python code changes take effect immediately.
These cases need an extra step:

- **aiflow's or aiflow_nodes' dependencies changed** (`pyproject.toml` and
  `recipe/recipe.yaml`): run `pixi install`, which re-resolves the
  environment and updates `pixi.lock`. A new dependency goes into the package
  (see [Dependencies](#dependencies)), not into `pixi.toml`.
- **`aiflow/frontend` changed**: rebuild the GUI with `pixi run gui-build` and
  restart the Jupyter kernel.

## Update dependency versions

To move the packages in `pixi.lock` to the newest versions allowed by
`pixi.toml`:

```bash
pixi update
```

Run the tests, then commit the new `pixi.lock` (and `pixi.toml`, if you
changed version pins) to this repository.

## Update the pre-commit hooks

The hook versions are pinned in each repository's `.pre-commit-config.yaml`,
so a `git pull` there updates them and `pre-commit` installs the new versions
on its next run. Nothing needs to be reinstalled.

# Dependencies

Each requirement is written down once, in the layer that owns it:

| Layer | Where | Holds | Versions |
|---|---|---|---|
| Requirements | `[project]` in `aiflow/pyproject.toml` and `aiflow_nodes/pyproject.toml` | everything the code imports: the core in `dependencies`, the rest in extras | lower bounds, or caps for known API breaks |
| Conda packaging | `recipe/recipe.yaml` in each repository | one output per extra (`aiflow-gui`, `aiflow-nodes-lammps`, ...), plus only what PyPI cannot provide (LAMMPS, data packages) | as in `[project]` |
| Environment | `pixi.toml` and `pixi.lock` here | the packages' recipe outputs in `[dev]`, the editable installs, the tools only this bundle uses (JupyterLab, Node.js, ...), `[constraints]` for known breakage | exact, in `pixi.lock` |

The recipes mirror `[project]`; `tools/check_conda_recipe.py` in each
repository (pre-commit hook, CI) fails when they disagree. Listing a recipe
output in `[dev]` installs its requirements without building it; the packages
themselves are the editable PyPI installs. The recipes are also the templates
for the conda-forge feedstocks.

Some of this waits for publishing; see the TODOs in `pixi.toml` and the
repositories, and the checklist below.

## Publishing checklist

Prepared but not done yet, in this order:

1. **Rename the aiflow distribution.** `aiflow` on PyPI belongs to an unrelated
   project. Rename it in `aiflow/pyproject.toml`, the recipe outputs,
   `aiflow/.github/workflows/release.yml`, the aiflow requirement of
   aiflow_nodes and aiflow-evals, and `pixi.toml` here. The import name
   `aiflow` can stay.
2. **License aiflow_nodes.** It has no license yet; PyPI metadata and
   conda-forge need one (`license` in `pyproject.toml`, `license` and
   `license_file` in its recipe).
3. **Make `pyiron/aiflow` public**, then drop the `AIFLOW_READ` token from
   aiflow_nodes' CI.
4. **Get sim-atlas and sim-atlas-toolkit 0.0.19 onto conda-forge** (conda-forge
   has 0.0.17). Then replace `anywidget`, `ipytree` and `ipywidgets` in
   `pixi.toml` with `aiflow-all` in `[dev]`, and make the `conda-recipe` CI job
   in aiflow blocking.
5. **Release aiflow** to PyPI (`release.yml`), then submit its recipe to
   [conda-forge/staged-recipes](https://github.com/conda-forge/staged-recipes),
   with `source` replaced by the PyPI sdist (url and sha256).
6. **aiflow_nodes:** require a released aiflow (`aiflow>=…` instead of the git
   revs in `[tool.pixi.dev]` and `[tool.pixi.pypi-dependencies]`), release it
   to PyPI (it needs a release workflow like aiflow's), submit its recipe to
   staged-recipes, and make its `conda-recipe` CI job blocking.
7. **Here:** add a user environment that installs `aiflow-all` and the
   `aiflow-nodes-*` outputs from conda-forge; keep the development environment
   on the clones.

Once the feedstocks exist, the conda-forge bot follows the PyPI releases, and
the check scripts keep the in-repository recipes in step with `[project]`.

## If using vscode

Set python interpreter to .pixi/envs/default/bin/python
