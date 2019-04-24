Our commit message guidelines match those you'll find in the [git book](https://www.git-scm.com/book/en/v2/Distributed-Git-Contributing-to-a-Project)
and (other great articles)[https://chris.beams.io/posts/git-commit/], so you may be familiar with them.


## Subject Lines

* Write a short title or subject line that summarizes the change.
* Aim for 50 characters or less.
* Write in the imperative. If the title starts "Fix", "Change", "Add", or "Merge" it's probably good.

ℹ️ **Why**: The short title is often the only part of the message displayed in many UIs. It helps if it is fully readable and not truncated. Keeping language consistent makes it easier to read a log.


## Commit body

* Elaborate in further paragraphs as needed, after a blank line.
* Explain why the change is being made, and differences in resulting behaviour
* Hard wrap at 70ish characters
* There's no need to mention any details that are obvious from the other commit details, such as the files changed or who you are.
* There's no need to start writing a post-hoc spec of the change if one already exists. If there's already a ticket or document that explains the context of the change, link to it instead of repeating it.
* If there's some need to explain the actual lines of code you're written, add comments in the source rather than explaining in the commit message.
* Any magic words to trigger integrations (such as auto-resolving a Sentry issue) go at the end. These are optional.

ℹ️ **Why**: There's two main audiences for the commit message: anyone reviewing the code before it is merged, and a future developer tracking down an issue and wondering what the hell you were thinking. They can already see the text of the change you made; what's needed here is the context of what you were intending and why.


## Squashing

Most Zego projects have the convention of squashing a branch to a single commit when it lands, to give the default branch a simple, linear change log of complete, passing changes.

Whether this is done with a commandline rebase or through the GiHub UI, there's a chance to revise the commit message for the squashed commit, rather than accepting all the previous messages concatenated together.

* Make sure you still have a good title that describes the whole change
* GitHub will add the pull request reference to the title, this is good to keep
* Any commit messages about fighting with tests or placating your code reviewer can probably be removed, as they will no longer have the context of the individual diff. In the simplified history, your code lands already in it's perfect state.

ℹ️ **Why**: A clean commmit history is more usable and understandable with cleaned up commit messages.


## Example

For example, see tego commit [abfdb1](https://github.com/Zegocover/tego/commit/abfdb102ec110a8370ebc0710a61890da5bf7fec):

```
commit abfdb102ec110a8370ebc0710a61890da5bf7fec
Author: Henry Cook <HenryCook@users.noreply.github.com>
Date:   Thu Mar 21 14:19:55 2019 +0000

    Collect website logs (#3626)

    * Create and send website logs to ES

    * Send output and errors to one log file for website

    * Better logging time format

    * Multiple outputs are not supported by filebeat

    Removing second output to new ES cluster while we it is being built
    towards being production ready. This is causing filebeat on the website
    instances to fail starting due to broken config:

    https://github.com/elastic/beats/issues/1035
```
