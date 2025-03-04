setuptools_scm
==============

``setuptools_scm`` handles managing your Python package versions
in SCM metadata instead of declaring them as the version argument
or in a SCM managed file.

Additionally ``setuptools_scm`` provides setuptools with a list of files that are managed by the SCM
(i.e. it automatically adds all of the SCM-managed files to the sdist).
Unwanted files must be excluded by discarding them via ``MANIFEST.in``.

``setuptools_scm`` support the following scm out of the box:

* git
* mercurial



.. image:: https://github.com/pypa/setuptools_scm/workflows/python%20tests+artifacts+release/badge.svg
    :target: https://github.com/pypa/setuptools_scm/actions

.. image:: https://tidelift.com/badges/package/pypi/setuptools-scm
   :target: https://tidelift.com/subscription/pkg/pypi-setuptools-scm?utm_source=pypi-setuptools-scm&utm_medium=readme


``pyproject.toml`` usage
------------------------

The preferred way to configure ``setuptools_scm`` is to author
settings in a ``tool.setuptools_scm`` section of ``pyproject.toml``.

This feature requires Setuptools 42 or later, released in Nov, 2019.
If your project needs to support build from sdist on older versions
of Setuptools, you will need to also implement the ``setup.py usage``
for those legacy environments.

First, ensure that ``setuptools_scm`` is present during the project's
built step by specifying it as one of the build requirements.

.. code:: toml

    # pyproject.toml
    [build-system]
    requires = ["setuptools>=45", "wheel", "setuptools_scm[toml]>=6.0"]

Note that the ``toml`` extra must be supplied.

That will be sufficient to require ``setuptools_scm`` for projects
that support PEP 518 (`pip <https://pypi.org/project/pip>`_ and
`pep517 <https://pypi.org/project/pep517/>`_). Many tools,
especially those that invoke ``setup.py`` for any reason, may
continue to rely on ``setup_requires``. For maximum compatibility
with those uses, consider also including a ``setup_requires`` directive
(described below in ``setup.py usage`` and ``setup.cfg``).

To enable version inference, add this section to your pyproject.toml:

.. code:: toml

    # pyproject.toml
    [tool.setuptools_scm]

Including this section is comparable to supplying
``use_scm_version=True`` in ``setup.py``. Additionally,
include arbitrary keyword arguments in that section
to be supplied to ``get_version()``. For example:

.. code:: toml

    # pyproject.toml

    [tool.setuptools_scm]
    write_to = "pkg/version.py"


``setup.py`` usage
------------------

The following settings are considered legacy behavior and
superseded by the ``pyproject.toml`` usage, but for maximal
compatibility, projects may also supply the configuration in
this older form.

To use ``setuptools_scm`` just modify your project's ``setup.py`` file
like this:

* Add ``setuptools_scm`` to the ``setup_requires`` parameter.
* Add the ``use_scm_version`` parameter and set it to ``True``.

For example:

.. code:: python

    from setuptools import setup
    setup(
        ...,
        use_scm_version=True,
        setup_requires=['setuptools_scm'],
        ...,
    )

Arguments to ``get_version()`` (see below) may be passed as a dictionary to
``use_scm_version``. For example:

.. code:: python

    from setuptools import setup
    setup(
        ...,
        use_scm_version = {
            "root": "..",
            "relative_to": __file__,
            "local_scheme": "node-and-timestamp"
        },
        setup_requires=['setuptools_scm'],
        ...,
    )

You can confirm the version number locally via ``setup.py``:

.. code-block:: shell

    $ python setup.py --version

.. note::

   If you see unusual version numbers for packages but ``python setup.py
   --version`` reports the expected version number, ensure ``[egg_info]`` is
   not defined in ``setup.cfg``.


``setup.cfg`` usage
-------------------

as ``setup_requires`` is deprecated in favour of ``pyproject.toml``
usage in ``setup.cfg`` is considered deprecated,
please use ``pyproject.toml`` whenever possible.

Programmatic usage
------------------

In order to use ``setuptools_scm`` from code that is one directory deeper
than the project's root, you can use:

.. code:: python

    from setuptools_scm import get_version
    version = get_version(root='..', relative_to=__file__)

See `setup.py Usage`_ above for how to use this within ``setup.py``.


Retrieving package version at runtime
-------------------------------------

If you have opted not to hardcode the version number inside the package,
you can retrieve it at runtime from PEP-0566_ metadata using
``importlib.metadata`` from the standard library (added in Python 3.8)
or the `importlib_metadata`_ backport:

.. code:: python

    from importlib.metadata import version, PackageNotFoundError

    try:
        __version__ = version("package-name")
    except PackageNotFoundError:
        # package is not installed
        pass

Alternatively, you can use ``pkg_resources`` which is included in
``setuptools``:

.. code:: python

   from pkg_resources import get_distribution, DistributionNotFound

   try:
       __version__ = get_distribution("package-name").version
   except DistributionNotFound:
        # package is not installed
       pass

However, this does place a runtime dependency on ``setuptools`` and can add up to
a few 100ms overhead for the package import time.

.. _PEP-0566: https://www.python.org/dev/peps/pep-0566/
.. _importlib_metadata: https://pypi.org/project/importlib-metadata/


Usage from Sphinx
-----------------

It is discouraged to use ``setuptools_scm`` from Sphinx itself,
instead use ``importlib.metadata`` after editable/real installation:

.. code:: python

    from importlib.metadata import version

    release = version("package-name")
    version = '.'.join(release.split('.')[:2])

The underlying reason is, that services like *Read the Docs* sometimes change
the working directory for good reasons and using the installed metadata
prevents using needless volatile data there.

Notable Plugins
---------------

`setuptools_scm_git_archive <https://pypi.python.org/pypi/setuptools_scm_git_archive>`_
    Provides partial support for obtaining versions from git archives that
    belong to tagged versions. The only reason for not including it in
    ``setuptools_scm`` itself is Git/GitHub not supporting sufficient metadata
    for untagged/followup commits, which is preventing a consistent UX.


Default versioning scheme
-------------------------

In the standard configuration ``setuptools_scm`` takes a look at three things:

1. latest tag (with a version number)
2. the distance to this tag (e.g. number of revisions since latest tag)
3. workdir state (e.g. uncommitted changes since latest tag)

and uses roughly the following logic to render the version:

no distance and clean:
    ``{tag}``
distance and clean:
    ``{next_version}.dev{distance}+{scm letter}{revision hash}``
no distance and not clean:
    ``{tag}+dYYYYMMDD``
distance and not clean:
    ``{next_version}.dev{distance}+{scm letter}{revision hash}.dYYYYMMDD``

The next version is calculated by adding ``1`` to the last numeric component of
the tag.


For Git projects, the version relies on `git describe <https://git-scm.com/docs/git-describe>`_,
so you will see an additional ``g`` prepended to the ``{revision hash}``.

Semantic Versioning (SemVer)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Due to the default behavior it's necessary to always include a
patch version (the ``3`` in ``1.2.3``), or else the automatic guessing
will increment the wrong part of the SemVer (e.g. tag ``2.0`` results in
``2.1.devX`` instead of ``2.0.1.devX``). So please make sure to tag
accordingly.

.. note::

    Future versions of ``setuptools_scm`` will switch to `SemVer
    <http://semver.org/>`_ by default hiding the the old behavior as an
    configurable option.


Builtin mechanisms for obtaining version numbers
------------------------------------------------

1. the SCM itself (git/hg)
2. ``.hg_archival`` files (mercurial archives)
3. ``PKG-INFO``

.. note::

    Git archives are not supported due to Git shortcomings


File finders hook makes most of MANIFEST.in unnecessary
-------------------------------------------------------

``setuptools_scm`` implements a `file_finders
<https://setuptools.readthedocs.io/en/latest/setuptools.html#adding-support-for-revision-control-systems>`_
entry point which returns all files tracked by your SCM. This eliminates
the need for a manually constructed ``MANIFEST.in`` in most cases where this
would be required when not using ``setuptools_scm``, namely:

* To ensure all relevant files are packaged when running the ``sdist`` command.

* When using `include_package_data <https://setuptools.readthedocs.io/en/latest/setuptools.html#including-data-files>`_
  to include package data as part of the ``build`` or ``bdist_wheel``.

``MANIFEST.in`` may still be used: anything defined there overrides the hook.
This is mostly useful to exclude files tracked in your SCM from packages,
although in principle it can be used to explicitly include non-tracked files
too.


Configuration parameters
------------------------

In order to configure the way ``use_scm_version`` works you can provide
a mapping with options instead of a boolean value.

The currently supported configuration keys are:

:root:
    Relative path to cwd, used for finding the SCM root; defaults to ``.``

:version_scheme:
    Configures how the local version number is constructed; either an
    entrypoint name or a callable.

