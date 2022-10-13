# Awesome `pytest` speedup

A checklist of things to check and try to speed up your [pytest](TODO) suite.

* [ ] [Hardware is fast](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#hardware)
* [ ] [Collection is fast](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#collection)
* [ ] [PYTHONDONTWRITEBYTECODE=1 is set](https://github.com/zupo/awesome-pytest-speedup/blob/main/README.md#pythondontwritebytecode)   
* [ ] Built-in pytest plugins are disabled
* [ ] Network access is disabled
* [ ] Disk access is disabled
* [ ] Database access is optimized
* [ ] Tests run in parallel


# Measure first!

Before you do any of the changes below, remember the old *measure twice, optimize once* mantra. Instead of changing some pytest config and blindly believing it will help make your test suite faster, always measure.

In other words:
1. Change one thing.
2. Measure locally and on CI.
3. Commit and push if there is a difference.
4. Leave it running for a few days.
5. Rinse & repeat.

For timing CLI commands, [`hyperfine`](https://github.com/sharkdp/hyperfine) is *fantastic*.

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


## Disable plugins you don’t need

Did you know that `pytest` comes with over 30 builtin plugins. You probably don’t need all of them.

* List them with `pytest --trace-config`
* Disable with `pytest -p no:doctest`
* I usually only disable the legacy ones: `-p no:pastebin -p no:nose -p no:doctest`


## --durations

Pytest comes with a built-in flag `--durations` that prints out the slowest tests. This is a great starting point.

## pytest-monitor / pytest-testmon

## profiling

###

### https://github.com/inconshreveable/sqltap


# How do you write your tests?

## Network access

- gravatar

## Disk access


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



# Bonus karma points

## [Pyramid] config.scan()

If you are using Pyramid's `config.scan()`, that is a potential bottleneck in large codebases. You could speed it up by [telling it to ignore folders with tests](https://medium.com/partoo/speeding-up-tests-with-pytest-and-postgresql-a308b28228fe).

```
config.scan(ignore=[".tests"])
config.scan("myapp", ignore=re.compile("tests?$").search)
```
