---
title: "Events reference"
linkTitle: "Events Reference"
reference_title: "Events"
type: "reference"
description: "An event is a generic container that Sensu uses to provide context for checks and metrics. You can use events to represent the state of your infrastructure and create automated monitoring workflows. Read the reference doc to learn about events."
weight: 20
version: "6.2"
product: "Sensu Go"
platformContent: false
menu:
  sensu-go-6.2:
    parent: observe-events
---

An event is a generic container used by Sensu to provide context to checks and metrics.
The context, called observation data or event data, contains information about the originating entity and the corresponding check or metric result.
An event must contain a [status][4] or [metrics][5].
In certain cases, an event can contain [both a status and metrics][19].
These generic containers allow Sensu to handle different types of events in the pipeline.
Because events are polymorphic in nature, it is important to never assume their contents (or lack of content).

## Create events using the Sensu agent

The Sensu agent is a powerful event producer and monitoring automation tool.
You can use Sensu agents to produce events automatically using service checks and metric checks.
Sensu agents can also act as a collector for metrics throughout your infrastructure.

- [Create events using service checks][10]
- [Create events using metric checks][6]
- [Create events using the agent API][11]
- [Create events using the agent TCP and UDP sockets][12]
- [Create events using the StatsD listener][13]

## Create events using the events API

You can send events directly to the Sensu pipeline using the [events API][16].
To create an event, send a JSON event definition to the [events API PUT endpoint][14].

If you use the events API to create a new event referencing an entity that does not already exist, the sensu-backend will automatically create a proxy entity when the event is published.

## Manage events

You can manage events using the [Sensu web UI][15], [events API][16], and [sensuctl][17] command line tool.

### View events

To list all events:

{{< code shell >}}
sensuctl event list
{{< /code >}}

To show event details in the default [output format][18]:

{{< code shell >}}
sensuctl event info entity-name check-name
{{< /code >}}

With both the `list` and `info` commands, you can specify an [output format][18] using the `--format` flag:

- `yaml` or `wrapped-json` formats for use with [`sensuctl create`][8]
- `json` format for use with the [events API][16]

{{< code shell >}}
sensuctl event info entity-name check-name --format yaml
{{< /code >}}

### Delete events

To delete an event:

{{< code shell >}}
sensuctl event delete entity-name check-name
{{< /code >}}

You can use the `--skip-confirm` flag to skip the confirmation step:

{{< code shell >}}
sensuctl event delete entity-name check-name --skip-confirm
{{< /code >}}

You should see a confirmation message upon success:

{{< code shell >}}
Deleted
{{< /code >}}

### Resolve events

You can use sensuctl to change the status of an event to `0` (OK).
Events resolved by sensuctl include the output message `Resolved manually by sensuctl`.

{{< code shell >}}
sensuctl event resolve entity-name check-name
{{< /code >}}

You should see a confirmation message upon success:

{{< code shell >}}
Resolved
{{< /code >}}

## Event format

Sensu events contain:

- `entity` scope (required)
  - Information about the source of the event, including any attributes defined in the [entity specification][2]
- `check` scope (optional if the `metrics` scope is present)
  - Information about how the event was created, including any attributes defined in the [check specification][20]
  - Information about the event and its history, including any check attributes defined in the [event specification on this page][21]
- `metrics` scope (optional if the `check` scope is present)
  - Metric points in [Sensu metric format][22]
- `timestamp`
  - Time that the event occurred in seconds since the Unix epoch
- `event_id`
  - Universally unique identifier (UUID) for the event

## Use event data

Observability data in events is a powerful tool for automating monitoring workflows.
For example, the [`state` attribute][36] provides handlers with a high-level description of check status.
Filtering events based on this attribute can help [reduce alert fatigue][23].

### State attribute

The `state` event attribute adds meaning to the check status:

- `passing` means the check status is `0` (OK).
- `failing` means the check status is non-zero (WARNING or CRITICAL).
- `flapping` indicates an unsteady state in which the check result status (determined based on per-check [low and high flap thresholds][37] attributes) is not settling on `passing` or `failing` according to the [flap detection algorithm][39].

Flapping typically indicates intermittent problems with an entity, provided your low and high flap threshold settings are properly configured.
Although some teams choose to filter out flapping events to reduce unactionable alerts, we suggest sending flapping events to a designated handler for later review.
If you repeatedly observe events in flapping state, Sensu's per-check flap threshold configuration allows you to adjust the sensitivity of the [flap detection algorithm][39].

#### Flap detection algorithm

Sensu uses the same flap detection algorithm as [Nagios][38].
Every time you run a check, Sensu records whether the `status` value changed since the previous check.
Sensu stores the last 21 `status` values and uses them to calculate the percent state change for the entity/check pair.
Then, Sensu's algorithm applies a weight to these status changes: more recent changes have more value than older changes.

After calculating the weighted total percent state change, Sensu compares it with the [low and high flap thresholds][37] set in the check attributes.