:local_scheme:
    Configures how the local component of the version is constructed; either an
    entrypoint name or a callable.

:write_to:
    A path to a file that gets replaced with a file containing the current
    version. It is ideal for creating a ``version.py`` file within the
    package, typically used to avoid using `pkg_resources.get_distribution`
    (which adds some overhead).

    .. warning::

      Only files with :code:`.py` and :code:`.txt` extensions have builtin
      templates, for other file types it is necessary to provide
      :code:`write_to_template`.

:write_to_template:
    A newstyle format string that is given the current version as
    the ``version`` keyword argument for formatting.

:relative_to:
    A file from which the root can be resolved.
    Typically called by a script or module that is not in the root of the
    repository to point ``setuptools_scm`` at the root of the repository by
    supplying ``__file__``.

:tag_regex:
   A Python regex string to extract the version part from any SCM tag.
    The regex needs to contain either a single match group, or a group
    named ``version``, that captures the actual version information.

    Defaults to the value of ``setuptools_scm.config.DEFAULT_TAG_REGEX``
    (see `config.py <src/setuptools_scm/config.py>`_).

:parentdir_prefix_version:
    If the normal methods for detecting the version (SCM version,
    sdist metadata) fail, and the parent directory name starts with
    ``parentdir_prefix_version``, then this prefix is stripped and the rest of
    the parent directory name is matched with ``tag_regex`` to get a version
    string.  If this parameter is unset (the default), then this fallback is
    not used.

    This is intended to cover GitHub's "release tarballs", which extract into
    directories named ``projectname-tag/`` (in which case
    ``parentdir_prefix_version`` can be set e.g. to ``projectname-``).

:fallback_version:
    A version string that will be used if no other method for detecting the
    version worked (e.g., when using a tarball with no metadata). If this is
    unset (the default), setuptools_scm will error if it fails to detect the
    version.

:parse:
    A function that will be used instead of the discovered SCM for parsing the
    version.
    Use with caution, this is a function for advanced use, and you should be
    familiar with the ``setuptools_scm`` internals to use it.

:git_describe_command:
    This command will be used instead the default ``git describe`` command.
    Use with caution, this is a function for advanced use, and you should be
    familiar with the ``setuptools_scm`` internals to use it.

    Defaults to the value set by ``setuptools_scm.git.DEFAULT_DESCRIBE``
    (see `git.py <src/setuptools_scm/git.py>`_).

To use ``setuptools_scm`` in other Python code you can use the ``get_version``
function:

.. code:: python

    from setuptools_scm import get_version
    my_version = get_version()

It optionally accepts the keys of the ``use_scm_version`` parameter as
keyword arguments.

Example configuration in ``setup.py`` format:

.. code:: python

    from setuptools import setup

    setup(
        use_scm_version={
            'write_to': 'version.py',
            'write_to_template': '__version__ = "{version}"',
            'tag_regex': r'^(?P<prefix>v)?(?P<version>[^\+]+)(?P<suffix>.*)?$',
        }
    )

Environment variables
---------------------

:SETUPTOOLS_SCM_PRETEND_VERSION:
    when defined and not empty,
    its used as the primary source for the version number
    in which case it will be a unparsed string


:SETUPTOOLS_SCM_PRETEND_VERSION_FOR_${UPPERCASED_DIST_NAME}:
    when defined and not empty,
    its used as the primary source for the version number
    in which case it will be a unparsed string

    it takes precedence over ``SETUPTOOLS_SCM_PRETEND_VERSION``


:SETUPTOOLS_SCM_DEBUG:
    when defined and not empty,
    a lot of debug information will be printed as part of ``setuptools_scm``
    operating

