# JOB-YAML

Updated February 25, 2011

# NAME

job-yaml-v13 - The 'job' YAML file declares job entries for Rundeck.

## Loading and unloading

Rundeck job definitions can be dumped and saved to a file via
rd jobs list command:

```bash
rd jobs list -p project --file /tmp/jobs.yaml -F yaml
```

This file can be batch loaded via [rd] jobs load command:

```bash
rd jobs load -p project --file /path/to/jobs.yaml -F yaml
```

[rd]: https://rundeck.github.io/rundeck-cli/

## Structure

The YAML document can contain multiple Job definitions, in a sequence:

```yaml
- # job 1
  name: ...
- # job 2
  name: ...
```

Each Job definition is a Map consisting of some required and some optional entries, as listed below.

## Job Map Contents

Each Job definition requires these values:

`name`

: the job name (required)

`description`

: the job description (can be blank). The firstline is the "simple description". Remaining lines are the "extended description".

`loglevel`

: the loglevel to use for the job, the value must be one of:

    * `DEBUG`
    * `VERBOSE`
    * `INFO`
    * `WARN`
    * `ERROR`

If the description contains more than one line of text, then the first line is used as the "short description" of the job, and rendered exactly as text. The remaining lines are the "extended description", rendered using Markdown format as HTML in the Rundeck GUI. Markdown can also embed HTML directly if you like. See [Wikipedia - Markdown](https://en.wikipedia.org/wiki/Markdown#Example).

The HTML is sanitized to remove disallowed tags before rendering to the browser (such as `<script>`, etc.).
You can disable all extended description HTML rendering
via a configuration flag.
See [GUI Customization](/administration/configuration/gui-customization.md).

[`sequence`](#sequence)

: The workflow sequence definition

A minimal job definition example:

```yaml
- name: job name
  description: ''
  loglevel: INFO
  sequence:
    commands:
      - exec: a command
```

Extended description using yaml 'literal' scalar string format (beginning with a `|`). Make sure each line is indented to the correct level.

```yaml
- name: job name
  description: |
    Performs a service

    This is <b>html</b>
    <ul><li>bulleted list</li></ul>

    <a href="/">Top</a>

    1. this is a markdown numbered list
    2. second item

    [a link](http://example.com)
  loglevel: INFO
  sequence:
    commands:
      - exec: a command
```

In addition, these optional entries can be present:

`uuid`

: Unique UUID

`group`

: Job group name

`multipleExecutions`

: 'true/false': if true, the job can have more than one execution at once.

`timeout`

: a maximum runtime before the job will be stopped.

    * `120` - indicates 120 seconds
    * `6h 30m` indicates 6 hours and 30 minutes
    * `${option.timeout}` reference to a job option value

`retry`

: Number of times to retry the job if it fails or times out. Allowed values:

    * An integer number indicating the maximum retries
    * `${option.retry}` reference to a job option value

Alternatively the retry can be set with delay between retries:

    * `120` - indicates 120 seconds
    * `6h 30m` indicates 6 hours and 30 minutes
    * `${option.delay}` reference to a job option value

Example of retry with delay:

```yaml
  retry:
      delay: 1h1m1s
      retry: '${option.retry}'
```

Example of simple retry:

```yaml
  retry: ${option.retry}
```

`loglimit`

: An optional logging limit.
(See [Jobs - Log Limit](/manual/04-jobs.md#log-limit)). Allowed values:

    * `###` If you specify a number, that is treated as the "Maximum total number of log lines"
    * `###/node` If you specify a number followed by `/node`, the number is treated as the "Maximum number of log lines for a single node"
    * `###[GMK]B` If you specify a number followed by a filesize suffix, that is treated as the "total log file size".  The file size suffixes allowed are "GB" (gigabyte), "MB" (megabyte), "KB" (kilobyte) and "B" (byte).

`loglimitAction`

: The action to perform if the `loglimit` value is exceeded.
If `loglimit` is specified, but no `loglimitAction` is set, it will default to a
value of `halt`. Allowed values:

    * `halt` - halt and fail the job (default)
    * `truncate` - do not halt the job, and truncate all further output

`loglimitStatus`

: The status for the Job when halted. If no `loglimitStatus` is set, it will default to a
value of `failed`. Allowed values:

    * `failed`
    * `aborted`
    * (any string)

[`options`](#options)

: Set of Options for the Job

[`schedule`](#schedule)

: Job schedule

`nodeFilterEditable`

: Boolean option that determines if Node filters can be edited when running the job. (true | false)

[`nodefilters`](#nodefilters)

: Node filter definition

`nodesSelectedByDefault`

: Boolean option that determines if Nodes (based on node filter) are selected automatically or if the user running the job must select nodes from the results. (true | false)

[`notification`](#notification)

: Job result notifications.

[`orchestrator`](#orchestrator)

: Orchestration Plugin configuration to determine node processing order.

[`plugins`](#plugins)

: Job scoped plugins

_Note:_ The UUID can be set manually (if
you are writing the job definition from scratch), or will be assigned at job
creation time by the Rundeck server using a random UUID. This string should be
as unique as possible if you set it manually.

This identifier is used to uniquely identify jobs when ported between Rundeck
instances.

### Sequence

This defines the Workflow options and execution sequence.

Example:

```yaml
  sequence:
    keepgoing: true
    strategy: node-first
    commands:
    - exec: ...
    - script: ...
      args: ...
    - scriptfile: ...
      args:
    - scripturl: ...
      args:
    - jobref:
        name: jobname
        group: group
        args: args
    - nodeStep: true/false
      type: plugin-type
      configuration:
        key: value
        another: value
```

The sequence has these required entries:

`keepgoing`

: "true/false" - whether the sequence should keep going if an error occurs

`strategy`

: "node-first" or "step-first". Determines the strategy for executing the sequence across a set of nodes. See the [Rundeck User Manual](/manual/04-jobs.md#workflow-control-settings) for more info.

`commands`

: This is a Sequence of: \* One or more [Command Definitions](#command)

### Command

Each command in the [Sequence](#sequence) can be of these different types:

- [Simple command execution entry](#simple-command-entry)
- [Script execution entry](#script-execution-entry)
- [Script file execution entry](#script-file-execution-entry)
- [Job Reference entry](#job-reference-entry)
- [Plugin step entry](#plugin-step-entry)

Each command can also embed an [Error Handler](#error-handler).

Each command can have a [description](#description).

### Error Handler

An Error Handler defines a secondary action in case the first one
fails. An Error Handler is a map keyed with the name:

`errorhandler`

The Error Handler contents can be exactly the same as a [Command](#command), except it
cannot contain another Error Handler. The contents are defined by one of these types:

- [Simple command execution entry](#simple-command-entry)
- [Script execution entry](#script-execution-entry)
- [Script file execution entry](#script-file-execution-entry)
- [Job Reference entry](#job-reference-entry)

The errorhandler has this additional optional entry:

`keepgoingOnSuccess`

: "true/false" - If true, and the error handler succeeds, the workflow sequence will continue even if the workflow `keepgoing` is false.

### description

Defines a description for a step.

`description`

: Text to describe this step (optional).

### Simple Command Entry

This [Command](#command) consists of a single entry:

`exec`

: the command to execute

### Script Execution Entry

This [Command](#command) executes the script content specified.

`script`

: The script content. It is useful to use the YAML "literal" scalar syntax shown below

`args`

: Optional string defining arguments to pass to the script.

Example:

```yaml
   - script: |-
      #!/bin/bash

      echo this is a script
      echo this is option value: @option.test@
    args: arguments passed to the script
```

### Script File Execution Entry

This [Command](#command) executes a script file stored on the server.

`scriptfile`

: path to the script file

`args`

: optional arguments to the script

Example:

```yaml
  - scriptfile: /path/to/script
    args: arguments to script
```

### Script URL Execution Entry

This [Command](#command) downloads a script file from a URL and executes it.

`scripturl`

: URL to the script file

`args`

: optional arguments to the script

Example:

```yaml
  - scripturl: http://example.com/path/to/script
    args: arguments to script
```

### Script Interpreter

For `script`, `scriptfile` and `scripturl`, you can optionally declare an "interpreter" string to use to execute the script, and whether the arguments are quoted or not.

`scriptInterpreter`

: Optional string to declare an interpreter line for the script. The script and args will be passed to this command, rather than executed directly.

Example:

```yaml
   - script: |-
      #!/bin/bash

      echo this is a script
      echo this is option value: @option.test@
    args: arguments passed to the script
    scriptInterpreter: interpreter -flag
```

This script will then be executed as:

    interpreter -flag script.sh arguments ...

`interpreterArgsQuoted`

: Optional boolean, indicating whether the script and arguments should be quoted when passed to the interpreter.

If `interpreterArgsQuoted` is `true`, then the script will then be executed as:

    interpreter -flag 'script.sh arguments ...'

### Job Reference Entry

This [Command](#command) executes another Rundeck Job.

`jobref`

: map consisting of these entries:

    `name`

    :    Name of the Job

    `group`

    :    Group of the Job (optional)

    `args`

    :    Arguments to pass to the job when executed

    `nodeStep`

    :    Execute as a Node Step (optional). `true/false`.

    [nodefilters](#job-reference-nodefilters)

    :    Overriding node filters and dispatch options.

Example:

```yaml
  - jobref:
      group: test
      name: simple job test
      args: args for the job
```

If `nodeStep` is set to "true", then the Job Reference step will operate as a _Node Step_ instead of the
default. As a _Node Step_ it will execute once for each matched node in the containing Job workflow, and
can use node attribute variable expansion in the arguments to the job reference.

#### Job Reference Nodefilters

A `nodefilters` map entry specifies the Nodes to use for the referenced job, and the node-dispatch options. Contains the following entries:

`dispatch`

: a Map containing:

    `keepgoing`

    :    "true/false" - whether to keepgoing on remaining nodes if a node fails

    `threadcount`

    :    Number of threads to use for parallel dispatch (default "1")

    `rankAttribute`

    :    Name of the Node attribute to use for ordering the sequence of nodes (default is the node name)

    `rankOrder`

    :    Order direction for node ranking. Either "ascending" or "descending" (default "ascending")

The `nodefilters` should contain a `filter` entry. The value is a string defining a node filter. See [User Guide - Node Filters](/manual/11-node-filters.md).

`filter`

: A node filter string

Example:

```yaml
- jobref:
   name: jobname
   group: group
   args: args
   nodefilters:
      dispatch:
        threadcount: 1
        keepgoing: false
        rankAttribute: rank
        rankOrder: descending
      filter: 'tags: web name: web-.* !os-family: windows'
```

### Plugin Step Entry

This [Command](#command) executes a plugin. There are two types of step plugins: Node step, and Workflow step.

`nodeStep`

: boolean: true indicates it is a Node step plugin, false indicates a Workflow step plugin.

`type`

: The plugin provider type identifier.

`configuration`

: map consisting of a single level of configuration entries for the plugin. Refer to the plugin documentation for appropriate configuration keys and values.

Example:

```yaml
  - nodeStep: false
    type: jenkins-build
    configuration:
      job: "${option.job}"
```

### Options

Options for a job can be specified with a list of Maps. Each map contains a `name` key with the name of the option, and the content is a map defining the [Option](#option).

```yaml
  options:
  - {definition..}
  - {definition..}
```

Note: for backwards compatibility, a Map format is also accepted on import:

```yaml
  options:
    optname1:
      {definition..}
    optname2:
      {definition..}
```

### Option

An option definition requires at least a `name` key to identify it:

```yaml
  options:
  - name: myoption
```

Optional map entries are:

`description`

: description of the option, will be rendered as Markdown

`value`

: a default value for the option

`values`

: A set of possible values for the option. This must be a YAML Sequence of strings.

`required`

: "true/false" - whether the option is required or not

`enforced`

: "true/false" - whether the option value must be one of the specified possible values

`regex`

: A regular expression defining what option values are acceptable

`valuesUrl`

: A URL to an endpoint that will return a JSON-formatted set of values for the option.

`multivalued`

: "true/false" - whether the option supports multiple input values

`delimiter`

: A string used to conjoin multiple input values. (Required if `multivalued` is "true")

`multivalueAllSelected`

: "true/false" - whether all values should be selected by default

`secure`

: "true/false" - whether the option is a secure input option. Not compatible with "multivalued"

`valueExposed`

: "true/false" - whether a secure input option value is exposed to scripts or not. `false` means the option will be used only as a Secure Remote Authentication option. default: `false`.

`storagePath`

: for a secure option, a storage path to password value to use as default

`isDate`

: "true/false" - the option should display as a date/time input field

`dateFormat`

: The date/time format to use in the UI. Using the [momentjs format](https://momentjs.com/docs/#/displaying/format/).

`sortIndex` (deprecated)

: _integer_ - A number indicating the order this option should appear in the GUI. If specified this
option will be arranged in order with other options with a `sortIndex` value.
If the [Options](#options) are defined in a list, the order specified will be preserved.

The `description` for an Option will be rendered with Markdown in the GUI.

Example:

```yaml
  test:
    required: true
    description: a test option
    value: dvalue
    regex: ^[abcd]value$
    values:
    - avalue
    - bvalue
    - cvalue
    multivalued: true
    delimiter: ','
```

Example using multiple lines for the description:

```yaml
  test:
    required: true
    description: |
      example option description

      * this content will be rendered
      * as markdown
    value: dvalue
    regex: ^[abcd]value$
    values:
    - avalue
    - bvalue
    - cvalue
    multivalued: true
    delimiter: ','
```

#### valuesUrl JSON

The data returned from the valuesUrl can be formatted as a list of values:

```json
["x value","y value"]
```

or as Name-value list:

```json
[
  {name:"X Label", value:"x value"},
  {name:"Y Label", value:"y value"},
  {name:"A Label", value:"a value"}
]
```

- See the [Jobs Guide](/manual/04-jobs.md#remote-option-values) for more info.

### Schedule

Define a schedule for repeated execution of the Job. The schedule can be defined as a Crontab formatted string, or as individual components. The individual components support Crontab syntax.

- `crontab`: The crontab string, e.g. `"0 30 */6 ? Jan Mon *"`

Or use a structure of explicit components. All of these are optional, but likely you want to change them:

`time`

: a map containing:

    `seconds`

    :    seconds value (default: "0")

    `minute`

    :    minutes value (default: "0")

    `hour`

    :    hour value (default: "0")

`month`

: Month value (default: "\*")

`year`

: Year value (default "\*")

`dayofmonth`

    `day`

    :    day of month value. (mutually exclusive with `weekday`) Numerical values start with 1.

`weekday`

: Map containing:

    `day`

    :    Weekday value. (mutually exclusive with `dayofmonth`) (default: "*") Numerical values are 1-7 for Sunday-Saturday.

Example using crontab string:

```yaml
    schedule:
      crontab: '0 30 */6 ? Jan Mon *'
```

Example using structure:

```yaml
    schedule:
      time:
        hour: '05'
        minute: '01'
        seconds: '0'
      month: APR,MAR,MAY
      year: '*'
      weekday:
        day: FRI,MON,TUE
```

### Nodefilters

Specifies the Nodes to use for the job, and the node-dispatch options. Contains the following entries:

`dispatch`

: a Map containing:

    `keepgoing`

    :    "true/false" - whether to keepgoing on remaining nodes if a node fails

    `excludePrecedence`

    :    "true/false" (default "true") - determines precedence for filters

    `threadcount`

    :    Number of threads to use for parallel dispatch (default "1")

    `rankAttribute`

    :    Name of the Node attribute to use for ordering the sequence of nodes (default is the node name)

    `rankOrder`

    :    Order direction for node ranking. Either "ascending" or "descending" (default "ascending")

The `nodefilters` should contain a `filter` entry. The value is a string defining a node filter. See [User Guide - Node Filters](/manual/11-node-filters.md).

`filter`

: A node filter string

Example:

```yaml
  nodefilters:
    dispatch:
      threadcount: 1
      keepgoing: false
      excludePrecedence: true
      rankAttribute: rank
      rankOrder: descending
    filter: 'tags: web name: web-.* !os-family: windows'
```

**Note:** The `include` and `exclude` map entries are deprecated and will be removed in a later version of Rundeck.

The `nodefilters` must also contain ONE of `include` or `exclude` filter specifiers.

`include`/`exclude`

: A Map containing filter entries:

    `hostname`

    :    Hostname filter

    `name`

    :    Node name filter

    `tags`

    :    Tags filter.  Supports boolean operators AND ("+") and OR (",").

    `os-name`

    :    OS name filter

    `os-family`

    :    OS Family filter

    `os-arch`

    :    OS Arch filter

    `os-version`

    :    OS Version filter

Deprecated Example:

```yaml
  nodefilters:
    dispatch:
      threadcount: 1
      keepgoing: false
      excludePrecedence: true
      rankAttribute: rank
      rankOrder: descending
    include:
      tags: web
      name: web-.*
    exclude:
      os-family: windows
```

### Notification

Defines a notification for the job. You can include any of `onsuccess`, `onfailure`, `onstart`, `onavgduration`, or `onretryablefailure` notifications. Each type of notification can define any of the built in notifications, or define plugin notifications.

`onsuccess`/`onfailure`/`onstart`/`onavgduration`/`onretryablefailure`

::: tip
`onavgduration` also requires the following attribute set at the same level as `notification`

`notifyAvgDurationThreshold` - Add or set a threshold value to the avg duration in order to trigger this notification. Options: - percentage => eg: 20% - time delta => eg: +20s, +20 - absolute time => 30s, 5m Time in seconds if you don't specify time units Can include option value references like ${option.avgDurationThreshold}.
(see example below)
:::

: A Map containing either or both of:

    `recipients`

    :    A comma-separated list of Email addresses

    `urls`

    :    A comma-separated list of URLs to use as webhooks
    `httpMethod`

    :    HTTP method (get/post) used in the webhooks
    `format`

    :    The payload format (xml/json) used in the webhooks

    [`plugin`](#plugin)

    :    Defines a plugin notification.

Example:

```yaml
  notifyAvgDurationThreshold: '+30'
  notification:
    onavgduration:
    - email:
        recipients: test@example.com
        subject: Job Exceeded average duration
    - plugin:
      - type: my-plugin
        configuration:
          somekey: somevalue
    onfailure:
    - email:
        recipients: 'tom@example.com,shirley@example.com'
    - email:
        attachLog: 'true'
        attachLogInline: true
        recipients: manager@example.com
        subject: JOB-FAILURE
    onretryablefailure:
    - plugin:
      - type: my-plugin
        configuration:
          somekey: somevalue
      - type: my-plugin
        configuration:
          somekey: somevalue
    onstart:
      email:
        attachLog: true
        attachLogInFile: true
        recipients: tom@example.com
        subject: JOB-STARTED
    onsuccess:
    - format: xml
      httpMethod: post
      urls: http://server/callback?id=${execution.id}&status=${execution.status}&trigger=${notification.trigger}
    - plugin:
      - type: my-plugin
        configuration:
          somekey: somevalue
```

#### Another valid format
If there is no need to have more than one notification of the same type on **any** trigger the following is a valid definition. 

Example:

```yaml
  notifyAvgDurationThreshold: '+30'
  notification:
    onstart:
      email:
        attachLog: 'true'
        attachLogInFile: true
        recipients: tom@example.com
        subject: JOB-STARTED
    onfailure:
      email:
        recipients: 'tom@example.com,shirley@example.com'
    onsuccess:
      urls: 'http://server/callback?id=${execution.id}&status=${execution.status}&trigger=${notification.trigger}'
      httpMethod: post
      format: xml
      plugin:
        type: my-plugin
        configuration:
          somekey: somevalue
    onavgduration:
      email:
        recipients: test@example.com
        subject: Job Exceeded average duration
      plugin:
        type: my-plugin
        configuration:
          somekey: somevalue
    onretryablefailure:
      plugin:
        type: my-plugin
        configuration:
          somekey: somevalue
```

- For more information about the Webhook mechanism used, see the chapter [Integration - Webhooks](/manual/04-jobs.md#webhooks).

### Orchestrator

Defines a Orchestrator Plugin that can be used to determine the order in which nodes are processed.

`type`

: The type identifier of the Orchestrator plugin

`configuration`

: Contains configuration attributes for the plugin.  Values will vary depending on Orchestrator plugin.

Example:

```yaml
    orchestrator:
    configuration:
      attribute: sort-attr-on-node
      sort: highest
    type: orchestrator-highest-lowest-attribute
```

#### plugin

Defines a plugin notification section, can contain a single Map, or a Sequence of Maps. Each such map must have these contents:

`type`

: The type identifier of the plugin

`configuration`

: A Map containing any custom configuration key/values for the plugin.

### Plugins

Defines job-scoped plugin entries, such as Execution Lifecycle Plugins.

Example:

```yaml
  plugins:
    ExecutionLifecycle:
      myplugin: {}
      anotherPlugin:
        prop1: value
```

The `plugins` entry contains an entry for each plugin service, such as `ExecutionLifecycle`.
This will contain entries for each provider type, using the provider name as the key.

Each provider may contain a configuration Map, or if there is no configuration for the plugin, an empty map, `{}`.

# SEE ALSO


<http://yaml.org/>