- If the entity was **not** already flapping and the weighted total percent state change for the entity/check pair is greater than or equal to the `high_flap_threshold` setting, the entity has started flapping.
- If the entity **was** already flapping and the weighted total percent state change for the entity/check pair is less than the `low_flap_threshold` setting, the entity has stopped flapping.

Depending on the result of this comparison, Sensu will trigger the appropriate event filters based on [check attributes][40] like `event.check.high_flap_threshold` and `event.check.low_flap_threshold`.

### Occurrences and occurrences watermark

The `occurrences` and `occurrences_watermark` event attributes give you context about recent events for a given entity and check.
You can use these attributes within [event filters][24] to fine-tune incident notifications and reduce alert fatigue.

Starting at `1`, the `occurrences` attribute increments for events with the same [status][25] as the preceding event (OK, WARNING, CRITICAL, or UNKNOWN) and resets whenever the status changes.
You can use the `occurrences` attribute to create a [state-change-only filter][27] or an [interval filter][28].

The `occurrences_watermark` attribute gives you useful information when looking at events that change status between non-OK (WARNING, CRITICAL, or UNKNOWN) and OK.
For these resolution events, the `occurrences_watermark` attribute tells you the number of preceding events with a non-OK status.
Sensu resets `occurrences_watermark` to `1` on the first non-OK event.
Within a sequence of only OK or only non-OK events, Sensu increments `occurrences_watermark` when the `occurrences` attribute is greater than the preceding `occurrences_watermark`.

The following table shows the occurrences attributes for a series of example events:

| event sequence   | `occurrences`   | `occurrences_watermark` |
| -----------------| --------------- | ----------------------- |
|1. OK event        | `occurrences: 1`| `occurrences_watermark: 1`
|2. OK event        | `occurrences: 2`| `occurrences_watermark: 2`
|3. WARNING event   | `occurrences: 1`| `occurrences_watermark: 1`
|4. WARNING event   | `occurrences: 2`| `occurrences_watermark: 2`
|5. WARNING event   | `occurrences: 3`| `occurrences_watermark: 3`
|6. CRITICAL event  | `occurrences: 1`| `occurrences_watermark: 3`
|7. CRITICAL event  | `occurrences: 2`| `occurrences_watermark: 3`
|8. CRITICAL event  | `occurrences: 3`| `occurrences_watermark: 3`
|9. CRITICAL event  | `occurrences: 4`| `occurrences_watermark: 4`
|10. OK event       | `occurrences: 1`| `occurrences_watermark: 4`
|11. CRITICAL event | `occurrences: 1`| `occurrences_watermark: 1`

## Events specification

### Top-level attributes

type         | 
-------------|------
description  | Top-level attribute that specifies the [`sensuctl create`][8] resource type. Events should always be type `Event`.
required     | Required for events in `wrapped-json` or `yaml` format for use with [`sensuctl create`][8].
type         | String
example      | {{< code shell >}}"type": "Event"{{< /code >}}

api_version  | 
-------------|------
description  | Top-level attribute that specifies the Sensu API group and version. For events in this version of Sensu, `api_version` should always be `core/v2`.
required     | Required for events in `wrapped-json` or `yaml` format for use with [`sensuctl create`][8].
type         | String
example      | {{< code shell >}}"api_version": "core/v2"{{< /code >}}

metadata     | 
-------------|------
description  | Top-level scope that contains the event `namespace` and `created_by` field. The `metadata` map is always at the top level of the check definition. This means that in `wrapped-json` and `yaml` formats, the `metadata` scope occurs outside the `spec` scope.  See the [metadata attributes][29] for details.
required     | Required for events in `wrapped-json` or `yaml` format for use with [`sensuctl create`][8].
type         | Map of key-value pairs
example      | {{< code shell >}}"metadata": {
  "namespace": "default",
  "created_by": "admin"
}{{< /code >}}

