# Configuration

Var only does what it's told, so to get started you must create a configuration
file with the instructions you want.

## Table of contents

* [Config file](#config-file)
* [Rulesets](#rulesets)
* [Rules](#rules)
* [Events](#events)
* [Variables](#variables)
* [Conditions](#conditions)
* [Actions](#actions)
* [Attachments](#attachments)

## Config file

When Var receives an incoming event via GitHub's webhook, which is set up when
you connect a repository, the first thing it does it try to load your config
file.

For now, this means creating a `.varci.yml` at the root-level of your
repository, in your default branch (e.g. `master`).

For example, for a GitHub username or organisation name called `my-org` and a
repository called `repo-name`, Var would request the following file:

    https://raw.githubusercontent.com/my-org/repo-name/master/.varci.yml

> Support coming soon for default config files (at the org or user level) so you
> don't need the duplicate your `.varci.yml` file in every repository.

### Example

```yaml
filters:
  short_description:
    name: "Close issues with short descriptions"
    events: [ issues ]
    close: true
    label: invalid
    when: length(body) < 50
    message: >
      @{{ user.login }}, this issue has been closed because it was too short.
      Please use the template provided when submitting issues.
```

## Rulesets

Your config file's ruleset represents the collection of rules you want
performed.

### Anatomy of a ruleset

Your ruleset is declared by the `ruleset:` key at the top-level of your config
file.

It can contain one or more [Rules](#rules) that can be identified by their key.

```yaml
ruleset:
  rule_a:
    ...
  rule_b:
    ...
```

## Rules

Each rule represents one or more actions you would like Var to perform, and the
conditions under which to perform them.

### Anatomy of a rule

A rule has the following required keys:

- `name`: A human readable name for the filter.
- [`events`](#events): An array of GitHub events the filter matches on (e.g.
    `[ issues, pull_request ]`).
- [`conditions`](#conditions): A string/array of conditions that an incoming
    event must satisfy before the actions are performed.
- There is also a requirement that a rule should contain one or more
    [Actions](#actions).

#### Example

```yaml
ruleset:
  rule_a:
    name: "My first rule"
    events: [ issues ]
    when: length(body) < 50
    ...
```

## Events

When users perform actions on your repositories, GitHub sends event data to
Var's webhook URL for processing (the webhook is set up for you when connecting
your repository).

Currently we support the following GitHub events:

- [`issues`](https://developer.github.com/v3/activity/events/types/#issuesevent)
- [`pull_request`](https://developer.github.com/v3/activity/events/types/#pullrequestevent)

## Variables

Before we get started with explaining conditions, it is useful to know how
variable access works within Var configs.

In the following config example:

```yaml
ruleset:
  rule_a:
    ...
    when: issue.title contains "README"
```

- `when:` denotes the start of the condition string.
- `issue.title contains "README"` is the condition:
   - `issue.title` is a variable
   - `contains` is an inline operator
   - `"README"` is the string to match

You can access variables nested deep in JSON objects by using dot-notation. In
the example `issue.title` means the `title` field inside the `issue` object.

You should refer to the GitHub documentation to know what variables are
available to you in the incoming event data.

If you check the [webhook payload example](https://developer.github.com/v3/activity/events/types/#webhook-payload-example-8)
for the event named `issues`, you will see that there is a field located at
`issue.title`:

```json
{
  "action": "opened",
  "issue": {
    "title": "Spelling error in the README file"
  }
}
```

Now, astute readers might notice that accessing `issue.*` variables means that
the the same rule can not be used for `pull_request` events since the same
variables there are accessed using `pull_request.*`.

As such, the "subject" of an event is also made available at the root-level, and
is the recommended way of accessing such variables:

```json
{
  "action": "opened",
  "title": "Spelling error in the README file",
  "issue": {
    "title": "Spelling error in the README file"
  }
}
```

So rather than using `issue.title` or `pull_request.title`, you should simply
refer to it as `title`. (The same applies for all `issue.*` and `pull_request.*`
variables).

The above example rule can then be made portable between issues and pull
requests by rewriting it like so:

```yaml
ruleset:
  rule_a:
    when: title contains "README"
```

## Conditions

A `conditions:` key is required for all rules (or if you prefer, you can use
`when:`).

The value of the `conditions:` key can be on a single line (string) or over
multiple lines (array).

When writing conditions over multiple lines, each line is joined together with
`AND`.

### Example

In the following example, the conditions become the string
`action = "opened" AND length(body) < 50`:

```yaml
ruleset:
  rule_a:
    name: "My first rule"
    events: [ issues ]
    when: length(body) < 50
    when:
      - action = "opened"
      - "README.md" in files
      - length(body) < 50
```

The value of the `conditions:` key is always `true`.

### Operators

Within conditions, you can use the following operators on the event data:

#### Filter

The `filter()` operator supports all expression, and matcher components of
[filter path syntax](#filter-path-syntax). You can use filter to retrieve data
from arrays, along arbitrary paths quickly without having to loop through the
data structures. Instead you use path expressions to qualify which elements you
want returned.

##### Filter path syntax

A path expression is made of any number of tokens. Tokens are composed of two
groups. Expressions, are used to traverse the array data, while matchers are
used to qualify elements. You apply matchers to expression elements.

###### Expression types

| Expression | Definition                                                                        |
|------------|-----------------------------------------------------------------------------------|
| {n}        | Represents a numeric key. Will match any string or numeric key.                   |
| {s}        | Represents a string. Will match any string value including numeric string values. |
| Foo        | Matches keys with the exact same value.                                           |

###### Attribute matching types

| Matcher      | Definition                                                                  |
|--------------|-----------------------------------------------------------------------------|
| [id]         | Match elements with a given array key.                                      |
| [id=2]       | Match elements with id equal to 2.                                          |
| [id!=2]      | Match elements with id not equal to 2.                                      |
| [id>2]       | Match elements with id greater than 2.                                      |
| [id>=2]      | Match elements with id greater than or equal to 2.                          |
| [id<2]       | Match elements with id less than 2                                          |
| [id<=2]      | Match elements with id less than or equal to 2.                             |
| [text=/.../] | Match elements that have values matching the regular expression inside .... |

###### Common usage

Given the following data:

```json
{
  "users": [
    {"id": 1, "name": "mark"},
    {"id": 2, "name": "jane"},
    {"id": 3, "name": "sally"},
    {"id": 4, "name": "jose"}
  ]
}
```

... using the `filter()` operator, you can extract the names of all users:

```
filter(users, "name") => [ "mark", "jane", "sally", "jose" ]
```

###### Example condition

Given the following event data:

```json
{
  "action": "created",
  "issue": {
    "labels": [
      {
        "url": "https://api.github.com/repos/baxterthehacker/public-repo/labels/bug",
        "name": "bug",
        "color": "fc2929"
      }
    ]
  }
}
```

... the following conditions would resolve to `true`:

```yaml
ruleset:
  rule_a:
    when: filter(labels, "name") has "bug"
```

#### Length

The `length()` operator is equivalent to the Javascript [`string.length`
property](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/length).

###### Example condition

Given the following event data:

```json
{
  "action": "created",
  "issue": {
    "body": "I'm less than 50 characters"
  }
}
```

... the following conditions would resolve to `true`:

```yaml
ruleset:
  rule_a:
    when: length(body) < 50
```

#### Count

The `count()` operator is equivalent to the Javascript [`array.length`
property](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/length).

###### Example condition

Given the following event data:

```json
{
  "action": "created",
  "issue": {
    "labels": [
      {
        "url": "https://api.github.com/repos/baxterthehacker/public-repo/labels/bug",
        "name": "bug",
        "color": "fc2929"
      }
    ]
  }
}
```

... the following conditions would resolve to `true`:

```yaml
ruleset:
  rule_a:
    when: count(labels) = 1
```

#### Sum

The `sum()` operator is equivalent to the JavaScript code:

```javascript
arg.reduce(function (a, b) { return a + b });
```

### Inline operators

The following

| Operator       | JavaScript/jQuery equivalent |
|----------------|------------------------------|
| `A and B`      | `(A && B)`                   |
| `A or B`       | `(A || B)`                   |
| `not A`        | `!(A)`                       |
| `A = B`        | `A == B`                     |
| `A is B`       | `A === B`                    |
| `A != B`       | `A != B`                     |
| `A > B`        | `A > B`                      |
| `A >= B`       | `A >= B`                     |
| `A < B`        | `A < B`                      |
| `A <= B`       | `A <= B`                     |
| `A in B`       | `$.inArray(A, B)`            |
| `A contains B` | `A.indexOf(B) !== -1`        |
| `A has B`      | `$.inArray(B, A)`            |
| `A matches B`  | `A.match(B) !== null`        |

#### Contains

The `contains` inline operator checks if a string contains another string.

###### Example condition

Given the following event data:

```json
{
  "action": "created",
  "issue": {
    "body": "Issue type:\n- [x] bug\n- [x] enhancement\n- [x] feature-discussion (RFC)"
  }
}
```

... the following conditions would resolve to `true`:

```yaml
ruleset:
  rule_a:
    when: body contains "[x] bug"
```

#### Has

The `has` inline operator checks if an array contains a string.

###### Example condition

Given the following event data:

```json
{
  "action": "created",
  "issue": {
    "labels": [
      {
        "url": "https://api.github.com/repos/baxterthehacker/public-repo/labels/bug",
        "name": "bug",
        "color": "fc2929"
      }
    ]
  }
}
```

... the following conditions would resolve to `true`:

```yaml
ruleset:
  rule_a:
    when: filter(labels, "name") has "bug"
```

#### Matches

The `matches` inline operator checks if a string matches a pattern.

```yaml
ruleset:
  rule_a:
    when: not (body matches "/[0-9]{1,2}\.[0-9]+(\.[0-9]+)?/")
```

## Actions

The following actions are supported:

### Label

By adding a `label:` key to a rule, Var can automatically label issues under
certain conditions.

The value of the `label:` key can be a single label name (string) or multiple
label names (array).

Any label prefixed with a minus sign will be removed from the current labels of
an issue/pull request.

#### Example

In the following example, the `invalid` label will be added and the `valid`
label will be removed, if the length of body is less than 50 characters:

```yaml
ruleset:
  rule_a:
    name: "My first rule"
    events: [ issues ]
    when: length(body) < 50
    label: [ invalid, -valid ]
```

### Close

By adding a `close:` key to a rule, Var can automatically close issues and pull
requests under certain conditions.

The value of the `close:` key is always `true`.

#### Example

In the following example, an issue will be closed if the length of body is less
than 50 characters:

```yaml
ruleset:
  rule_a:
    name: "My first rule"
    events: [ issues ]
    when: length(body) < 50
    close: true
```

### Reopen

By adding a `reopen:` key to a rule, Var can automatically reopen issues and
pull requests under certain conditions.

The value of the `reopen:` key is always `true`.

#### Example

In the following example, an issue will be reopened if the length of body is
greater than 50 characters:

```yaml
ruleset:
  rule_a:
    name: "My first rule"
    events: [ issues ]
    when: length(body) > 50
    reopen: true
```

### Comment

By adding a `comment:` key to a rule, Var can automatically comment on issues
and pull requests under certain conditions.

This action will be performed by the GitHub user:
[VarCI-bot](https://github.com/VarCI-bot)

The value of the `comment:` key is a string with the comment body you want the
bot to post, and supports [Placeholders](#placeholders).

#### Placeholders

You can insert event data into comments by using placeholders. Placeholders
reference keys in the event data and are surrounded by double curly brackets
(e.g. if `bob` opens a pull request, then `{{ user.login }}` resolves to `bob`,
and if you want to mention the user `@{{ user.login }}` will resolve to `@bob`).

#### Example

In the following example, if a user called `contributor` opens an issue where
the length of body is less than 50 characters, `VarCI-bot` will comment with the
message:

> @contributor, the issue description is too short. Please reopen it once you
> have amended the description to contain more than the requirement of 50
> characters.

```yaml
ruleset:
  rule_a:
    name: "My first rule"
    events: [ issues ]
    when: length(body) < 50
    reopen: true
    message: >
      @{{ user.login }}, the issue description is too short. Please reopen it
      once you have amended the description to contain more than the requirement
      of 50 characters.
```

### Merge

By adding a `merge:` key to a rule, Var can automatically merge pull requests
under certain conditions.

The value of the `merge:` key is always `true`.

#### Example

In the following example, a pull request will be automatically merged if the
only file that was changed was the `.varci.yml` file:

```yaml
ruleset:
  rule_a:
    name: "My first rule"
    events: [ issues ]
    when:
      - "README.md" in files
      - length(body) < 50
    merge: true
```

## Attachments

To give you more power, Var has the ability to attach extra data to incoming
event data from GitHub. This extra data is only requested if a rule tries to
access that data.

### Diff attachment

On pull request events, if a rule's conditions try to access any `diff` (or
`pull_request.diff`) keys, Var will populate this with a list of diff
information obtained by making a separate request to GitHub's pull request API.

#### Example

Given the following event data:

```json
{
  "action": "opened",
  "pull_request": {}
}
```

... when the event data after files are attached becomes:

```json
{
  "action": "opened",
  "pull_request": {
    "diff": {
        "added": [
            "Nobody reads"
        ],
        "removed": [
            "# Project name"
        ],
        "unchanged": []
    },
    "diff_raw": "diff --git a/README.md b/README.md\nindex f247396..93a9c32 100644\n--- a/README.md\n+++ b/README.md\n@@ -1 +1 @@\n-# Project name\n+Nobody reads\n\ No newline at end of file",
  }
}
```

... the following conditions would resolve to `true`:

```yaml
ruleset:
  rule_a:
    when: diff.removed has "# Project name"
```

> Hint: it's better to use the existing counts for `additions`, `deletion` and
> `changed_files` rather than performing a `count(diff.added)` as no extra API
> calls will be required.

### File attachment

On pull request events, if a rule's conditions try to access the `files` (or
`pull_request.files`) key, Var will populate this with a list of file names
obtained by making a separate request to GitHub's pull request API.

#### Example

Given the following event data:

```json
{
  "action": "opened",
  "pull_request": {}
}
```

... when the event data after files are attached becomes:

```json
{
  "action": "opened",
  "pull_request": {
    "files": [
      "README.md",
      "CONTRIBUTING.md"
    ]
  }
}
```

... the following conditions would resolve to `true`:

```yaml
ruleset:
  rule_a:
    when: files has "README.md"
```

### Link checker attachment

On `issues` or `pull_request` events, if a rule's conditions try to access the
`body_links` or `diff_links` keys, Var will populate this with a list of links
obtained by extracting the URLs from the comment body or pull request diff
respectively. This list will also include the status of each link found which
Var obtains by performing a `HEAD` request for each URL.

#### Example

Given the following event data:

```json
{
  "action": "opened",
  "pull_request": {}
}
```

... when the event data after files are attached becomes:

```json
{
  "action": "opened",
  "pull_request": {
    "body_links": [],
    "diff_links": {
      "added": [
        "http://github.com/varci",
        "https://github.com/varci/404"
      ],
      "broken": [
        "https://github.com/varci/404"
      ],
      "suggested": [
        "https://github.com/varci"
      ]
    }
  }
}
```

... the following conditions would resolve to `true`:

```yaml
ruleset:
  broken_links:
    name: "Check for broken links in README.md"
    events: [ pull_request ]
    message: >
      @{{ user.login }}, one of the links in the diff did not return an HTTP
      status code of 200. Please check for broken links. The first broken link
      is: {{ diff_links.broken.0 }}
    when:
      - action = "opened" or action = "reopened"
      - files has "README.md"
      - count(diff_links.broken) > 0
```
