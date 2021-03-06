# Guild config

The module `guild.config` handles user config.

    >>> from guild import config

## User config inheritance

User config can use the `extends` construct to extend other config
sections.

The function `_apply_config_inherits` can be used with parsed user
config as a dict. We'll create a helper function to print the resolved
structure.

    >>> def apply(data):
    ...    config._apply_config_inherits(data, "test")
    ...    pprint(data)

NOTE: For the test below, we'll use the naming convention `sN` for
sections, `iN` for section items, and `aN` for item attributes.

A section item may extend other items in the same section.

    >>> apply({
    ...   "s1": {"i1": {"a1": 1, "a2": 2},
    ...          "i2": {"extends": "i1", "a2": 3}}
    ... })
    {'s1': {'i1': {'a1': 1, 'a2': 2},
            'i2': {'a1': 1, 'a2': 3}}}

It may also extend items under the `config` section.

    >>> apply({
    ...   "config": {"i1": {"a1": 1, "a2": 2}},
    ...   "s1": {"i2": {"extends": "i1", "a2": 3}}
    ... })
    {'s1': {'i2': {'a1': 1, 'a2': 3}}}

If the name occurs in both the current section and the `config`
section, the item under the current section is selected.

    >>> apply({
    ...   "config": {"i1": {"a1": 1, "a2": 2}},
    ...   "s1": {"i1": {"a1": 3, "a2": 4},
    ...          "i2": {"extends": "i1", "a2": 5}}
    ... })
    {'s1': {'i1': {'a1': 3, 'a2': 4},
            'i2': {'a1': 3, 'a2': 5}}}

Cycles are silently ignored by dropping the last leg of the cycle.

    >>> apply({
    ...   "s1": {"i1": {"extends": "i1"}}
    ... })
    {'s1': {'i1': {}}}

    >>> apply({
    ...   "s1": {"i1": {"extends": "i2", "a1": 1, "a2": 2},
    ...          "i2": {"extends": "i1", "a2": 3, "a3": 4}}
    ... })
    {'s1': {'i1': {'a1': 1, 'a2': 2, 'a3': 4},
            'i2': {'a1': 1, 'a2': 3, 'a3': 4}}}

    >>> apply({
    ...   "config": {"i1": {"extends": "i2", "a1": 1, "a2": 2},
    ...              "i2": {"extends": "i3", "a1": 3, "a3": 4},
    ...              "i3": {"extends": "i1", "a1": 5}},
    ...   "s1": {"i4": {"extends": "i1", "a1": 6, "a4": 7},
    ...          "i5": {"extends": "i2", "a2": 8, "a5": 9}}
    ... })
    {'s1': {'i4': {'a1': 6, 'a2': 2, 'a3': 4, 'a4': 7},
            'i5': {'a1': 3, 'a2': 8, 'a3': 4, 'a5': 9}}}

If a parent can't be found, `config.ConfigError` is raised.

    >>> apply({
    ...   "s1": {"i1": {"extends": "i2"}}
    ... })
    Traceback (most recent call last):
    ConfigError: cannot find 'i2' in test

## _Config objects

The class `config._Config` can be used to read config data.

    >>> cfg = config._Config(sample("config/remotes.yml"))
    >>> pprint(cfg.read())
    {'remotes': {'v100': {'ami': 'ami-4f62582a',
                          'description': 'V100 GPU running on EC2',
                          'init': 'echo hello',
                          'instance-type': 'p3.2xlarge',
                          'region': 'us-east-2',
                          'type': 'ec2'},
                 'v100x8': {'ami': 'ami-4f62582a',
                            'description': 'V100 x8 running on EC2',
                            'init': 'echo hello there',
                            'instance-type': 'p3.16xlarge',
                            'region': 'us-east-2',
                            'type': 'ec2'}}}

## Python Executable

When running a Python program on behalf of a user (as opposed to
running a Python program to implement an internal Guild task, usually
to provide process isolation), the Python interpreter is provide by
`config.python_exe()`.

The logic used to select the appropriate Python interpreter is as
follows:

- Use `GUILD_PYTHON_EXE` env var if available
- Use `CONDA_PYTHON_EXE` env var if available
- Use `bin/python` under `VIRTUAL_ENV` if exists
- Use value returned by `which python` (emulates /usr/bin/env python)
- Use `sys.interpreter`

To test the complete logic, we need to replace `util.which` with a
function that always returns None.

    >>> from guild import util
    >>> which_save = util.which
    >>> util.which = lambda _: None

    >>> with Env({}, replace=True):
    ...     exe = config.python_exe()

Without any environment and with `util.which` returning None, the
Python exe is the same as `sys.exectuable`.

    >>> exe == sys.executable, (exe, sys.executable)
    (True, ...)

Next, we use a function for `guild.which` that returns a marker.

    >>> marker = object()
    >>> util.which = lambda _: marker

    >>> with Env({}, replace=True):
    ...     exe = config.python_exe()

    >>> exe is marker, exe
    (True, ...)

If `VIRTUAL_ENV` is defined in the environment, `bin/python` in that
directory is returned, provided it exists. Otherwise, `which python`
is returned.

    >>> venv_dir = mkdtemp()

    >>> with Env({"VIRTUAL_ENV": venv_dir}, replace=True):
    ...     exe = config.python_exe()

In this case, `bin/python` doesn't exist in the environment, so the
exe is our marker.

    >>> exe is marker, exe
    (True, ...)

Let's create `bin/python` in the env.

    >>> mkdir(path(venv_dir, "bin"))
    >>> venv_python_exe = path(venv_dir, "bin", "python")
    >>> touch(venv_python_exe)

    >>> with Env({"VIRTUAL_ENV": venv_dir}, replace=True):
    ...     exe = config.python_exe()

This time, the Python exe is the venv exe.

    >>> exe == venv_python_exe, (exe, venv_python_exe)
    (True, ...)

If `CONDA_PYTHON_EXE` is defined in the environment, it is always
provided, even if the exe doesn't exist.

    >>> conda_python_exe = path(venv_dir, "conda-exe")

    >>> with Env({"CONDA_PYTHON_EXE": conda_python_exe,
    ...           "VIRTUAL_ENV": venv_dir}, replace=True):
    ...     exe = config.python_exe()

    >>> exe == conda_python_exe, (exe, conda_python_exe)
    (True, ...)

Finally, if `GUILD_PYTHON_EXE` is defined, it is always used,
regardless of other environment variables.

    >>> guild_python_exe = path(venv_dir, "guild-exe")

    >>> with Env({"GUILD_PYTHON_EXE": guild_python_exe,
    ...           "CONDA_PYTHON_EXE": conda_python_exe,
    ...           "VIRTUAL_ENV": venv_dir}, replace=True):
    ...     exe = config.python_exe()

    >>> exe == guild_python_exe, (exe, guild_python_exe)
    (True, ...)

Restore `util.which`:

    >>> util.which = which_save