spec         | 
-------------|------
description  | Top-level map that includes the event [spec attributes][9].
required     | Required for events in `wrapped-json` or `yaml` format for use with [`sensuctl create`][8].
type         | Map of key-value pairs
example      | {{< code shell >}}
"spec": {
  "check": {
    "check_hooks": null,
    "command": "/opt/sensu-plugins-ruby/embedded/bin/metrics-curl.rb -u \"http://localhost\"",
    "duration": 0.060790838,
    "env_vars": null,
    "executed": 1552506033,
    "handlers": [],
    "high_flap_threshold": 0,
    "history": [
      {
        "executed": 1552505833,
        "status": 0
      },
      {
        "executed": 1552505843,
        "status": 0
      }
    ],
    "interval": 10,
    "is_silenced": true,
    "issued": 1552506033,
    "last_ok": 1552506033,
    "low_flap_threshold": 0,
    "metadata": {
      "name": "curl_timings",
      "namespace": "default"
    },
    "occurrences": 1,
    "occurrences_watermark": 1,
    "silenced": [
      "webserver:*"
    ],
    "output": "sensu-go-sandbox.curl_timings.time_total 0.005 1552506033\nsensu-go-sandbox.curl_timings.time_namelookup 0.004",
    "output_metric_format": "graphite_plaintext",
    "output_metric_handlers": [
      "influx-db"
    ],
    "proxy_entity_name": "",
    "publish": true,
    "round_robin": false,
    "runtime_assets": [],
    "state": "passing",
    "status": 0,
    "stdin": false,
    "subdue": null,
    "subscriptions": [
      "entity:sensu-go-sandbox"
    ],
    "timeout": 0,
    "total_state_change": 0,
    "ttl": 0
  },
  "entity": {
    "deregister": false,
    "deregistration": {},
    "entity_class": "agent",
    "last_seen": 1552495139,
    "metadata": {
      "name": "sensu-go-sandbox",
      "namespace": "default"
    },
    "redact": [
      "password",
      "passwd",
      "pass",
      "api_key",
      "api_token",
      "access_key",
      "secret_key",
      "private_key",
      "secret"
    ],
    "subscriptions": [
      "entity:sensu-go-sandbox"
    ],
    "system": {
      "arch": "amd64",
      "hostname": "sensu-go-sandbox",
      "network": {
        "interfaces": [
          {
            "addresses": [
              "127.0.0.1/8",
              "::1/128"
            ],
            "name": "lo"
          },
          {
            "addresses": [
              "10.0.2.15/24",
              "fe80::5a94:f67a:1bfc:a579/64"
            ],
            "mac": "08:00:27:8b:c9:3f",
            "name": "eth0"
          }
        ]
      },
      "os": "linux",
      "platform": "centos",
      "platform_family": "rhel",
      "platform_version": "7.5.1804",
      "processes": null
    },
    "user": "agent"
  },
  "metrics": {
    "handlers": [
      "influx-db"
    ],
    "points": [
      {
        "name": "sensu-go-sandbox.curl_timings.time_total",
        "tags": [],
        "timestamp": 1552506033,
        "value": 0.005
      },
      {
        "name": "sensu-go-sandbox.curl_timings.time_namelookup",
        "tags": [],
        "timestamp": 1552506033,
        "value": 0.004
      }
    ]
  },
  "timestamp": 1552506033,
  "event_id": "431a0085-96da-4521-863f-c38b480701e9"
}
{{< /code >}}

### Metadata attributes

| namespace  |      |
-------------|------
description  | [Sensu RBAC namespace][26] that this event belongs to.
required     | false
type         | String
default      | `default`
example      | {{< code shell >}}"namespace": "production"{{< /code >}}

| created_by |      |
-------------|------
description  | Username of the Sensu user who created the event or last updated the event. Sensu automatically populates the `created_by` field when the event is created or updated.
required     | false
type         | String
example      | {{< code shell >}}"created_by": "admin"{{< /code >}}

### Spec attributes

|timestamp   |      |
-------------|------
description  | Time that the event occurred. In seconds since the Unix epoch.
required     | false
type         | Integer
default      | Time that the event occurred
example      | {{< code shell >}}"timestamp": 1522099512{{< /code >}}

event_id     |      |
-------------|------
description  | Universally unique identifier (UUID) for the event.
required     | false
type         | String
example      | {{< code shell >}}"event_id": "431a0085-96da-4521-863f-c38b480701e9"{{< /code >}}

|entity      |      |
-------------|------
description  | [Entity attributes][2] from the originating entity (agent or proxy). If you use the [events API][35] to create a new event referencing an entity that does not already exist, the sensu-backend will automatically create a proxy entity when the event is published.
type         | Map
required     | true
example      | {{< code shell >}}
"entity": {
  "deregister": false,
  "deregistration": {},
  "entity_class": "agent",
  "last_seen": 1552495139,
  "metadata": {
    "name": "sensu-go-sandbox",
    "namespace": "default"
  },
  "redact": [
    "password",
    "passwd",
    "pass",
    "api_key",
    "api_token",
    "access_key",
    "secret_key",
    "private_key",
    "secret"
  ],
  "subscriptions": [
    "entity:sensu-go-sandbox"
  ],
  "system": {
    "arch": "amd64",
    "hostname": "sensu-go-sandbox",
    "network": {
      "interfaces": [
        {
          "addresses": [
            "127.0.0.1/8",
            "::1/128"
          ],
          "name": "lo"
        },
        {
          "addresses": [
            "10.0.2.15/24",
            "fe80::5a94:f67a:1bfc:a579/64"
          ],
          "mac": "08:00:27:8b:c9:3f",
          "name": "eth0"
        }
      ]
    },
    "os": "linux",
    "platform": "centos",
    "platform_family": "rhel",
    "platform_version": "7.5.1804"
  },
  "user": "agent"
}
{{< /code >}}

<a name="checks"></a>

