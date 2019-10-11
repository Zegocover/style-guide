# Python Style Guide

Your goal is to write code that is clean, correct, secure, efficient, consistent, clear to read, beautiful, and easy to review. This needs conscious thought, time and effort to achieve. These guidelines are one of the tools you have to help with that goal.

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

Loggers should be declared at the module level as `logger = logging.getLogger(__name__)`

Log messages should use `%`-style format strings and should not pre-format the message: allow the logger to do the formatting. This helps group log messages correctly when sent to Sentry, and allows formatting to be skipped when loggers are disabled.


### Tools & Linting

You should configure your editor or IDE to show you linting warnings. `flake8` or the PyCharm linter are both great. VS Code defaults to a restricted set of `pylint` warnings which are a bit different – you may wish to switch it to use `flake8`.

Avoid changing existing code that you come across to your personal formatting preference if it is otherwise readable. Avoid allowing your editor to auto-format existing code. However, feel free to improve any code that you are changing anyway.

If you come across code that is unacceptably messy and should be brought in line with current guidelines, make formatting changes in a separate branch that changes no logic. This allows your code reviewers to more easily approve both your formatting, layout or structures changes and your logic changes, as they are much clearer in the diff.


## Django

Models, Enum classes and apps should have singular names.

Avoid `select_related` with no parameters. It's performance characteristics can change wildly as the data model evolves.

Where the standard Django app modules `models.py`, `forms.py`, `admin.py`, etc get too large to be workable, convert into a package containing multiple modules.


## How to enforce these guidelines

### Automatically

We run `flake8` over python code before tests during our continuous integration suite of checks. Only relatively serious issues, such as undeclared locals, missing imports, mutable defaults, or remaining breakpoints will fail the build.

There's no particular need to mention these in code review, as developers need to pass both CI checks and code review before merging.

### In code review

You should raise all issues that cause readability of code or clarity of intention to suffer.

It's not usually valuable to point out every technical violation where code is otherwise clear and readable. Time spent on clear naming, layout and structure is much more valuable that time spent shuffling whitespace for either a linter or reviewer.
