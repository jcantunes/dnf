..
  Copyright (C) 2014-2018 Red Hat, Inc.

  This copyrighted material is made available to anyone wishing to use,
  modify, copy, or redistribute it subject to the terms and conditions of
  the GNU General Public License v.2, or (at your option) any later version.
  This program is distributed in the hope that it will be useful, but WITHOUT
  ANY WARRANTY expressed or implied, including the implied warranties of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General
  Public License for more details.  You should have received a copy of the
  GNU General Public License along with this program; if not, write to the
  Free Software Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA
  02110-1301, USA.  Any Red Hat trademarks that are incorporated in the
  source code or documentation are not subject to the GNU General Public
  License and may only be used or replicated with the express permission of
  Red Hat, Inc.

======================
 Queries and Subjects
======================

.. module:: dnf.query

.. class:: Query

  Facilitates lookup of packages in a :class:`~dnf.sack.Sack` based on given criteria. Query actually does not consult the information in the :class:`~!dnf.sack.Sack` until it is evaluated. The evaluation happens either explicitly using :meth:`~dnf.query.Query.run` or by iterating the query, for example::

    #!/usr/bin/python3
    import dnf

    base = dnf.Base()
    base.fill_sack()

    q = base.sack.query()
    i = q.installed()
    i = i.filter(name='dnf')

    packages = list(i)  # i only gets evaluated here
    print("Installed dnf package:")
    for pkg in packages:
        print(pkg, pkg.reponame)

  or::

    #!/usr/bin/python3
    import dnf

    base = dnf.Base()
    base.read_all_repos()
    base.fill_sack()

    q = base.sack.query()
    a = q.available()
    a = a.filter(name='dnf')

    print("Available dnf packages:")
    for pkg in a:  # a only gets evaluated here
        print('{} in repo {}'.format(pkg, pkg.reponame))


  Notice that none of the filtering methods mutates the state of the :class:`~dnf.query.Query` but produces a new object instead.

  .. method:: available()

    Returns a new query limiting the original query to the packages available from the repositories.

  .. method:: difference(other)

    Returns a new query that contains only those results of original query that are not in the results of the ``other`` query.

  .. method:: downgrades()

    Returns a new query that limits the result only to packages that can be downgrade candidates to other packages in the current set. Downgrade candidate has the same name, lower EVR and the architecture of the original and the downgrade candidate are suitable for a downgrade. Specifically, the filtering does not take any steps to establish that the downgrade candidate can actually be installed.

  .. method:: duplicated()

    Returns a new query that limits the result only to installed packages of same name and different version. Optional argument exclude accepts a list of package names that will be excluded from result.

  .. method:: extras()

    Returns a new query that limits the result to installed packages that are not present in any repo

  .. method:: filter(\*\*kwargs)

    Returns a new query limiting the original query to the key/value pairs from `kwargs`. Multiple `kwargs` can be passed, the filter then works by applying all of them together (logical AND). Values inside of list or query are cumulative (logical OR).

    Allowed keys are:

    ===============   ============== ======================================================
    key               value type     value meaning
    ===============   ============== ======================================================
    arch              string         match against packages' architecture
    downgrades        boolean        see :meth:`downgrades`. Defaults to ``False``.
    empty             boolean        ``True`` limits to empty result set.
                                     Defaults to ``False``.
    epoch             integer        match against packages' epoch.
    file              string         match against packages' files
    latest            integer        limit to all packages of number of versions
    latest_per_arch   integer        see :meth:`latest`.
    name              string         match against packages' names
    release           string         match against packages' releases
    reponame          string         match against packages repositories' names
    version           string         match against packages' versions
    pkg               Query          match against packages in query
    pkg*              list           match against hawkey.Packages in list
    provides          string         match against packages' provides
    provides*         Hawkey.Reldep  match against packages' provides
    <DEP>             string         match against packages' <DEP>
    <DEP>*            Hawkey.Reldep  match a reldep against packages' <DEP>
    <DEP>*            Query          match the result of a query against packages' <DEP>
    <DEP>*            list(Package)  match the list of hawkey.Packages against packages' <DEP>
    sourcerpm         string         match against packages' source rpm
    upgrades          boolean        see :meth:`upgrades`. Defaults to ``False``.
    ===============   ============== ======================================================

    ``<DEP>`` can be any of: requires, conflicts, obsoletes, enhances, recomments, suggests, supplements

    \* The key can also accept a list of values with specified type.

    The key name can be supplemented with a relation-specifying suffix, separated by ``__``:

    ==========   =========== ==========================================================
    key suffix   value type  semantics
    ==========   =========== ==========================================================
    eq           any         exact match; This is the default if no suffix is specified.
    glob         string      shell-style wildcard match
    gt           integer     the actual value is greater than specified
    gte          integer     the actual value is greater than or equal to specified
    lt           integer     the actual value is less than specified
    lte          integer     the actual value is less than or equal to specified
    neq          any         does not equal
    substr       string      the specified value is contained in the actual value
    ==========   =========== ==========================================================

    For example, the following creates a query that matches all packages containing the string "club" in its name::

      q = base.sack.query().filter(name__substr="club")

    Note that using packages or queries for dependency filtering performs a more advanced resolution than using a string or a reldep. When a package list or a query is used, rich dependencies are resolved in a more precise way than what is possible when a string or a reldep is used.

  .. method:: filterm(\*\*kwargs)

    Similar to :meth:`dnf.query.Query.filter` but it modifies the query in place.

  .. method:: installed()

    Returns a new query that limits the result to the installed packages only.

  .. method:: intersection(other)

    Returns a new query where the result contains only packages that are found in both original and ``other`` queries.

  .. method:: latest(limit=1)

    Returns a new query that limits the result to ``limit`` highest version of packages per package
    name and per architecture. In case the limit is negative number, it excludes the number of
    latest versions according to limit.

  .. method:: run()

    Evaluate the query. Returns a list of matching :class:`dnf.package.Package` instances.

  .. method:: union(other)

    Returns a new query where the results of the ``other`` query are added to the results of the original query.

  .. method:: upgrades()

    Returns a new query that limits the result only to packages that can be upgrade candidates to at least one package in the current set. Upgrade candidate has the same name, higher EVR and the architectures of the original and the upgrade candidate package are suitable for an upgrade. Specifically, the filtering does not take any steps to establish that the upgrade candidate can actually be installed.