|check       |      |
-------------|------
description  | [Check definition][1] used to create the event and information about the status and history of the event. The check scope includes attributes described in the [event specification][21] and the [check specification][20].
type         | Map
required     | true
example      | {{< code shell >}}
"check": {
  "check_hooks": null,
  "command": "/opt/sensu-plugins-ruby/embedded/bin/metrics-curl.rb -u \"http://localhost\"",
  "duration": 0.060790838,
  "env_vars": null,
  "executed": 1552506033,
  "handlers": [],
  "high_flap_threshold": 0,
  "history": [
    {
      "executed": 1552505833,
      "status": 0
    },
    {
      "executed": 1552505843,
      "status": 0
    }
  ],
  "interval": 10,
  "is_silenced": true,
  "issued": 1552506033,
  "last_ok": 1552506033,
  "low_flap_threshold": 0,
  "metadata": {
    "name": "curl_timings",
    "namespace": "default"
  },
  "occurrences": 1,
  "occurrences_watermark": 1,
  "silenced": [
    "webserver:*"
  ],
  "output": "sensu-go-sandbox.curl_timings.time_total 0.005",
  "output_metric_format": "graphite_plaintext",
  "output_metric_handlers": [
    "influx-db"
  ],
  "proxy_entity_name": "",
  "publish": true,
  "round_robin": false,
  "runtime_assets": [],
  "state": "passing",
  "status": 0,
  "stdin": false,
  "subdue": null,
  "subscriptions": [
    "entity:sensu-go-sandbox"
  ],
  "timeout": 0,
  "total_state_change": 0,
  "ttl": 0
}
{{< /code >}}

<a name="metrics"></a>

|metrics     |      |
-------------|------
description  | Metrics collected by the entity in Sensu metric format. See the [metric attributes][30].
type         | Map
required     | false
example      | {{< code shell >}}
"metrics": {
  "handlers": [
    "influx-db"
  ],
  "points": [
    {
      "name": "sensu-go-sandbox.curl_timings.time_total",
      "tags": [],
      "timestamp": 1552506033,
      "value": 0.005
    },
    {
      "name": "sensu-go-sandbox.curl_timings.time_namelookup",
      "tags": [],
      "timestamp": 1552506033,
      "value": 0.004
    }
  ]
}
{{< /code >}}

### Check attributes

Sensu events include a `check` scope that contains information about how the event was created, including any attributes defined in the [check specification][20], and information about the event and its history, including the attributes defined below.

duration     |      |
-------------|------
description  | Command execution time. In seconds.
required     | false
type         | Float
example      | {{< code shell >}}"duration": 1.903135228{{< /code >}}

executed     |      |
-------------|------
description  | Time at which the check request was executed. In seconds since the Unix epoch.
required     | false
type         | Integer
example      | {{< code shell >}}"executed": 1522100915{{< /code >}}

history      |      |
-------------|------
description  | Check status history for the last 21 check executions. See [history attributes][32].
required     | false
type         | Array
example      | {{< code shell >}}
"history": [
  {
    "executed": 1552505983,
    "status": 0
  },
  {
    "executed": 1552505993,
    "status": 0
  }
]
{{< /code >}}

issued       |      |
-------------|------
description  | Time that the check request was issued. In seconds since the Unix epoch.
required     | false
type         | Integer
example      | {{< code shell >}}"issued": 1552506033{{< /code >}}

last_ok      |      |
-------------|------
description  | Last time that the check returned an OK status (`0`). In seconds since the Unix epoch.
required     | false
type         | Integer
example      | {{< code shell >}}"last_ok": 1552506033{{< /code >}}

occurrences  |      |
-------------|------
description  | Number of preceding events with the same status as the current event (OK, WARNING, CRITICAL, or UNKNOWN). Starting at `1`, the `occurrences` attribute increments for events with the same status as the preceding event and resets whenever the status changes. See [Use event data][31] for more information.
required     | false
type         | Integer greater than 0
example      | {{< code shell >}}"occurrences": 1{{< /code >}}

occurrences_watermark | |
-------------|------
description  | For incident and resolution events, the number of preceding events with an OK status (for incident events) or non-OK status (for resolution events). The `occurrences_watermark` attribute gives you useful information when looking at events that change status between OK (`0`)and non-OK (`1`-WARNING, `2`-CRITICAL, or UNKNOWN).<br><br>Sensu resets `occurrences_watermark` to `1` whenever an event for a given entity and check transitions between OK and non-OK. Within a sequence of only OK or only non-OK events, Sensu increments `occurrences_watermark` only when the `occurrences` attribute is greater than the preceding `occurrences_watermark`. See [Use event data][31] for more information.
required     | false
type         | Integer greater than 0
example      | {{< code shell >}}"occurrences_watermark": 1{{< /code >}}

is_silenced  | |
-------------|------
description  | If `true`, the event was silenced at the time of processing. Otherwise, `false`. If `true`, the event.Check definition will also list the silenced entries that match the event in the `silenced` array.
required     | false
type         | Boolean
example      | {{< code shell >}}"is_silenced": "true"{{< /code >}}

silenced     | |
-------------|------
description  | Array of silencing entries that match the event. The `silenced` attribute is only present for events if one or more silencing entries matched the event at time of processing. If the `silenced` attribute is not present in an event, the event was not silenced at the time of processing.
required     | false
type         | Array
example      | {{< code shell >}}"silenced": [
  "webserver:*"
]{{< /code >}}

output       |      |
-------------|------
description  | Output from the execution of the check command.
required     | false
type         | String
example      | {{< code shell >}}
"output": "sensu-go-sandbox.curl_timings.time_total 0.005"
{{< /code >}}

