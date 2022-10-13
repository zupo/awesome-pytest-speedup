# Awesome `pytest` speedup

A checklist of things to check and try to speed up your [pytest](https://pypi.org/project/pytest/) suite.

TL;DR:

* [ ] Collection is fast
* [ ] Hardware is fast
* [ ] PYTHONDONTWRITEBYTECODE=1 is set
* [ ] Built-in pytest plugins are disabled
* [ ] [werkzeug/pyramid only] config.scan() is restrained
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

## Collection

## Better hardware

## Split into smaller packages

## PYTHONDONTWRITEBYTECODE

## Disable plugins you don’t need

## config.scan()


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
