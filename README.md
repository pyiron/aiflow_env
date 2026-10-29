# Installation

This repository holds the [pixi](https://pixi.sh) environment for developing
[aiflow](https://github.com/pyiron/aiflow) and
[pyiron_nodes](https://github.com/pyiron/pyiron_nodes) together. Both are
cloned into the root of this repository as independent git repositories (not
submodules), so you can branch, commit and push in each of them as usual. They
are listed in `.gitignore`.

```
aiflow_env/
├── README.md
├── pixi.toml
├── pixi.lock
├── aiflow/          # git clone, installed editable into the environment
└── pyiron_nodes/    # git clone, importable via PYTHONPATH
```

The environment supports `linux-64` and `osx-arm64` (Apple Silicon). Intel
Macs (`osx-64`) are not supported, since conda-forge has no recent TensorFlow
build for them. On macOS, GPAW is not installed, because the only osx-arm64
build needs a newer numpy than pyiron_atomistics allows. All other packages
are the same, and only the GPAW calculator nodes in pyiron_nodes are
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

## 3. Clone pyiron_nodes

```bash
git clone git@github.com:pyiron/pyiron_nodes.git
```

You can also work with your own fork instead.

```bash
git clone git@github.com:<your-user>/pyiron_nodes.git pyiron_nodes
```

## 4. Clone aiflow

```bash
git clone git@github.com:pyiron/aiflow.git
```

Or, from your own fork:

```bash
git clone git@github.com:<your-user>/aiflow.git aiflow
```

Either way the clones must be named `aiflow` and `pyiron_nodes` (the explicit
target directory above takes care of renamed forks), since `pixi.toml` refers
to them by these paths.

`pixi.toml` installs aiflow from `./aiflow` (editable, with the `all` extras)
and `core_shim` from `./aiflow/core_shim`, so the clone must exist before the
next step.

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
Activation also sets:

- `PYTHONPATH` to this directory, so `import pyiron_nodes` works;
- `GLIBC_TUNABLES=glibc.rtld.execstack=2`, needed for the conda-forge
  TensorFlow build on glibc >= 2.41 (has no effect on macOS).

To run a single command without a subshell use `pixi run <command>`, e.g.
`pixi run lab` to start JupyterLab.

Check the installation:

```bash
python -c "import pyiron_nodes, aiflow; print('ok')"
```

## 8. Install the pre-commit hooks

`pre-commit` is part of the pixi environment. With the environment active,
install the hooks in each repository:

```bash
cd aiflow && pre-commit install && cd ..
cd pyiron_nodes && pre-commit install && cd ..
```

- **aiflow** installs a `pre-commit` hook (ruff lint and format) and a
  `pre-push` hook that runs the fast unit tests with `python -m pytest`.
  The push hook uses the active Python, so push from inside `pixi shell`.
- **pyiron_nodes** installs a `pre-commit` hook (ruff lint/format and black).

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

## Update aiflow and pyiron_nodes

Pull in each clone as usual:

```bash
git -C aiflow pull
git -C pyiron_nodes pull
```

pyiron_nodes is imported from the clone and aiflow is installed editable, so
Python code changes take effect immediately. Three cases need an extra step:

- **aiflow's dependencies changed** (its `pyproject.toml`): run `pixi install`,
  which re-resolves the environment and updates `pixi.lock`.
- **`aiflow/core_shim` changed**: it is installed non-editable, so reinstall it
  with `pixi reinstall core_shim`.
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