state         |      |
-------------|------
description  | State of the check: `passing` (status `0`), `failing` (status other than `0`), or `flapping`. You can use the `low_flap_threshold` and `high_flap_threshold` [check attributes][33] to configure `flapping` state detection.
required     | false
type         | String
example      | {{< code shell >}}"state": "passing"{{< /code >}}

status       |      |
-------------|------
description  | Exit status code produced by the check.<ul><li><code>0</code> indicates “OK”</li><li><code>1</code> indicates “WARNING”</li><li><code>2</code> indicates “CRITICAL”</li></ul>Exit status codes other than <code>0</code>, <code>1</code>, or <code>2</code> indicate an “UNKNOWN” or custom status.
required     | false
type         | Integer
example      | {{< code shell >}}"status": 0{{< /code >}}

total_state_change | |
-------------|------
description  | Total state change percentage for the check's history.
required     | false
type         | Integer
example      | {{< code shell >}}"total_state_change": 0{{< /code >}}

#### History attributes

executed     |      |
-------------|------
description  | Time at which the check request was executed. In seconds since the Unix epoch.
required     | false
type         | Integer
example      | {{< code shell >}}"executed": 1522100915{{< /code >}}

status       |      |
-------------|------
description  | Exit status code produced by the check.<ul><li><code>0</code> indicates “OK”</li><li><code>1</code> indicates “WARNING”</li><li><code>2</code> indicates “CRITICAL”</li></ul>Exit status codes other than <code>0</code>, <code>1</code>, or <code>2</code> indicate an “UNKNOWN” or custom status.
required     | false
type         | Integer
example      | {{< code shell >}}"status": 0{{< /code >}}

### Metric attributes

handlers     |      |
-------------|------
description  | Array of Sensu handlers to use for events created by the check. Each array item must be a string.
required     | false
type         | Array
example      | {{< code shell >}}
"handlers": [
  "influx-db"
]
{{< /code >}}

points       |      |
-------------|------
description  | Metric data points, including a name, timestamp, value, and tags. See [points attributes][34].
required     | false
type         | Array
example      | {{< code shell >}}
"points": [
  {
    "name": "sensu-go-sandbox.curl_timings.time_total",
    "tags": [
      {
        "name": "response_time_in_ms",
        "value": "101"
      }
    ],
    "timestamp": 1552506033,
    "value": 0.005
  },
  {
    "name": "sensu-go-sandbox.curl_timings.time_namelookup",
    "tags": [
      {
        "name": "namelookup_time_in_ms",
        "value": "57"
      }
    ],
    "timestamp": 1552506033,
    "value": 0.004
  }
]
{{< /code >}}

#### Points attributes

name         |      |
-------------|------
description  | Metric name in the format `$entity.$check.$metric` where `$entity` is the entity name, `$check` is the check name, and `$metric` is the metric name.
required     | false
type         | String
example      | {{< code shell >}}"name": "sensu-go-sandbox.curl_timings.time_total"{{< /code >}}

tags         |      |
-------------|------
description  | Optional tags to include with the metric. Each element of the array must be a hash that contains two key value pairs: the `name` of the tag and the `value`. Both values of the pairs must be strings.
required     | false
type         | Array
example      | {{< code shell >}}
"tags": [
  {
    "name": "response_time_in_ms",
    "value": "101"
  }
]
{{< /code >}}

timestamp    |      |
-------------|------
description  | Time at which the metric was collected. In seconds since the Unix epoch.
required     | false
type         | Integer
example      | {{< code shell >}}"timestamp": 1552506033{{< /code >}}

value        |      |
-------------|------
description  | Metric value.
required     | false
type         | Float
example      | {{< code shell >}}"value": 0.005{{< /code >}}

## Examples

### Example check-only event data

{{< language-toggle >}}

{{< code yml >}}
type: Event
api_version: core/v2
metadata:
  namespace: default
spec:
  check:
    check_hooks: null
    command: check-cpu.sh -w 75 -c 90
    duration: 1.07055808
    env_vars: null
    executed: 1552594757
    handlers: []
    high_flap_threshold: 0
    history:
    - executed: 1552594757
      status: 0
    interval: 60
    is_silenced: true
    issued: 1552594757
    last_ok: 1552594758
    low_flap_threshold: 0
    metadata:
      name: check-cpu
      namespace: default
    occurrences: 1
    occurrences_watermark: 1
    output: |
      CPU OK - Usage:3.96
    silenced:
      entity:gin:server-health
    output_metric_format: ""
    output_metric_handlers: []
    proxy_entity_name: ""
    publish: true
    round_robin: false
    runtime_assets: []
    state: passing
    status: 0
    stdin: false
    subdue: null
    subscriptions:
    - linux
    timeout: 0
    total_state_change: 0
    ttl: 0
  entity:
    deregister: false
    deregistration: {}
    entity_class: agent
    last_seen: 1552594641
    metadata:
      name: sensu-centos
      namespace: default
    redact:
    - password
    - passwd
    - pass
    - api_key
    - api_token
    - access_key
    - secret_key
    - private_key
    - secret
    subscriptions:
    - linux
    - entity:sensu-centos
    system:
      arch: amd64
      hostname: sensu-centos
      network:
        interfaces:
        - addresses:
          - 127.0.0.1/8
          - ::1/128
          name: lo
        - addresses:
          - 10.0.2.15/24
          - fe80::9688:67ca:3d78:ced9/64
          mac: 08:00:27:11:ad:d2
          name: enp0s3
        - addresses:
          - 172.28.128.3/24
          - fe80::a00:27ff:fe6b:c1e9/64
          mac: 08:00:27:6b:c1:e9
          name: enp0s8
      os: linux
      platform: centos
      platform_family: rhel
      platform_version: 7.4.1708
      processes: null
    user: agent
  timestamp: 1552594758
  event_id: 3a5948f3-6ffd-4ea2-a41e-334f4a72ca2f
{{< /code >}}

