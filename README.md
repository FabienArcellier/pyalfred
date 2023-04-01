## Alfred

Alfred is an extensible building tool that can replace a Makefile or Fabric.
Writing commands in python is done in a few minutes, even in the case of a mono-repository
which contains several products.

In this dev, we are eating our own dog food. We are using `alfred` for the continuous integration process
of itself instead of `Makefile` as I usually do.

```bash
# run the continuous integration process
alfred ci

# publish the package on pypi
alfred publish
```

[![version](https://img.shields.io/pypi/v/alfred-cli.svg?label=version)](https://pypi.org/project/alfred-cli/) [![MIT](https://img.shields.io/badge/license-MIT-007EC7.svg)](LICENSE.md)

[![ci](https://github.com/FabienArcellier/alfred-cli/actions/workflows/ci.yml/badge.svg)](https://github.com/FabienArcellier/alfred-cli/actions/workflows/ci.yml) [![ci-windows](https://github.com/FabienArcellier/alfred-cli/actions/workflows/ci-windows.yml/badge.svg)](https://github.com/FabienArcellier/pyalfred/actions/workflows/ci-windows.yml)

<!-- TOC start -->
- [Getting started](#getting-started)
  * [Add a new build command](#add-a-new-build-command)
- [Behind the scene](#behind-the-scene)
- [Why using alfred instead of Makefile or Bash scripts](#why-using-alfred-instead-of-makefile-or-bash-scripts)
- [Why not using alfred](#why-not-using-alfred)
- [The latest version](#the-latest-version)
- [Reference](#reference)
  * [`.Alfred.yml`](#alfredyml)
    + [`Plugins` section](#plugins-section)
    + [`Environment` section](#environment-section)
- [Cookbook](#cookbook)
  * [Display the commands really executed behind the scene](#display-the-commands-really-executed-behind-the-scene)
  * [Customize a command for a specific OS](#customize-a-command-for-a-specific-os)
  * [Override environment variables](#override-environment-variables)
    + [Add directories into pythonpath](#add-directories-into-pythonpath)
- [Developper guideline](#developper-guideline)
  * [Install development environment](#install-development-environment)
  * [Install production environment](#install-production-environment)
  * [Initiate or update the library requirements](#initiate-or-update-the-library-requirements)
  * [Activate the python environment](#activate-the-python-environment)
  * [Run the linter and the unit tests](#run-the-linter-and-the-unit-tests)
- [Contributors](#contributors)
- [License](#license)
<!-- TOC end -->

## Getting started

To configure a python project to use alfred, here is the procedure:

```bash
pip3 install alfred-cli
alfred init
```

A hello_world command was created for the example:

```bash
alfred hello_world --name "Fabien"
```

A file `.alfred.yml` will be initialized at the root of the repository.

### Add a new build command

You can add your command in a new module in `./alfred`.
In this example we will add the command `alfred lint` :

```python
import os

import alfred

ROOT_DIR = os.path.realpath(os.path.join(__file__, "..", ".."))

@alfred.command('lint', help="validate alfred using pylint on the package alfred")
def lint():
    # get the command pylint in the user system or show error message if it's missing
    pylint = alfred.sh('pylint', "pylint is not installed")

    # behind the scene, it invokes the command `pylint alfred`
    alfred.run(pylint, ["src/alfred"])
```

## Behind the scene

Alfred rely heavily on click and plumblum :

* [click](https://click.palletsprojects.com/en/8.0.x/)
* [plumblum](https://plumbum.readthedocs.io/en/latest/)

## Why using alfred instead of Makefile or Bash scripts

One of the advantages of `bash` and `Makefile` is their native presence in many environments.
By default, a `Makefile` allows you to segment these commands efficiently. Autocompletion is first-citizen
feature. Alfred doesn't have it yet.

Alfred allows you to create more complex commands than with Make. From the start, you benefit from a
formatted documentation for each of your orders. It is easy to create one command per file  thanks
to auto discovery. You can see an implementation in this repository in [`alfred_cmd/`](alfred/).

Thanks to the power of Click, it's easy to add options to your commands.
They allow for example to implement flags for your CI process which
offer you an execution for the frontend.

Alfred allows you to mix shell code with python instructions. In some cases, it allows you
to perform efficient processing on API calls. You can use either the cli (for git, ...) or
pythons libraries depending on the nature of the treatment you want to perform.

In our development process, we frequently need to operate on application with several process (frontend in react,
server in flask, two external service in flask). To mount those process, we use `honcho` with alfred
to load `Procfile` that will manage those process.

## Why not using alfred

If you want to create a cli you will distribute, alfred is not designed for that. I won't recommand
as well to use it to build a data application even if you can use python and many library.

Alfred command can import only installed library. You can't use relative import. That makes difficult to
share code between your commands.

## The latest version

You can find the latest version to ...

```bash
git clone https://github.com/FabienArcellier/alfred-cli.git
```

## Reference

### `.Alfred.yml`

The configuration file supports several attributes to tune the behavior of
Alfred. It is required to add

#### `Plugins` section

the `plugins` section tells Alfred where to look for commands to render
accessible to the user. The pointed folder contains several python modules which
are loaded one by one. The `__init __. Py` module is ignored.

If you specify a prefix, the commands configured in the imported modules
will be prefixed by this label in alfred. This feature is useful in a mono-repository
when you want to have an .alfred.yml file for each project and an .alfred.yml file
at the root of the project.

```yaml
plugins:
    - path: alfred

    - path: sofware1/alfred
      prefix: "software1:"
```

#### `Environment` section

`environment` section allows you to hard configure environment variables to load
before executing a command.

This feature facilitates the integration with `pipenv` for example which has a behavior
different between mac and linux. In this case, it allows you to set the necessary flags
to erase these differences.

```yaml
environment:
    - "VAR=1"
    - "VAR"
```

## Cookbook

### Display the commands really executed behind the scene

You can display the commands really executed, either to debug the arguments,
either to run in your terminal again with other attributes.

The option `d` / `--debug` display all the shell commands that are executed by
`alfred.run()` in your alfred command.

```bash
$ alfred -d ci

2022-02-07 19:38:31,834 DEBUG - /home/far/.local/share/virtualenvs/20210821_1530__alfred-cli-a8dwJte3/bin/python -m unittest discover units - wd: /home/far/documents/projects/20210821_1530__alfred-cli/tests
.
----------------------------------------------------------------------
Ran 1 test in 0.000s

OK
```

### Customize a command for a specific OS

Alfred can run a specific part of the build for an OS,
for example to only run the linter on a linux machine.

```python
@alfred.command('ci', help="execute continuous integration process of alfred")
@alfred.option('-v', '--verbose', is_flag=True)
def ci(verbose: bool):
    if alfred.is_posix():
        alfred.invoke_command('lint', verbose=verbose)
    else:
        print("linter is not supported on non posix platform as windows")

    alfred.invoke_command('tests', verbose=verbose)
```

the ``alfred.is_posix``, ``alfred.is_linux``, ``alfred.is_macos``, ``alfred.is_windows`` functions allow you to quickly
target the environment on which specific processing must be performed.

### Override environment variables

```python
@alfred.command('ci', help="execute continuous integration process of alfred")
def ci():
    with alfred.env(SCREEN="display"):
        bash = alfred.sh("bash")
        bash.run("-c" "echo $SCREEN")
```

#### Add directories into pythonpath

Adding a folder in the pythonpath variable allows you to expose packages without declaring them in the manifest.

This pattern is useful with poetry to be able to reuse the code of the package tests in this one for example.

The ``alfred.pythonpath`` decorator adds the project root. You can save specific folders here.

```python
@alfred.command('ci', help="execute continuous integration process of alfred")
@alfred.pythonpath()
def ci():
    with alfred.env(SCREEN="display"):
        bash = alfred.sh("bash")
        alfred.run(bash, ["-c" "echo $SCREEN"])
```

```python
@alfred.command('ci', help="execute continuous integration process of alfred")
@alfred.pythonpath(['tests'], append_project=False)
def ci():
    with alfred.env(SCREEN="display"):
        bash = alfred.sh("bash")
        alfred.run(bash, ["-c", "echo $SCREEN"])
```


## Developper guideline

```bash
pipenv install
pipenv shell
```

```
$ alfred
Usage: alfred [OPTIONS] COMMAND [ARGS]...

  alfred is a building tool to make engineering tasks easier to develop and to
  maintain

Options:
  -d, --debug  display debug information like command runned and working
               directory
  --help       Show this message and exit.

Commands:
  ci                 execute continuous integration process of alfred
  dist               build distribution packages
  lint               validate alfred using pylint on the package alfred
  publish            tag a new release and trigger pypi publication
  tests              validate alfred with all the automatic testing
  tests:acceptances  validate alfred with acceptances testing
  tests:units        validate alfred with unit testing
```

### Install development environment

Use make to instanciate a python virtual environment in ./venv and install the
python dependencies.

```bash
pipenv install --dev
```

### Install production environment

```bash
pipenv install
```

### Initiate or update the library requirements

If you want to initiate or update all the requirements `install_requires` declared in `setup.py`
and freeze a new `Pipfile.lock`, use this command

```bash
pipenv update
```

### Activate the python environment

When you setup the requirements, a `venv` directory on python 3 is created.
To activate the venv, you have to execute :

```bash
pipenv shell
```

### Run the linter and the unit tests

Before commit or send a pull request, you have to execute `pylint` to check the syntax
of your code and run the unit tests to validate the behavior.

```bash
alfred ci
```

## Contributors

* Fabien Arcellier

## License

MIT License

Copyright (c) 2021-2022 Fabien Arcellier

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