.. module:: dnf.subject

.. class:: Subject

  As :ref:`explained on the DNF man page <specifying_packages-label>`, users of the CLI are able to select packages for an operation in different formats, leaving seemingly arbitrary parts out of the spec and even using globbing characters. This class implements a common approach to parsing such input and produce a :class:`~dnf.query.Query` listing all packages matching the input or a :class:`~dnf.selector.Selector` selecting a single package that best matches the input given a transaction operation.

  .. method:: __init__(pkg_spec, ignore_case=False)

    Initialize the :class:`Subject` with `pkg_spec` input string with following :ref:`semantic <specifying_packages-label>`. If `ignore_case` is ``True`` ignore the case of characters in `pkg_spec`.

  .. method:: get_best_query(sack, with_nevra=True, with_provides=True, with_filenames=True, forms=None)

    Returns a :class:`~Query` yielding packages matching the given input. The result of the returned
    query can be an empty set if no package matches. `sack` is the :class:`~dnf.sack.Sack` that the
    returned query will search. `with_nevra` enable search by nevra, `with_provides` indicates
    whether besides package names also packages' provides are searched for a match, and
    `with_filenames` indicates whether besides package provides also packages' file provides are
    searched for a match. `forms` is a list of pattern forms from `hawkey`_. Leaving the parameter
    to ``None`` results in using a reasonable default list of forms.

  .. method:: get_best_selector(sack, forms=None, obsoletes=True, reponame=None)

    Returns a :class:`~dnf.selector.Selector` that will select a single best-matching package when
    used in a transaction operation. `sack` and `forms` have the same meaning as in
    :meth:`get_best_query`. If ``obsoletes``, selector will also contain packages that obsoletes
    requested packages (default is True). If ``reponame``, the selection of available packages is
    limited to packages from that repo (default is None).

  .. method:: get_nevra_possibilities(self, forms=None)

    Returns generator for every possible nevra. Each possible nevra is represented by NEVRA class
    (libdnf) that has attributes name, epoch, version, release, arch. `forms` have the same
    meaning as in :meth:`get_best_query`.

    Example how to use it when it is known that string could be full NEVRA or NEVR::

        #!/usr/bin/python3
        import dnf
        import hawkey

        nevra_string = "dnf-0:4.2.2-2.fc30.noarch"
        subject = dnf.subject.Subject(nevra_string)
        possible_nevra = subject.get_nevra_possibilities(
            forms=[hawkey.FORM_NEVRA, hawkey.FORM_NEVR])

        for i,nevra in enumerate(possible_nevra):
            print("Possibility {} for \"{}\":".format(i+1, nevra_string))
            print("name: {}".format(nevra.name))
            print("epoch: {}".format(nevra.epoch))
            print("version: {}".format(nevra.version))
            print("release: {}".format(nevra.release))
            print("architecture: {}".format(nevra.arch))
            print()
