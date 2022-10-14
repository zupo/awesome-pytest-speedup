# Awesome `pytest` speedup

A checklist of best-practices to speed up your [pytest](https://pypi.org/project/pytest/) suite.

Tick them off, one by one:

* [ ] [Hardware is fast](https://github.com/zupo/awesome-pytest-speedup#hardware)
* [ ] [Collection is fast](https://github.com/zupo/awesome-pytest-speedup#collection)
* [ ] [PYTHONDONTWRITEBYTECODE=1 is set](https://github.com/zupo/awesome-pytest-speedup#pythondontwritebytecode)
* [ ] [Built-in pytest plugins are disabled](https://github.com/zupo/awesome-pytest-speedup#builtin-plugins)
* [ ] [Only a subset of tests are executed](https://github.com/zupo/awesome-pytest-speedup#be-picky)
* [ ] [Network access is disabled](https://github.com/zupo/awesome-pytest-speedup#network-access)
* [ ] [Disk access is disabled](https://github.com/zupo/awesome-pytest-speedup#disk-access)
* [ ] [Database access is optimized](https://github.com/zupo/awesome-pytest-speedup#database-access)
* [ ] [Tests run in parallel](https://github.com/zupo/awesome-pytest-speedup#parallelization)

With [general guidelines](https://github.com/zupo/awesome-pytest-speedup#measure-first) and some [extra tips](https://github.com/zupo/awesome-pytest-speedup#extra-tips).

Let's start!

# Measure first!

Before you do any of the changes below, remember the old *measure twice, optimize once* mantra. Instead of changing some pytest config and blindly believing it will help make your test suite faster, always measure.

In other words:
1. Change one thing.
2. Measure locally and on CI.
3. Commit and push if there is a difference.
4. Leave it running for a few days.
5. Rinse & repeat.

For timing the entire test suite, [`hyperfine`](https://github.com/sharkdp/hyperfine) is a *fantastic* tool.

For timing single tests, `pytest --durations 10` will print out the slowest ten tests.

For measuring CPU usage and memory consumption, look at [`pytest-monitor`](https://pypi.org/project/pytest-monitor/).

For detailed per-function-call profiling of tests, [`pytest-profiling`](https://pypi.org/project/pytest-profiling/) is a great start. Or you can try [using the cProfile module directly](https://maciej.lasyk.info/2016/Dec/14/python-unittest-cprofile-mock/).

Another popular profiler [`pyinstrument`](https://pypi.org/project/pyinstrument/) provides examples [how to use it with `pytest`](https://pyinstrument.readthedocs.io/en/latest/guide.html#profile-pytest-tests).

# How do you run your tests?

## Hardware

Modern laptops are incredibly fast. Why spend hours of time waiting for tests if you can throw money at the problem? Get a faster machine!

Using a CI that gets very expensive as you increase CPU cores? Most of them allow you to use [self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners) for cheap. Buy a few Mac Minis and have them run your tests, using all of their modern fast cores.

## Collection

Collection is the first part of running a test suite: finding all tests that need to be run. It has to be super fast because it happens even if you run a single test.

Use the `-—collect-only` flag to see how fast (or slow) the collection step is. On a modern laptop, the collection should be take around 1s per 1000 tests, maybe up to 2s or 3s per 1000 tests for large codebases.

If it is slower, you can try the following:

* Tell `pytest` not to look into certain folders:

    ```
    # pytest.ini
    [pytest]
    norecursedirs = docs *.egg-info .git .tox var/large_ml_model/
    ```

* Tell `pytest` where exactly the tests are so it doesn't look anywhere else

    ```
    pytest src/my_app/tests
    ```

* Maybe collection is slow because of some code in `contest.py`? Try running `pytest --collect-only --noconftest` to see if there is a difference. Much faster? Then it’s something in `conftest.py` that is causing the slowdown.

* Maybe imports are making collection slow. Try this:
    * `* python -X importtime -m pytest`
    * Paste the output into https://kmichel.github.io/python-importtime-graph/
    * See who the big offenders are
    * Try moving top-level imports into function imports.
    * This is especially likely to be the culprit if your code imports large libraries such as pytorch, opencv, Django, Plone, etc.


## PYTHONDONTWRITEBYTECODE

As per [The Hitchhiker's Guide to Python](https://docs.python-guide.org/writing/gotchas/#disabling-bytecode-pyc-files) there is no need to generate bytecode files on development machines.

On CIs it makes even less sense to do so. And can potentially even slow down the execution of your test suite.

Most people won't benefit a ton from this, but it's an easy thing to try: add `export PYTHONDONTWRITEBYTECODE=1` to your `~/.profile` and your CI configuration.


## Builtin plugins

Did you know that `pytest` comes with over 30 builtin plugins? You probably don’t need all of them.

* List them with `pytest --trace-config`
* Disable with `pytest -p no:doctest`

There is not much speedup to gain here, so I usually only disable the legacy ones: `-p no:pastebin -p no:nose -p no:doctest`


## Be picky!

Hot take: you don't *always* have to run all tests:

* [`pytest-skip-slow`](https://pypi.org/project/pytest-skip-slow/) Skip known slow tests by adding the `@pytest.mark.slow` decorator. Do it on local dev and potentially in CI branch runs. Run all tests in main CI run with `--slow`.
* [`pytest-incremental`](https://pypi.org/project/pytest-incremental/) analyses your codebase & file modifications between test runs to decide which tests need to be run. Useful for local development, but also in CI: only run the full suite after diff-suite is green, and save some CPU minutes.
* [`pytest-testmon`](https://pypi.org/project/pytest-testmon/) does the same, but using [a smarter algorithm](https://testmon.org/determining-affected-tests.html) that includes looking into coverage report to decide which tests should run for a certain line change.

# How do you write your tests?

## Network access

Unit tests rarely need to access the Internet. Any network traffic is usually [mocked](https://realpython.com/testing-third-party-apis-with-mocks/), to be able to test various [responses](https://pypi.org/project/responses/), and to ensure tests are fasts by not having to wait for responses.

However, often we don't even realize that the code being tests is using the network. Maybe someone added support for loading profile pics from Gravatar and now a bunch of tests are pinging Gravatar API under the hood.

[`pytest-socket`](https://pypi.org/project/pytest-socket/) is a great plugin to prevent inadvertent Internet access. Straightforward to use and with a bunch of escape hatches for those rare cases when you do in fact need network access in your test code.

## Disk access

Unit tests ideally should not rely on the current filesystem to run successfully. If they do, they are more error prone as the filesystem can change, and are also slower if they write stuff to the disk.

One alternative is [mocking disk i/o calls](https://nickolaskraus.io/articles/how-to-mock-the-built-in-function-open/).

Another is a temporary in-memory filesystem, provided by [`pyfakefs`](https://github.com/pytest-dev/pyfakefs), making your tests both safer and faster.

## Database access

Testing web apps usually requires some database access in tests, and that's OK. There are still several optimizations possible.

### Do all tests need a database?

Some tests could potentially be rewritten in a way that does not require a database to do the testing. For example, if calculating a user's age, there is no need to fetch a birth date from a DB. It's better to provide a fake birth date to the calculation function, and remove the test's dependency on the database fixture.

### Do all tests need the entire database?

Populating the database with test data takes time. Do all tests need the entire dummy dataset? If a group of test is testing some user profile feature, they potentially don't need to populate non-relevant tables in the database.

### Only prepare the database once

Preparing, using and then destroying the dummy database for every test is wasteful. There are better approaches.

#### Truncate

Instead of destroying the database after a test run, rather [`TRUNCATE` its tables](https://www.niklas-meinzer.de/post/2019-07_pytest-performance/#database-setup-and-tear-down). This saves your from re-creating the database for the next test.

1. Create your test database, once.
2. Populate it with data for the test.
3. Execute a test over it.
4. `TRUNCATE` your tables, i.e. fast remove all data.
5. Populate with data for next test and run the test, rinse & repeat.


#### Rollback

But data population is slow too! Can we save/cache that as well? We sure can! Instead of `TRUNCATE`-ing all tables, we could just rollback the transaction the test used. And such, no data needs to be prepared for the next test, just run it.

1. Create your test database, once.
2. Populate it with dummy data, once.
3. Execute a test over it, making sure the test does not commit anything.
4. Rollback the transaction.

Note that this approach does require you to be a bit more careful when writing tests, as they shouldn't do any database commits. If they do, you need to manually revert their changes.


# Parallelization

By default, pytest uses a single CPU core. Your laptop likely has multiple cores. CI runners also come with multiple cores. It's just a waste of everyone's time not to use them all!


## pytest-xdist

The most popular tool to help you use all cores is [`pytest-xdist`](https://pypi.org/project/pytest-xdist). It supports running across multiple cores, multiple CPUs, even on remote machines!

It usually doesn't work out-of-the-box in a real-world project with complex fixtures, databases involved, etc. The main reason is that `session`-scoped fixtures run on all workers, not just once. There are [a couple of workarounds](https://pypi.org/project/pytest-xdist/#making-session-scoped-fixtures-execute-only-once), but they are not trivial.

For example, you can create a separate database for each `pytext-xdist` worker process and use the worker name/number as a suffix. [`pytest-django`](https://pytest-django.readthedocs.io/en/latest/database.html#use-the-same-database-for-all-xdist-processes) does this by default.

## pytest-split

Compared to `pytest-xdist`, `pytest-split` is easier to use. It does not really help with local development, but can greatly decrease the speed of your CI runs, without much or any changes to your tests.

The way `pytest-split` works is that it splits the test suite to equally sized sub-suites based on test execution time. These sub-suites can be run in parallel in as many CI workers as your budget allows.

Caveats:
* To ensure 100% test coverage, you need to run an additional CI runner after all test runners have finished, that [collects and merges coverage reports](https://github.com/jerry-git/pytest-split-gh-actions-demo/blob/4f1331565e1c31aed536e1e0c2fcedf6656fb09a/.github/workflows/test.yml#L29).
* `pytest-randomly` [seed needs to be pre-computed](https://jerry-git.github.io/pytest-split/#interactions-with-other-pytest-plugins) and fed into all CI runners.


# Extra tips

## Keep 'em fast!

It is really annoying to invest time into speeding up your test suite, only to come back to the codebase a few months later to find out that the tests are slow again. No more! Check out BlueRacer.io, a simple GitHub App that block a Pull Request from merging if tests have become slower: https://github.com/apps/blueracer-io

## `pytest --lf`

Or [`pytest --last-failed`](https://docs.pytest.org/en/7.1.x/how-to/cache.html?highlight=last%20failed), it tells `pytest` to only run tests that have failed during the last run. Handy in local development to decrease iteration time.

## Handy pytest plugins

There are some pytest plugins that I tend to use in every project. I listed them on https://niteo.co/blog/indispensable-pytest-plugins.


## config.scan()

If you are using Pyramid's [`config.scan()`](https://github.com/teamniteo/pyramid-realworld-example-app/blob/f1fed0d0592b1c6e9a4feb6b162cffd4605f5f44/src/conduit/__init__.py#L57), that is a potential bottleneck in large codebases. You could speed it up by [telling it to ignore folders with tests](https://medium.com/partoo/speeding-up-tests-with-pytest-and-postgresql-a308b28228fe).

```
config.scan(ignore=[".tests"])
config.scan("myapp", ignore=re.compile("tests?$").search)
```