{{< code json >}}
{
  "type": "Event",
  "api_version": "core/v2",
  "metadata": {
    "namespace": "default"
  },
  "spec": {
    "check": {
      "check_hooks": null,
      "command": "check-cpu.sh -w 75 -c 90",
      "duration": 1.07055808,
      "env_vars": null,
      "executed": 1552594757,
      "handlers": [],
      "high_flap_threshold": 0,
      "history": [
        {
          "executed": 1552594757,
          "status": 0
        }
      ],
      "interval": 60,
      "is_silenced": true,
      "issued": 1552594757,
      "last_ok": 1552594758,
      "low_flap_threshold": 0,
      "metadata": {
        "name": "check-cpu",
        "namespace": "default"
      },
      "occurrences": 1,
      "occurrences_watermark": 1,
      "output": "CPU OK - Usage:3.96\n",
      "silenced": [
        "entity:gin:scheck-cpu"
      ],
      "output_metric_format": "",
      "output_metric_handlers": [],
      "proxy_entity_name": "",
      "publish": true,
      "round_robin": false,
      "runtime_assets": [],
      "state": "passing",
      "status": 0,
      "stdin": false,
      "subdue": null,
      "subscriptions": [
        "linux"
      ],
      "timeout": 0,
      "total_state_change": 0,
      "ttl": 0
    },
    "entity": {
      "deregister": false,
      "deregistration": {},
      "entity_class": "agent",
      "last_seen": 1552594641,
      "metadata": {
        "name": "sensu-centos",
        "namespace": "default"
      },
      "redact": [
        "password",
        "passwd",
        "pass",
        "api_key",
        "api_token",
        "access_key",
        "secret_key",
        "private_key",
        "secret"
      ],
      "subscriptions": [
        "linux",
        "entity:sensu-centos"
      ],
      "system": {
        "arch": "amd64",
        "hostname": "sensu-centos",
        "network": {
          "interfaces": [
            {
              "addresses": [
                "127.0.0.1/8",
                "::1/128"
              ],
              "name": "lo"
            },
            {
              "addresses": [
                "10.0.2.15/24",
                "fe80::9688:67ca:3d78:ced9/64"
              ],
              "mac": "08:00:27:11:ad:d2",
              "name": "enp0s3"
            },
            {
              "addresses": [
                "172.28.128.3/24",
                "fe80::a00:27ff:fe6b:c1e9/64"
              ],
              "mac": "08:00:27:6b:c1:e9",
              "name": "enp0s8"
            }
          ]
        },
        "os": "linux",
        "platform": "centos",
        "platform_family": "rhel",
        "platform_version": "7.4.1708",
        "processes": null
      },
      "user": "agent"
    },
    "timestamp": 1552594758,
    "event_id": "3a5948f3-6ffd-4ea2-a41e-334f4a72ca2f"
  }
}
{{< /code >}}

{{< /language-toggle >}}

### Example event with check and metric data

{{< language-toggle >}}

{{< code yml >}}
type: Event
api_version: core/v2
metadata:
  namespace: default
