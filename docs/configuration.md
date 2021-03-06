# Configuration

Because different project may want to track different data, and that volumetry on [high activity
projects](https://github.com/docker/docker) makes it unreasonable to store every bits of
information, vossibility-collector was designed to be highly configurable.

The project use the `toml` file format for its configurations. Example files can be found in the
[`examples/`](https://github.com/icecrime/vossibility-collector/tree/master/examples) directory.

## Index

  - [Top-level keys](#top-level-keys)
  - [NSQ configuration](#nsq-section)
  - [Managing repositories](#repositories-section)
  - [Customizing mappings](#mapping-section)
  - [User-defined functions](#functions-section)

## Configuration reference

### Top-level keys

Element            | Type   | Description
------------------ | -------|------------
`elasticsearch`    | String | Host or address for the Elastic Search server
`github_api_token` | String | Optional GitHub API token (important for [API rate limiting](https://developer.github.com/v3/#rate-limiting))
`sync_periodicity` | String | Interval at which the complete state of repositories are synced (`hourly`, `daily`, or `weekly`)

### `[nsq]` section

The `[nsq]` section defines configuration relative to the NSQ queue.

Element            | Type   | Description
------------------ | -------|------------
`channel`          | String | NSQ channel to listen on
`lookupd`          | String | Address of the lookupd server

### `[repositories]` section

The `[repositories]` section defines a collection of tables (in [toml
terminology](https://github.com/toml-lang/toml#table)), where [each of them](#repository-definition)
defines a GitHub repository for which data should be collected.

#### Repository definition

Element            | Type   | Description
------------------ | -------|------------
`user`             | String | GitHub username
`repo`             | String | GitHub repository
`topic`            | String | NSQ topic to listen on for this repository
`events`           | String | Optional reference to an [event set](#[event_set]-section) (defaults to `"default"`)
`start_index`      | String | Optional starting issues # for high activity repositories

### `[event_set]` section

The `[event_set]` section defines a collection of tables (in [toml
terminology](https://github.com/toml-lang/toml#table)), where [each of them](#event-set-definition)
defines a list of GitHub events to subscribe to, along with its related
[transformation](#transformation-definition).

#### Event set definition

In an event set definition, each key should be a valid [GitHub event
identifier](https://developer.github.com/webhooks/#events), and each value a string that references
a given [transformation](#transformation-definition).

We also require two special entries in an event set definition: `snapshot_issue` (which applies to
snapshoted issue content) and `snapshot_pull_request` (which applies to snapshoted pull request
content). Those are mandatory because we assume that issues and pull requests are always being
stored.

### `[transformations]` section

The `[transformation]` section is both the most complex and most interesting section. It defines a
collection of tables (in [toml terminology](https://github.com/toml-lang/toml#table)), where [each
of them](#transformation-definition) defines the data model for storing a particular GitHub object.

#### Transformation definition

A GitHub event is very rich. For example, a simple [pull request
event](https://developer.github.com/v3/activity/events/types/#pullrequestevent) has hundreds of
fields, most of them you'll probably never query or use. That's the point of a transformation: be
able to define your very own data model for the things you care about, and provide a
configuration-style definition of the way that data should be filled.

For this, the project relies on a [modified
version](https://github.com/icecrime/vossibility-collector/tree/master/src/object/template)
of the standard [`text/template`](http://golang.org/pkg/text/template/) Go package. This allows you
to use the standard Go template DSL to describe your data mapping (think of it as a text-based API
for runtime reflection).

#### Transformation example

Let's take a concrete example:

```
    [transformations.issue]
    assignee = "{{ .assignee }}"
    author = "{{ user_data .user.login }}"
    body = "{{ .body }}"
    closed_at = "{{ .closed_at }}"
    comments = "{{ .comments }}"
    created_at = "{{ .created_at }}"
    labels = "{{ range .labels }}{{ .name }}{{ end }}"
    locked = "{{ .locked }}"
    milestone = "{{ if .milestone }}{{ .milestone.title }}{{end}}"
    number = "{{ .number }}"
    state = "{{ .state }}"
    updated_at = "{{ .updated_at }}"
```

In a transformation definition, the resulting data structure will have one field per key in the
table. In this case, the result of applying that transformation to a GitHub object will be a
structure with an `assignee` field, which values is the result of applying the template `{{ .
assignee }}` to the source object. This is a trivial field: the output value is directly that of the
input.

But templates can do much more! One thing they can do is flow control, such as loops or tests. For
example, the `milestone` field in a GitHub issue can either be `nil`, or an object containing
multiple fields, or which we only care about the `title` in that particular example. The template
language allow us to express that we specifically care about the `title` field for a non-`nil`
milestone, in all other cases the resulting `milestone` field will be `nil`.

Same goes for labels, which is an array of objects in the GitHub model, but in most cases we really
just care about the label `name` rather than its `url` or `color` code. The standard template
`range` construct allow us to loop on each label and extract specifically the field we're interested
in. In that case, the output `labels` field will be an array of strings, not an array of objects.

Finally, the template language supports functions. Looking at the `author` field, you'll see that
its value should be the result of evaluating `{{ user_data .user.login }}`. That means: apply the
function `user_data` to the result of evaluating `.user.login` on the source object (`user_data` is
a [builtin function](#builtin-functions) described below).

#### Builtin functions

We currently support the following builtin functions:

- `apply_transformation` takes a single argument and uses this as the name of a transformation to
apply to a sub element of the source object. This is for example particularly useful for a GitHub
[`issue_event`](https://developer.github.com/v3/activity/events/types/#issuesevent) that contains a
nested issue object on which we'd like to apply the same transformation we usually do.

- `context` is a constant function that returns contextual information about the data being
processed. It currently has a single field `Repository` that exposes two methods: `FullName` (e.g.,
`docker/docker`) and `PrettyName` (e.g., `engine (docker:docker)`).

- `days_difference` computes the difference between two GitHub formatted dates and returns the
result as a floating number of days.

- `user_data` takes a single argument, uses its value as a document identifier in the to query an
object of type `user` in the index `users` of the Elastic Search backend, and returns the source
JSON.
  
  This effectively allows to enrich the value of a field with information in database: in our case,
this allows to replace a single login such as `icecrime` into a structure object that contains
information about the person's employer and maintainer status.

### `[mapping]` section

The `[mapping]` section defines configuration relative to the Elastic Search [mapping
mechanism](https://www.elastic.co/guide/en/elasticsearch/guide/current/mapping-intro.html).
It supports a single value of type string array that defines a list of patterns
that should _not_ be analyzed by Elastic Search. In particular, anything that
relates to labels, user names, or company names, are things that you most likely
do not want analyzed using the default analyzer.

Element            | Type   | Description
------------------ | -------|------------
`not_analyzed`     | String array | Patterns to exclude from analyzer

### `[functions]` section

The `[functions]` section defines a collection of user-defined functions that can later be used in
transformation definition. Each key defines the name for the user-defined function, and the value
should be the path of an executable. During template evaluation, arguments to the function are
passed as command line arguments to the executable, and its output is parsed as JSON and returned as
the result of the function. The executable receives the `$ELASTICSEARCH` environment variable which
value is the URL of the Elasticsearch server.
