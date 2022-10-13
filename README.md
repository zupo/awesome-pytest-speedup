# Awesome `pytest` speedup

A checklist of potential pitfalls to check when your [pytest](https://pypi.org/project/pytest/) suite is too slow.

* [ ] [Hardware is fast](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#hardware)
* [ ] [Collection is fast](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#collection)
* [ ] [PYTHONDONTWRITEBYTECODE=1 is set](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#pythondontwritebytecode)
* [ ] [Built-in pytest plugins are disabled](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#builtin-plugins)
* [ ] [Only a subset of tests are executed](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#be-picky)
* [ ] [Network access is disabled](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#network-access)
* [ ] [Disk access is disabled](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#disk-access)
* [ ] Database access is optimized
* [ ] Tests run in parallel

With [general guidelines](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#measure-first) and some [extra tips](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#extra-tips).

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

Most people won't benefit a ton from this, but it's an easy thing to try.


## Builtin plugins

Did you know that `pytest` comes with over 30 builtin plugins? You probably don’t need all of them.

* List them with `pytest --trace-config`
* Disable with `pytest -p no:doctest`
* I usually only disable the legacy ones: `-p no:pastebin -p no:nose -p no:doctest`


## Be picky!

Hot take: you don't *always* have to run all tests:

* [`pytest-skip-slow`](https://pypi.org/project/pytest-incremental/) Skip known slow tests by adding the `@pytest.mark.slow` decorator. Do it on local dev and potentially on CI branches. Run all tests on main CI run with `--slow`.
* [`pytest-incremental`](https://pypi.org/project/pytest-incremental/) analyses your codebase & file modifications between test runs to decide which tests need to be run. Useful for local development, but also in CI: only run the full suite after diff-suite is green.
* [`pytest-testmon`](https://pypi.org/project/pytest-testmon/) does the same, but using [a smarter algorithm](https://testmon.org/determining-affected-tests.html) that includes looking into coverage report to decide which tests should run for a certain line change.

# How do you write your tests?

## Network access

Unit tests rarely need to access the Internet. Any network traffic is usually mocked, to be able to test various responses, and to ensure tests don't wait for responses and are fast.

However, often we don't even realize our tests are using the network. Maybe someone added support for loading profile pics from Gravatar and now a bunch of tests are pinging Gravatar API under the hood.

`pytest-socket` is a great plugin to prevent inadvertent Internet access. Straightforward to use and with a bunch of escape hatches for those rare cases when you do in fact need network access in your test code.

## Disk access

Unit tests ideally should not rely on the current filesystem to run successfully. If they do, they are more error prone as the filesystem can change, and are also slower if they write stuff to the disk.

One alternative is [mocking disk i/o calls](https://nickolaskraus.io/articles/how-to-mock-the-built-in-function-open/).

Another is a temporary in-memory filesystem, provided by [`pyfakefs`](https://github.com/pytest-dev/pyfakefs), making your tests both safer and faster.


## Database access

* do everything on every test
* do it once and truncate
* do it once and don't commit()

# Parallelization

## pytest-xdist

## pytest-split


# Bonus karma points

## pytest --lf

## pytest-skip-slow for local dev

## pytest-incremental / pytest-cov-exclude for local and/or branch-only CI

## pytest-instafail

save CPU cycles, save money



## --durations

Pytest comes with a built-in flag `--durations` that prints out the slowest tests. This is a great starting point.

## pytest-monitor / pytest-testmon

## profiling

###

### https://github.com/inconshreveable/sqltap




# Extra tips

## `pytest --lf`

Or [`pytest --last-failed`](https://docs.pytest.org/en/7.1.x/how-to/cache.html?highlight=last%20failed), it tells `pytest` to only run tests that have failed during the last run. Handy in local development to decrease iteration time.

## Handy pytest plugins

There are some pytest plugins that I tend to use in every project. I listed them on https://niteo.co/blog/indispensable-pytest-plugins.


## [Pyramid] config.scan()

If you are using Pyramid's `config.scan()`, that is a potential bottleneck in large codebases. You could speed it up by [telling it to ignore folders with tests](https://medium.com/partoo/speeding-up-tests-with-pytest-and-postgresql-a308b28228fe).

```
config.scan(ignore=[".tests"])
config.scan("myapp", ignore=re.compile("tests?$").search)
```
