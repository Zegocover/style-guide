# Python Style Guide

Your goal is to write code that is clean, correct, secure, efficient, consistent, clear to read, beautiful, and easy to review. ~~This needs conscious thought, time and effort to achieve~~ So we use [black](https://github.com/psf/black). These guidelines are one of the tools you have to help with that goal.

These guidelines do not apply to any generated code, such as Django migrations.


## PEP 8

Like many Python projects, we adopt the [PEP 8 style guide](https://www.python.org/dev/peps/pep-0008/).

It's worth reading again: a lot of the advice and guidance in PEP 8 is nuanced and can not easily be checked by tooling.

Here are some of the more commonly overlooked guidelines:

* Prefer to break lines before binary operators (for readability)
* Docstrings use triple double quotes (for consistency)
* Custom Exception types should be named with suffix `Error`, and should derive from `Exception` (for readability)
* Don't use bare `except` clauses. (If you want to catch "all" errors, name `Exception` in the `except` clause) (for correctness)
* Use trailing commas in 1-tuples and collections where each element has its own line (for consistency)
* There is no need for trailing commas where the closing bracket is on the same line; they provide no advantage (for consistency)


## Additional guidelines

We also have additional guidelines that are stricter than PEP 8 or cover additional areas. These are intended to encourage consistency and readability.

### Whitespace

We use a 100 character nominal line length (not 80).

Set your editor or IDE to respect the `.editorconfig` file in the repo. This will configure it to use 4 space indents, automatically trim trailing whitespace, and automatically insert trailing newlines in Python files.

### Syntax

Where Python 2 character encoding declarations or future imports are present in files you are working on, it is fine to remove them.

Prefer writing list, dict and set literals to calling `list`, `dict` or `set`.

Type hints are allowed but not required. If a module is using them consistently, continue to use them.  [PEP 484 type annotations](https://www.python.org/dev/peps/pep-0484/) are preferred to docstring conventions or other comments.

We are not currently enforcing any type linting, so type annotations are mostly documentation, but they are used by some development environments so they should be accurate if used.

Docstrings are for explaining a purpose, API or contract – what it's for, what it does, how to call it, what to expect. `#` comments are for clarifying or explaining the implementation.


### Migrating to Black

As we are moving from a colourful world of competing auto-formatters and personal opinion to one where everything is black, Pull Requests should only contain linting/formatting changes to the files that also contain the logic/feature changes in the actual change. This makes reviewing the pull request much easier and avoid big bang changes to other parts of the code that aren't connected.

### Style

We have no requirement to stay compatible with old versions of Python. Feel free to use, and expect to see, Python features introduced in the 3.x line such as [`f` strings](https://www.python.org/dev/peps/pep-0498/),  [`Path` objects](https://www.python.org/dev/peps/pep-0519/), [keyword-only arguments](https://www.python.org/dev/peps/pep-3102/), [function](https://www.python.org/dev/peps/pep-0484/) and [variable](https://www.python.org/dev/peps/pep-0526/) annotations, and (backported) [data classes](https://www.python.org/dev/peps/pep-0557/) where they are appropriate.

Each import block should be alphabetized by line. If you are unsure as to which block a package belongs in, or the import blocks are very messy, the `isort` tool is configured to automatically fix imports. Using `isort` is not required, however.

Prefer `with` statements to `finally` clauses where a context manager is available.

If a list is alphabetically ordered, keep it that way when adding an item.

Avoid generator expressions, comprehensions or calls to `map` or `reduce` with side effects: commit to either a functional or imperative style, at least for each line of code.

Avoid nested generator expressions or comprehensions (with more than one `for`).

### Naming

Variable names should increase in length with increased scope. Very short names are ok in comprehensions, generator expressions, or tiny loops. Names that are confusable characters or that clash with common `pdb` commands are not allowed [`O`, `I`, `d`, `l`, `n`, `p`, `r`, `s`, `u`]

Boolean fields or properties should have a positive connotation (`enabled`, `active`, `allowed`, etc). Negative names have an implicit `not` that makes boolean expressions harder to follow.

Datetime fields or properties should usually have the suffix `_at`.


### Logging

First things first: **Do not log personal data!**

Logs are not secure, and we use a third party to process them, which means that personal data in logs is an information leak. Personal data includes the obvious things like names, emails, phone numbers, but also includes internal-only identifiers such as the `id` attribute of people or policies (use `customer_number` or `policy_number` respectively). If in doubt, don't log it.

A special case of not logging personal data is not logging any unvalidated input. One example is if you have a customer number input then do not log it until you have confirmed it matches a customer, because the wrong type of data (e.g. a phone number) may have been sent. Another is that you should usually not log responses from third-party services unless you can be sure that they will never contain personal data.

Treat logs as event streams. We use structured logging to make log events queryable; take advantage of this by adding structured data using the `extra` parameter rather than formatting it in the message. Use [standard attribute names](https://docs.datadoghq.com/logs/processing/attributes_naming_convention/) where possible.

```python
# bad
logger.info("Renewed policy for customer %s", customer.customer_number)

# good
logger.info("Renewed policy", extra={
    "usr.id": current_user.customer_number,
    "customer.id": customer.customer_number,
    "evt.name": "policy_renewal",
    "evt.outcome": "renewed"
})
```

The `usr.id` attribute should be used for the currently authenticated user, whereas `customer.id` should be used for the customer concerned. For requests made by the customer themselves these will be the same; for requests from the admin site they will be different as `usr.id` should be the admin user. Note that in the context of a Django request the log formatter will automatically add details of the HTTP request and the authenticated user, and in a Celery task it will add the task name and identifier.

Processing and storing logs costs money, and the costs from excessive logging add up fairly quickly, so only log at `INFO` level or above if the event is something we're likely to want to know about when troubleshooting. Great places to add logging are for errors, warnings, and interesting events, e.g.

- State changes, such as renewing or adjusting a policy
- Security-related events such as authenticating or logging out
- Third-party calls such as taking payment or querying claims data

The goal is that when we need to investigate things that have gone wrong we can see a timeline of all the key events leading up to it, regardless of which service the events originated from.

If you do want more detailed logging for specific situations then do it at `DEBUG` level which is not processed by default, but is stored and can be retrieved if necessary. As a rough guide, if you can't link the log to a key identifier such as customer number or policy number then it should probably either be at `DEBUG` level or a metric instead of a log.

```python
# bad
logger.info("Retrieved %d records", len(records))

# ok
logger.debug("Retrieved %d records", len(records))

# good
statsd.increment("records_retrieved", len(records))
```

Loggers should be declared at the module level as `logger = logging.getLogger(__name__)`. The logger name can be used to filter logs and customise how they are handled.

Log messages should use `%`-style format strings and should not pre-format the message: allow the logger to do the formatting. This helps group log messages correctly when sent to Sentry, and allows formatting to be skipped when loggers are disabled. Note, however, that you probably shouldn't need to do much or any formatting with structured logging.

### Tools & Linting

You should configure your editor or IDE to show you linting warnings. `flake8` or the PyCharm linter are both great. VS Code defaults to a restricted set of `pylint` warnings which are a bit different – you may wish to switch it to use `flake8`.

Avoid changing existing code that you come across to your personal formatting preference if it is otherwise readable. Avoid allowing your editor to auto-format existing code. However, feel free to improve any code that you are changing anyway.

If you come across code that is unacceptably messy and should be brought in line with current guidelines, make formatting changes in a separate branch that changes no logic. This allows your code reviewers to more easily approve both your formatting, layout or structures changes and your logic changes, as they are much clearer in the diff.


## Django

Models, Enum classes and apps should have singular names.

Avoid `select_related` with no parameters. It's performance characteristics can change wildly as the data model evolves.

Transactions should be as short as possible, as open transactions have a performance cost for the database and we can only maintain a certain number of open transactions at any one time. This means that you should prefer using `transaction.atomic` as a context manager (`with transaction.atomic()`) over as a decorator (`@transaction.atomic`), unless your function only does database access calls and you genuinely need transactional behaviour throughout. Avoid calling external services inside a transaction, as the round trip to and from the external service will cause your transaction to stay open for an artificially long time. 

Where the standard Django app modules `models.py`, `forms.py`, `admin.py`, etc get too large to be workable, convert into a package containing multiple modules.


## How to enforce these guidelines

### Automatically

We run `flake8` over python code before tests during our continuous integration suite of checks. Only relatively serious issues, such as undeclared locals, missing imports, mutable defaults, or remaining breakpoints will fail the build.

There's no particular need to mention these in code review, as developers need to pass both CI checks and code review before merging.

### In code review

You should raise all issues that cause readability of code or clarity of intention to suffer.

It's not usually valuable to point out every technical violation where code is otherwise clear and readable. Time spent on clear naming, layout and structure is much more valuable that time spent shuffling whitespace for either a linter or reviewer.
