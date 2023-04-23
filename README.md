# Rye

Rye is Armin's personal one-stop-shop for all his Python needs.  It installs and
manages Python installations, manages `pyproject.toml` files, installs and
uninstalls dependencies, manages virtualenvs behind the scenes.  It supports both
monorepos as well as global tool installations.

It is a wish of what Python was, with no guarantee to work for anyone else.  It's
an exploration, and it's far from perfect.

## Some of the things it does

It automatically installs and manages Python:

```
$ rye pin 3.1
$ rye run python
Python 3.11.1 (main, Jan 16 2023, 16:02:03) [Clang 15.0.7 ] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

Install tools in isolation globally:

```
$ rye install maturin
```

Manage dependencies of a local `pyproject.toml` and update the virtualenv
automatically:

```
$ rye add flask
$ rye sync
```

## Installation

Rye is built in Rust.  There is no binary distribution yet, it only works on Linux
and macOS as of today:

```
$ cargo install --git https://github.com/mitsuhiko/rye rye
```

After installing `rye`, all you need to enjoy automatic management of everything
is `rye sync` (and optionally `rye pin` to pick a specific Python version):

```shell
$ rye pin cpython@3.11
$ rye sync
```

The virtualenv that `rye` manages is placed in `.venv` next to your `pyproject.toml`.
You can activate and work with it as normal with one notable exception: the Python
installation in it does not contain `pip`.

Note that `python` will by default just be your regular Python.  To have it automatically
pick up the right Python without manually activating the virtualenv, you can add
`~/.rye/shims` to your `PATH` at higher preference than normal.  If you operate outside
of a rye managed project, the regular Python is picked up automatically.  For the global
tool installation you need to add the shims to the path.

## Decisions Made

To understand why things are the way they are:

* **Virtualenvs:** while I personally do not like virtualenvs that much, they are
  so widespread and have reasonable tooling support, so I chose this over
  `__pypackages__`.

* **No Default Dependencies:** the virtualenvs when they come up are completely void
  of dependencies.  Not even `pip` or `setuptools` are installed into it.  Rye
  manages the virutalenv from outside the virtualenv.

* **No Core Proprietary Stuff:** Rye (with the exception of it's own `tool` section
  in the `pyproject.toml`) uses standardized keys.  That means it uses regular
  requirements as you would expect.  It also does not use a proprietary lock file
  and use [`pip-tools`](https://github.com/jazzband/pip-tools) behind the scenes.

* **No Pip:** Rye uses pip, but it does not expose it.  I manage dependencies in
  `pyproject.toml` only.

* **No System Python:** I can't deal with any more linux distribution weird Python
  installations or whatever mess there is on macOS.  I used to build my own Pythons
  that are the same everywhere, not I use [indygreg's Python builds](https://github.com/indygreg/python-build-standalone).
  Rye will automatically download and manage Python builds from there.  No compiling,
  no divergence.

* **Project Local Shims:** Rye maintains a `python` shim that auto discovers the
  current `pyproject.toml` and automatically operates below it.  Just add the
  shims to your shell and you can run `python` and it will automatically always
  operate in the right project.

## What Could Be?

There are a few shortcomings in the Python packaging world, largely as a result of
lack of standardization.  Here is what this project ran into over the years:

* **No Python Binary Distributions:** CPython builds from python.org are completely
  inadequate.  On some platforms you only get an .msi in installer, on some you
  literally only get tarballs.  The various Python distributions that became popular
  over the years are diverging greatly and cause all kinds of nonsense downstream.
  This is why this Project uses the indygreg standalone builds.  I hope that with
  time someone will start distributing well maintained and reliable Python builds
  to replace the mess we are dealing with today.

* **No Dev Dependencies:** Rye currently needs a custom section in the `pyproject.toml`
  to represent dev dependencies.  There is no standard in the ecosystem for this.  It
  really should be added.

* **No Local Dependency Overlays:** There is no standard for how to represent local
  dependencies.  Rust for this purpose has something like `{ path = "../foo" }`
  which allows both remote and local references to co-exist and it rewrites them
  on publish.

* **No Exposed Pip:** pip is intentionally not exposed.  If you were to install somethign
  into the virtualenv, it disappears next time you sync.  If you symlink `rye` to
  `~/.rye/shims/pip` you can get access to pip without installating it into the
  virtualenv.  There be dragons.

* **No Workspace Spec:** for monorepos and things of that nature, the Python ecosystem
  would need a definition of workspaces.  Today that does not exist which forces every
  tool to come up with it's own solutions to this problem.

* **No Basic Script Section:** There should be a standard in `pyproject.toml` to
  represent scripts like `rye` does in `rye.tools.scripts`.

## Adding Dependencies

To add a new dependency run `rye add` with the name of the package that you want to
install.  Additionally a proprietary extension to `pyproject.toml` exists to add
development only packages.  For those add `--dev`.

```shell
$ rye add "flask>=2.0"
$ rye add --dev black
```

Adding dependencies will not directly install them.  To install them run `rye sync` again.

## Workspaces

To have multiple projects share the same virtualenv, it's possible to declare workspaces
in the `pyproject.toml`:

```toml
[tool.rye.workspace]
members = ["foo-*"]
```

When `rye sync` is run in a workspace, then all packages are installed at all times.  This
also means that they can inter-depend as they will all be installed editable by default.

## Lockfiles

Rye does not try to re-invent the world (yet!).  This means it uses `pip-tools` behind
the scenes automatically.  As neither `pip` nor `pip-tools` provide lockfiles today
Rye uses generated `requirements.txt` files as replacement.  Whenever you run
`rye sync` it updates the `requirements.lock` and `requirements-dev.lock` files
automatically.

## Scripts

`rye run` can be used to invoke a binary from the virtualenv or a configured script.
Rye allows you to define basic scripts in the `pyproject.toml` in the `tool.rye.scripts`
section:

```toml
[tool.rye.scripts]
serve = "python -m http.server 8000"
```

They are only available via `rye run <script_name>`.  They can either be a string or an
array where each item is an argument to the script.  The scripts will be run with the
virtualenv activated.

To see what's available run `rye run` without arguments and it will list all scripts.

## Python Distributions

Rye does not use system python installations.  Instead it uses Gregory Szorc's standalone
Python builds: [python-build-standalone](https://github.com/indygreg/python-build-standalone).
This is done to create a unified experience of Python installations and to avoid
incompatibilities created by different Python distributions.  Most importantly this also
means you never need to compile a Python any more, it just downloads prepared binaries.

## Global Tools

If you want tools to be installed into isolated virtualenvs (like pipsi and pipx), you
can use `rye` too (requires `~/.rye/shims` to be on the path):

```
$ rye install pycowsay
$ pycowsay Wow

  ---
< Wow >
  ---
   \   ^__^
    \  (oo)\_______
       (__)\       )\/\
           ||----w |
           ||     ||
```

To uninstall run `rye uninstall pycowsay` again.

## Using The Virtualenv

There are two ways to use the virtual environment.  One is to just activate it like you
would do normally:

```shell
$ . .venv/bin/activate
```

The other is to use `rye run <script>` to run a script (installed into `.venv/bin`) in
the context of the virtual environment.  This for instance can be used to run black:

```shell
$ rye add --dev black
$ rye sync
$ rye run black .
```

License: MIT