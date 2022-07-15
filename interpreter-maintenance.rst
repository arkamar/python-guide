=====================================
Maintenance of Python implementations
=====================================

Notes specific to Python interpreters
=====================================
CPython patchsets
-----------------
Gentoo is maintaining patchsets for all CPython versions.  These include
some non-upstreamable Gentoo patches and upstream backports.  While it
is considered acceptable to add a new patch (e.g. a security bug fix)
to ``files/`` directory, it should be eventually moved into
the respective patchset.

When adding a new version, it is fine to use an old patchset if it
applies cleanly.  If it does not, you should regenerate the patchset
for new version.

The origin for Gentoo patches are the ``gentoo-*`` tags the `Gentoo fork
of CPython repository`_.  The recommended workflow is to clone
the upstream repository, then add Gentoo fork as a remote, e.g.::

    git clone https://github.com/python/cpython
    cd cpython
    git remote add gentoo git@git.gentoo.org:fork/cpython.git
    git fetch --tags gentoo

In order to rebase the patchset, check out the tag corresponding
to the previous patchset version and rebase it against the upstream
release tag::

    git checkout gentoo-3.7.4
    git rebase v3.7.6

You may also add additional changes via ``git cherry-pick``.  Once
the new patches are ready, create the tarball and upload it, then
create the tag and push it::

    mkdir python-gentoo-patches-3.7.6
    cd python-gentoo-patches-3.7.6
    git format-patch v3.7.6
    cd ..
    tar -cf python-gentoo-patches-3.7.6.tar python-gentoo-patches-3.7.6
    xz -9 python-gentoo-patches-3.7.6.tar
    scp python-gentoo-patches-3.7.6.tar.xz ...
    git tag gentoo-3.7.6
    git push --tags gentoo


PyPy
----
Due to high resource requirements and long build time, PyPy on Gentoo
is provided both in source and precompiled form.  This creates a bit
unusual ebuild structure:

- ``dev-python/pypy-exe`` provides the PyPy executable and generated
  files built from source,
- ``dev-python/pypy-exe-bin`` does the same in precompiled binary form,
- ``dev-python/pypy`` combines the above with the common files.  This
  is the package that runs tests and satisfies the PyPy target.

Matching ``dev-python/pypy3*`` exist for PyPy3.

When bumping PyPy, ``pypy-exe`` needs to be updated first.  Then it
should be used to build a binary package and bump ``pypy-exe-bin``.
Technically, ``pypy`` can be bumped after ``pypy-exe`` and used to test
it but it should not be pushed before ``pypy-exe-bin`` is ready, as it
would force all users to switch to source form implicitly.

The binary packages are built using Docker_ nowadays, using
binpkg-docker_ scripts.  To produce them, create a ``local.diff``
containing changes related to PyPy bump and run ``amd64-pypy``
(and/or ``amd64-pypy3``) and ``x86-pypy`` (and/or ``x86-pypy3``) make
targets::

    git clone https://github.com/mgorny/binpkg-docker
    cd binpkg-docker
    (cd ~/git/gentoo && git diff origin) > local.diff
    make amd64-pypy amd64-pypy3 x86-pypy x86-pypy3

The resulting binary packages will be placed in your home directory,
in ``~/binpkg/${arch}/pypy``.  Upload them and use them to bump
``pypy-exe-bin``.


Adding a new Python implementation
==================================
Eclass and profile changes
--------------------------
When adding a new Python target, please remember to perform all
the following tasks:

- add the new target flags to ``profiles/desc/python_targets.desc``
  and ``python_single_target.desc``.

- force the new implementation on ``dev-lang/python-exec``
  via ``profiles/base/package.use.force``.

- mask the new target flags on stable profiles
  via ``profiles/base/use.stable.mask``.

- update ``python-utils-r1.eclass``:

  1. add the implementation to ``_PYTHON_ALL_IMPLS``

  2. update the patterns in ``_python_verify_patterns``

  3. update the patterns in ``_python_set_impls``

  4. update the patterns in ``_python_impl_matches``

  5. add the appropriate dependency to the case for ``PYTHON_PKG_DEP``

- update the tested version range in ``eclass/tests/python-utils-r1.sh``

- add the new implementation to the list
  in ``app-portage/gpyutils/files/implementations.txt``.


Porting initial packages
------------------------
The initial porting is quite hard due to a number of circular
dependencies.  To ease the process, it is recommended to temporarily
limit testing of the packages that feature many additional test
dependencies.  The packages needing this have implementation conditions
in place already.  An example follows:

.. code-block:: bash
   :emphasize-lines: 6,18,23

    PYTHON_TESTED=( python3_{8..10} pypy3 )
    PYTHON_COMPAT=( "${PYTHON_TESTED[@]}" python3_11 )

    BDEPEND="
        test? (
            $(python_gen_cond_dep '
                dev-python/jaraco-envs[${PYTHON_USEDEP}]
                >=dev-python/jaraco-path-3.2.0[${PYTHON_USEDEP}]
                dev-python/mock[${PYTHON_USEDEP}]
                dev-python/pip[${PYTHON_USEDEP}]
                dev-python/sphinx[${PYTHON_USEDEP}]
                dev-python/pytest[${PYTHON_USEDEP}]
                dev-python/pytest-fixture-config[${PYTHON_USEDEP}]
                dev-python/pytest-virtualenv[${PYTHON_USEDEP}]
                dev-python/pytest-xdist[${PYTHON_USEDEP}]
                >=dev-python/virtualenv-20[${PYTHON_USEDEP}]
                dev-python/wheel[${PYTHON_USEDEP}]
            ' "${PYTHON_TESTED[@]}")
        )
    "

    python_test() {
        has "${EPYTHON}" "${PYTHON_TESTED[@]/_/.}" || continue

        HOME="${PWD}" epytest setuptools
    }

It is important to remember to update the implementation range
and therefore enable testing once the test dependencies are ported.
Please do not remove the conditions entirely, as they will be useful
for the next porting round.

If only a non-significant subset of test dependencies is a problem,
it is better to make these dependencies conditional and run
the remainder of the test suite.  If tests are not skipped automatically
due to missing dependencies, using ``has_version`` to skip them
conditionally is preferred over hardcoding version ranges, e.g.:

.. code-block:: bash
   :emphasize-lines: 3-6,12

    BDEPEND="
        test? (
            $(python_gen_cond_dep '
                dev-python/pydantic[${PYTHON_USEDEP}]
            ' pypy3 python3_{8..10}  # TODO: python3_11
            )
        )
    "

    python_test() {
        local EPYTEST_DESELECT=()
        if ! has_version "dev-python/pydantic[${PYTHON_USEDEP}]"; then
            EPYTEST_DESELECT+=(
                tests/test_comparison.py::test_close_to_now_{false,true}
            )
        fi
        epytest
    }

During the initial testing it is acceptable to be more lenient on test
failures, and deselect failing tests on the new implementation when
the package itself works correctly for its reverse dependencies.
For example, during Python 3.11 porting we have deselected a few failing
tests on ``dev-python/attrs`` to unblock porting ``dev-python/pytest``.
Porting pytest in order to enable testing packages was far more
important than getting 100% passing tests on ``dev-python/attrs``.

The modern recommendation for the porting process is to focus
on ``dev-python/pytest`` as the first goal.  It is the most common test
dependency for Python packages, and porting it makes it possible to
start testing packages early.  The initial ported package set should
include all dependencies of pytest, except for test dependencies
of the package with large test dependency graphs (``dev-python/pytest``
itself, ``dev-python/setuptools``).  This amounts to around 40 packages.

Note that emerging the initial set requires installing
``dev-python/pytest`` with ``USE=-test`` first.  Once it is installed,
the previously installed dependencies should be reinstalled with tests
enabled.

After pushing the initial batch, the next recommended goal
is ``dev-python/urllib3``.  It should be followed by focusing
on reenabling tests in the packages where they were skipped.


Python build system bootstrap
=============================
Python build systems are often facing the bootstrap problem — that is,
the build system itself has some dependencies, while these dependencies
require the same build system to build.  The common upstream way
(actually recommended in `PEP 517 build requirements`_ section) is
to bundle the necessary dependencies as part of the build system.
However, this goes against best Gentoo practices.

The current Gentoo practice for bootstrap with dependency unbundling
is to:

1. Install the dependencies of flit_core and the eclass PEP 517 logic
   (installer, tomli) manually using ``python_domodule``.

2. Install flit_core using the standalone PEP 517 backend.

3. Install the dependencies of setuptools using flit (writing trivial
   ``pyproject.toml`` within the ebuild if necessary).

4. Install setuptools using the standalone PEP 517 backend.

5. The dependencies of other build systems can be installed using
   flit, setuptools or other previously unbundled build systems.

Note that for the purpose of bootstrap only obligatory baseline
dependencies are considered significant.  Non-obligatory dependencies
(i.e. ones that can be missing during the bootstrap process) can be
placed in ``PDEPEND``.  Test suite dependencies can include cycles
with the package itself — running tests is not considered obligatory
during the bootstrap process.

flit_core has been chosen as the base build system for unbundling since
it has the fewest external dependencies (i.e. only depends on tomli).
Its author indicates in the `flit_core vendoring README`_ that no other
dependencies will be added or vendored into flit_core.


.. _Gentoo fork of CPython repository:
   https://gitweb.gentoo.org/fork/cpython.git/
.. _Docker: https://www.docker.com/
.. _binpkg-docker: https://github.com/mgorny/binpkg-docker
.. _PEP 517 build requirements:
   https://www.python.org/dev/peps/pep-0517/#build-requirements
.. _flit_core vendoring README:
   https://github.com/pypa/flit/blob/main/flit_core/flit_core/vendor/README
