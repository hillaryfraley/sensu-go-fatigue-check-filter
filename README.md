# sensu-go-fatigue-check-filter
TravisCI: [![TravisCI Build Status](https://travis-ci.org/nixwiz/sensu-go-fatigue-check-filter.svg?branch=master)](https://travis-ci.org/nixwiz/sensu-go-fatigue-check-filter)

The Sensu Go Fatigue Check Filter is a [Sensu Event Filter][1] for managing alert fatigue.

A typical use of filters is to reduce [alert fatigue][2].  One of the most typical examples of this is create the following filter that only passes through events on their first occurrence and every hour after that.

```json
{
  "type": "EventFilter",
  "api_version": "core/v2",
  "metadata": {
    "name": "hourly",
    "namespace": "default"
  },
  "spec": {
    "action": "allow",
    "expressions": [
      "event.check.occurrences == 1 || event.check.occurrences % (3600 / event.check.interval) == 0"
    ],
    "runtime_assets": []
  }
}
```

However, the use of the filter above creates some limitations.  Suppose you have one check in particular that you want to change to only alert after three (3) occurrences.  Typically that might mean creating another handler and filter pair to assign to that check.  If you have to do this often enough and you start to have an unwieldy mass of handlers and filters.

That's where this Fatigue Check Filter comes in.  Using annotations, it makes the number of occurrences and the interval tunable on a per-check or per-entity basis.  It also allows you to control whether or not resolution events are passed through.

## Installation

Use asset from [Bonsai][4] with `sensuctl asset add nixwiz/sensu-go-fatigue-check-filter --rename fatigue-check-filter`.  Please note this requires Sensu 5.13 or later.  Also note that the --rename is not necessary, but references to the runtime asset in the filter definition as in the example below would need to be updated to match. 

You can create your own [asset][3] by creating a tar file containing `lib/fatigue_check.js` and creating your asset definition accordingly.

## Configuration

The Fatigue Check Filter makes use of four annotations within the check and/or entity metadata, with the entity annotations taking precedence.

|Annotation|Default|Usage|
|----------|-------|-----|
|fatigue_check/occurrences|1|On which occurrence to allow the initial event to pass through|
|fatigue_check/interval|1800|In seconds, at what interval to allow subsequent events to pass through, ideally a multiple of the check interval|
|fatigue_check/allow_resolution|true|Determines whether or not a resolution event is passed through|
|fatigue_check/suppress_flapping|true|Determines whether or not to suppress events for checks that are marked as flapping|

**Notes:**
* This filter makes use of the occurrences_watermark attribute that was buggy up until Sensu Go 5.9.  Your mileage may vary on prior versions.
* If the interval is not a multiple of the check's interval, then the actual interval is computed by rounding up the result of dividing the interval by the check's interval.  For example, an interval of 180s with a check interval of 25s would pass the event through on every 8 occurrences (200s).

#### Definition Examples
Asset (if not using `sensuctl asset add`):
```json
{
  "type": "Asset",
  "api_version": "core/v2",
  "metadata": {
    "name": "fatigue-check-filter",
    "namespace": "default"
  },
  "spec": {
    "sha512": "2e67975df7d993492cd5344edcb9eaa23b38c1eef7000576b396804fc2b33362b02a1ca2f7311651c175c257b37d8bcbbce1e18f6dca3ca04520e27fda552856",
    "url": "http://example.com/sensu/assets/fatigue-check.tar.gz"
  }
}
```
Filter:

```json
{
  "type": "EventFilter",
  "api_version": "core/v2",
  "metadata": {
    "name": "fatigue_check",
    "namespace": "default"
  },
  "spec": {
    "action": "allow",
    "expressions": [
      "fatigue_check(event)"
    ],
    "runtime_assets": [
      "fatigue-check-filter"
    ]
  }
}
```

Handler:

```json
{
    "api_version": "core/v2",
    "type": "Handler",
    "metadata": {
        "namespace": "default",
        "name": "email"
    },
    "spec": {
        "type": "pipe",
        "command": "sensu-email-handler -f from@example.com -t to@example.com -s smtp.example.com -u emailuser -p sup3rs3cr3t",
        "timeout": 10,
        "filters": [
            "is_incident",
            "not_silenced",
            "fatigue_check"
        ]
    }
}
```
Check:
```json
{
  "type": "CheckConfig",
  "api_version": "core/v2",
  "metadata": {
    "name": "linux-cpu-check",
    "namespace": "default",
    "annotations": {
      "fatigue_check/occurrences": "3",
      "fatigue_check/interval": "900",
      "fatigue_check/allow_resolution": "false"
    }
  },
  "spec": {
    "command": "check-cpu -w 90 c 95",
    "env_vars": null,
    "handlers": [
      "email"
    ],
    "high_flap_threshold": 0,
    "interval": 60,
    "low_flap_threshold": 0,
    "output_metric_format": "",
    "output_metric_handlers": null,
    "proxy_entity_name": "",
    "publish": true,
    "round_robin": false,
    "runtime_assets": null,
    "stdin": false,
    "subdue": null,
    "subscriptions": [
      "linux"
    ],
    "timeout": 0,
    "ttl": 0
  }
}

```

Entity (via the agent.yml):
```
---
##
# agent configuration
##

#name: ""

#namespace: "default"

#subscriptions: 
#  - "localhost"

annotations:
  fatigue_check/occurrences: "3"
  fatigue_check/interval: "900"
  fatigue_check/allow_resolution: "false"

[...]
```

[1]: https://docs.sensu.io/sensu-go/latest/reference/filters/
[2]: https://docs.sensu.io/sensu-go/latest/guides/reduce-alert-fatigue/
[3]: https://docs.sensu.io/sensu-go/latest/reference/assets/
[4]: https://bonsai.sensu.io/