spec:
  check:
    check_hooks: null
    command: /opt/sensu-plugins-ruby/embedded/bin/metrics-curl.rb -u "http://localhost"
    duration: 0.060790838
    env_vars: null
    executed: 1552506033
    handlers: []
    high_flap_threshold: 0
    history:
    - executed: 1552505833
      status: 0
    - executed: 1552505843
      status: 0
    interval: 10
    is_silenced: false
    issued: 1552506033
    last_ok: 1552506033
    low_flap_threshold: 0
    metadata:
      name: curl_timings
      namespace: default
    occurrences: 1
    occurrences_watermark: 1
    output: |-
      sensu-go-sandbox.curl_timings.time_total 0.005 1552506033
      sensu-go-sandbox.curl_timings.time_namelookup 0.004
    output_metric_format: graphite_plaintext
    output_metric_handlers:
    - influx-db
    proxy_entity_name: ""
    publish: true
    round_robin: false
    runtime_assets: []
    state: passing
    status: 0
    stdin: false
    subdue: null
    subscriptions:
    - entity:sensu-go-sandbox
    timeout: 0
    total_state_change: 0
    ttl: 0
  entity:
    deregister: false
    deregistration: {}
    entity_class: agent
    last_seen: 1552495139
    metadata:
      name: sensu-go-sandbox
      namespace: default
    redact:
    - password
    - passwd
    - pass
    - api_key
    - api_token
    - access_key
    - secret_key
    - private_key
    - secret
    subscriptions:
    - entity:sensu-go-sandbox
    system:
      arch: amd64
      hostname: sensu-go-sandbox
      network:
        interfaces:
        - addresses:
          - 127.0.0.1/8
          - ::1/128
          name: lo
        - addresses:
          - 10.0.2.15/24
          - fe80::5a94:f67a:1bfc:a579/64
          mac: 08:00:27:8b:c9:3f
          name: eth0
      os: linux
      platform: centos
      platform_family: rhel
      platform_version: 7.5.1804
      processes: null
    user: agent
  metrics:
    handlers:
    - influx-db
    points:
    - name: sensu-go-sandbox.curl_timings.time_total
      tags: []
      timestamp: 1552506033
      value: 0.005
    - name: sensu-go-sandbox.curl_timings.time_namelookup
      tags: []
      timestamp: 1552506033
      value: 0.004
  timestamp: 1552506033
  event_id: 431a0085-96da-4521-863f-c38b480701e9
{{< /code >}}

{{< code json >}}
{
  "type": "Event",
  "api_version": "core/v2",
  "metadata": {
    "namespace": "default"
  },
  "spec": {
    "check": {
      "check_hooks": null,
      "command": "/opt/sensu-plugins-ruby/embedded/bin/metrics-curl.rb -u \"http://localhost\"",
      "duration": 0.060790838,
      "env_vars": null,
      "executed": 1552506033,
      "handlers": [],
      "high_flap_threshold": 0,
      "history": [
        {
          "executed": 1552505833,
          "status": 0
        },
        {
          "executed": 1552505843,
          "status": 0
        }
      ],
      "interval": 10,
      "is_silenced": false,
      "issued": 1552506033,
      "last_ok": 1552506033,
      "low_flap_threshold": 0,
      "metadata": {
        "name": "curl_timings",
        "namespace": "default"
      },
      "occurrences": 1,
      "occurrences_watermark": 1,
      "output": "sensu-go-sandbox.curl_timings.time_total 0.005 1552506033\nsensu-go-sandbox.curl_timings.time_namelookup 0.004",
      "output_metric_format": "graphite_plaintext",
      "output_metric_handlers": [
        "influx-db"
      ],
      "proxy_entity_name": "",
      "publish": true,
      "round_robin": false,
      "runtime_assets": [],
      "state": "passing",
      "status": 0,
      "stdin": false,
      "subdue": null,
      "subscriptions": [
        "entity:sensu-go-sandbox"
      ],
      "timeout": 0,
      "total_state_change": 0,
      "ttl": 0
    },
    "entity": {
      "deregister": false,
      "deregistration": {},
      "entity_class": "agent",
      "last_seen": 1552495139,
      "metadata": {
        "name": "sensu-go-sandbox",
        "namespace": "default"
      },
      "redact": [
        "password",
        "passwd",
        "pass",
        "api_key",
        "api_token",
        "access_key",
        "secret_key",
        "private_key",
        "secret"
      ],
      "subscriptions": [
        "entity:sensu-go-sandbox"
      ],
      "system": {
        "arch": "amd64",
        "hostname": "sensu-go-sandbox",
        "network": {
          "interfaces": [
            {
              "addresses": [
                "127.0.0.1/8",
                "::1/128"
              ],
              "name": "lo"
            },
            {
              "addresses": [
                "10.0.2.15/24",
                "fe80::5a94:f67a:1bfc:a579/64"
              ],
              "mac": "08:00:27:8b:c9:3f",
              "name": "eth0"
            }
          ]
        },
        "os": "linux",
        "platform": "centos",
        "platform_family": "rhel",
        "platform_version": "7.5.1804",
        "processes": null
      },
      "user": "agent"
    },
    "metrics": {
      "handlers": [
        "influx-db"
      ],
      "points": [
        {
          "name": "sensu-go-sandbox.curl_timings.time_total",
          "tags": [],
          "timestamp": 1552506033,
          "value": 0.005
        },
        {
          "name": "sensu-go-sandbox.curl_timings.time_namelookup",
          "tags": [],
          "timestamp": 1552506033,
          "value": 0.004
        }
      ]
    },
    "timestamp": 1552506033,
    "event_id": "431a0085-96da-4521-863f-c38b480701e9"
  }
}
{{< /code >}}

{{< /language-toggle >}}

### Example metric-only event

{{< language-toggle >}}

{{< code yml >}}
type: Event
api_version: core/v2
metadata:
  namespace: default