:SOURCE_DATE_EPOCH:
    when defined, used as the timestamp from which the
    ``node-and-date`` and ``node-and-timestamp`` local parts are
    derived, otherwise the current time is used
    (https://reproducible-builds.org/docs/source-date-epoch/)


:SETUPTOOLS_SCM_IGNORE_VCS_ROOTS:
    when defined, a ``os.pathsep`` separated list
    of directory names to ignore for root finding

Extending setuptools_scm
------------------------

``setuptools_scm`` ships with a few ``setuptools`` entrypoints based hooks to
extend its default capabilities.

Adding a new SCM
~~~~~~~~~~~~~~~~

``setuptools_scm`` provides two entrypoints for adding new SCMs:

``setuptools_scm.parse_scm``
    A function used to parse the metadata of the current workdir
    using the name of the control directory/file of your SCM as the
    entrypoint's name. E.g. for the built-in entrypoint for git the
    entrypoint is named ``.git`` and references ``setuptools_scm.git:parse``

  The return value MUST be a ``setuptools_scm.version.ScmVersion`` instance
  created by the function ``setuptools_scm.version:meta``.

``setuptools_scm.files_command``
  Either a string containing a shell command that prints all SCM managed
  files in its current working directory or a callable, that given a
  pathname will return that list.

  Also use then name of your SCM control directory as name of the entrypoint.

Version number construction
~~~~~~~~~~~~~~~~~~~~~~~~~~~

``setuptools_scm.version_scheme``
    Configures how the version number is constructed given a
    ``setuptools_scm.version.ScmVersion`` instance and should return a string
    representing the version.

    Available implementations:

    :guess-next-dev: Automatically guesses the next development version (default).
        Guesses the upcoming release by incrementing the pre-release segment if present,
        otherwise by incrementing the micro segment. Then appends :code:`.devN`.
        In case the tag ends with ``.dev0`` the version is not bumped
        and custom ``.devN`` versions will trigger a error.
    :post-release: generates post release versions (adds :code:`.postN`)
    :python-simplified-semver: Basic semantic versioning. Guesses the upcoming release
        by incrementing the minor segment and setting the micro segment to zero if the
        current branch contains the string ``'feature'``, otherwise by incrementing the
        micro version. Then appends :code:`.devN`. Not compatible with pre-releases.
    :release-branch-semver: Semantic versioning for projects with release branches. The
        same as ``guess-next-dev`` (incrementing the pre-release or micro segment) if on
        a release branch: a branch whose name (ignoring namespace) parses as a version
        that matches the most recent tag up to the minor segment. Otherwise if on a
        non-release branch, increments the minor segment and sets the micro segment to
        zero, then appends :code:`.devN`.
    :no-guess-dev: Does no next version guessing, just adds :code:`.post1.devN`

``setuptools_scm.local_scheme``
    Configures how the local part of a version is rendered given a
    ``setuptools_scm.version.ScmVersion`` instance and should return a string
    representing the local version.
    Dates and times are in Coordinated Universal Time (UTC), because as part
    of the version, they should be location independent.

    Available implementations:

    :node-and-date: adds the node on dev versions and the date on dirty
                    workdir (default)
    :node-and-timestamp: like ``node-and-date`` but with a timestamp of
                         the form ``{:%Y%m%d%H%M%S}`` instead
    :dirty-tag: adds ``+dirty`` if the current workdir has changes
    :no-local-version: omits local version, useful e.g. because pypi does
                       not support it


Importing in ``setup.py``
~~~~~~~~~~~~~~~~~~~~~~~~~

To support usage in ``setup.py`` passing a callable into ``use_scm_version``
is supported.

Within that callable, ``setuptools_scm`` is available for import.
The callable must return the configuration.


.. code:: python

    # content of setup.py
    import setuptools

    def myversion():
        from setuptools_scm.version import get_local_dirty_tag
        def clean_scheme(version):
            return get_local_dirty_tag(version) if version.dirty else '+clean'

        return {'local_scheme': clean_scheme}

    setup(
        ...,
        use_scm_version=myversion,
        ...
    )


Note on testing non-installed versions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

While the general advice is to test against a installed version,
some environments require a test prior to install,

.. code::

  $ python setup.py egg_info
  $ PYTHONPATH=$PWD:$PWD/src pytest


Interaction with Enterprise Distributions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Some enterprise distributions like RHEL7 and others
ship rather old setuptools versions due to various release management details.

In those case its typically possible to build by using a sdist against ``setuptools_scm<2.0``.
As those old setuptools versions lack sensible types for versions,
modern setuptools_scm is unable to support them sensibly.

In case the project you need to build can not be patched to either use old setuptools_scm,
its still possible to install a more recent version of setuptools in order to handle the build
and/or install the package by using wheels or eggs.



Code of Conduct
---------------

Everyone interacting in the ``setuptools_scm`` project's codebases, issue
trackers, chat rooms, and mailing lists is expected to follow the
`PSF Code of Conduct`_.

.. _PSF Code of Conduct: https://github.com/pypa/.github/blob/main/CODE_OF_CONDUCT.md

Security Contact
================

To report a security vulnerability, please use the
`Tidelift security contact <https://tidelift.com/security>`_.
Tidelift will coordinate the fix and disclosure.