spec:
  entity:
    deregister: false
    deregistration: {}
    entity_class: agent
    last_seen: 1552495139
    metadata:
      name: sensu-go-sandbox
      namespace: default
    redact:
    - password
    - passwd
    - pass
    - api_key
    - api_token
    - access_key
    - secret_key
    - private_key
    - secret
    subscriptions:
    - entity:sensu-go-sandbox
    system:
      arch: amd64
      hostname: sensu-go-sandbox
      network:
        interfaces:
        - addresses:
          - 127.0.0.1/8
          - ::1/128
          name: lo
        - addresses:
          - 10.0.2.15/24
          - fe80::5a94:f67a:1bfc:a579/64
          mac: 08:00:27:8b:c9:3f
          name: eth0
      os: linux
      platform: centos
      platform_family: rhel
      platform_version: 7.5.1804
      processes: null
    user: agent
  metrics:
    handlers:
    - influx-db
    points:
    - name: sensu-go-sandbox.curl_timings.time_total
      tags: []
      timestamp: 1552506033
      value: 0.005
    - name: sensu-go-sandbox.curl_timings.time_namelookup
      tags: []
      timestamp: 1552506033
      value: 0.004
  timestamp: 1552506033
  event_id: 47ea07cd-1e50-4897-9e6d-09cd39ec5180
{{< /code >}}

{{< code json >}}
{
  "type": "Event",
  "api_version": "core/v2",
  "metadata": {
    "namespace": "default"
  },
  "spec": {
    "entity": {
      "deregister": false,
      "deregistration": {},
      "entity_class": "agent",
      "last_seen": 1552495139,
      "metadata": {
        "name": "sensu-go-sandbox",
        "namespace": "default"
      },
      "redact": [
        "password",
        "passwd",
        "pass",
        "api_key",
        "api_token",
        "access_key",
        "secret_key",
        "private_key",
        "secret"
      ],
      "subscriptions": [
        "entity:sensu-go-sandbox"
      ],
      "system": {
        "arch": "amd64",
        "hostname": "sensu-go-sandbox",
        "network": {
          "interfaces": [
            {
              "addresses": [
                "127.0.0.1/8",
                "::1/128"
              ],
              "name": "lo"
            },
            {
              "addresses": [
                "10.0.2.15/24",
                "fe80::5a94:f67a:1bfc:a579/64"
              ],
              "mac": "08:00:27:8b:c9:3f",
              "name": "eth0"
            }
          ]
        },
        "os": "linux",
        "platform": "centos",
        "platform_family": "rhel",
        "platform_version": "7.5.1804",
        "processes": null
      },
      "user": "agent"
    },
    "metrics": {
      "handlers": [
        "influx-db"
      ],
      "points": [
        {
          "name": "sensu-go-sandbox.curl_timings.time_total",
          "tags": [],
          "timestamp": 1552506033,
          "value": 0.005
        },
        {
          "name": "sensu-go-sandbox.curl_timings.time_namelookup",
          "tags": [],
          "timestamp": 1552506033,
          "value": 0.004
        }
      ]
    },
    "timestamp": 1552506033,
    "event_id": "47ea07cd-1e50-4897-9e6d-09cd39ec5180"
  }
}
{{< /code >}}

{{< /language-toggle >}}


[1]: ../../observe-schedule/checks/
[2]: ../../observe-entities/entities#entities-specification
[3]: ../../observe-entities/entities/
[4]: ../#status-only-events
[5]: ../#metrics-only-events
[6]: ../../observe-schedule/collect-metrics-with-checks/
[7]: ../../observe-schedule/checks/#check-specification
[8]: ../../../sensuctl/create-manage-resources/#create-resources
[9]: #spec-attributes
[10]: ../../observe-schedule/agent#create-observability-events-using-service-checks
[11]: ../../observe-schedule/agent#create-observability-events-using-the-agent-api
[12]: ../../observe-schedule/agent#create-observability-events-using-the-agent-tcp-and-udp-sockets
[13]: ../../observe-schedule/agent#create-observability-events-using-the-statsd-listener
[14]: ../../../api/events#eventsentitycheck-put
[15]: ../../../web-ui/
[16]: ../../../api/events/
[17]: ../../../sensuctl/
[18]: ../../../sensuctl/create-manage-resources/#sensuctl-event
[19]: ../#status-and-metrics-events
[20]: ../../observe-schedule/checks#check-specification
[21]: #check-attributes
[22]: #metrics
[23]: ../../observe-filter/reduce-alert-fatigue/
[24]: ../../observe-filter/filters/
[25]: ../../observe-schedule/checks/#check-result-specification
[26]: ../../../operations/control-access/rbac#namespaces
[27]: ../../observe-filter/filters/#handle-state-change-only
[28]: ../../observe-filter/filters/#handle-repeated-events
[29]: #metadata-attributes
[30]: #metric-attributes
[31]: #use-event-data
[32]: #history-attributes
[33]: ../../observe-schedule/checks#spec-attributes
[34]: #points-attributes
[35]: ../../../api/events#create-a-new-event
[36]: #state-attribute
[37]: ../../observe-schedule/checks/#flap-thresholds
[38]: https://assets.nagios.com/downloads/nagioscore/docs/nagioscore/3/en/flapping.html
[39]: #flap-detection-algorithm
[40]: ../../observe-filter/filters/#check-attributes-available-to-filters