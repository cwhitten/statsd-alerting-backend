[![Build Status](https://travis-ci.org/joshgummersall/statsd-alerting-backend.svg?branch=master)](https://travis-ci.org/joshgummersall/statsd-alerting-backend)

statsd-alerting-backend
======================

A backend plugin for [Statsd](https://github.com/etsy/statsd/) to perform some
basic alerting on events.

## Overview

Suppose you wanted to use StatsD as your main event stream aggregator due
to its simple interface and its ease of integration with existing services.
Perfect! Now, what if you want to send out some alerts based on certain events?
Let's do it!

## Configuring Alerts

There are two different types of alerts that are supported. Those are "event"
alerts and "metric" alerts (see example configuration file for both). "Event"
alerts are things that you want to be alerted on immediately. An example of an
event you would likely want to be alerted on immediately is an uncaught
exception.

"Metrics" alerts are a bit more complicated. StatsD publishes aggregate metrics
at a configurable rate (default is every ten seconds). Perhaps you wanted to be
alerted when the 90th percentile average time for an event exceeds a certain
value. You can do that (check out the example configuration for how to set that
up).

This plugin is very young and I plan to include more sophisticated alerting
features as well as more alert types. Feel free to leave feedback via the
issues for things you would like to see added.

## Installation

In your StatsD installation folder, run:

```bash
$ npm install statsd-alerting-backend
```

Include the backend in your `config.js` file (see example configuration file
below for complete configuration example).

```json
{
  "backends": ["statsd-alerting-backend"]
}
```

## Development

This plugin is written in CoffeeScript that is compiled to Javascript
automatically when publishing to NPM (see `gulpfile.js` and `package.json` for
more details). To work on this plugin, simply clone the repository and run
`npm install`. I would suggest running `gulp watch` in a separate shell to
watch the source Coffeescript files for changes and automatically compile them
to Javascript files.

## Configuration

#### `dispatchers`

Here you can define a set of dispatchers to use when sending alerts. They can
be named anything (names must be unique as it is a Javascript object). See the
example config file for formatting. Each dispatcher must have a type that is
one of the valid alert dispatchers (currently "slack", "pagerduty", "hipchat",
and "log").
Each dispatcher must also include the necessary configuration for the alert
dispatcher.

##### Dispatcher Templates

You can specify a template to use for your alert messages if you'd like. If you
don't specify a template, a sane default will be used for all alerts. Templates
must be written using [Handlebars.js](http://handlebarsjs.com/) (really only the
basic templating features are useful here). An example template might look like:

```json
{
  "template": "Event alert for {{name}} ({{metric}} {{type}})"
}
```

Templates are defined in each dispatcher's configuration block. You can specify
a template for event alerts (using the `template` property) and for metric
alerts (using the `metricTemplate` property). The context used when rendering
templates is the event object itself. Each even contains a `name`, `metric`,
`type`, and some other metadata.

#### `slack` configuration

List your Slack incoming webhook configuration information. Required keys are
`host` and `token`. The username will default to "statsd-alerts" and the channel
will default to #alerts.

#### `pagerduty` configuration

Simply list your Pagerduty service key.

#### `sentry` configuration

Simply list your Sentry DSN.

#### `hipchat` configuration

List your Hipchat API `key` and the `room` name to which you want to send the
alerts. Optionally include a `from` description and a `notify` boolean.

Make sure that the key provided has enabled the scope `send_notification`.

#### `log` configuration

Target, one of `stdout` or `stderr`. Defaults to `stdout`.

#### `email` configuration

You must provide a `transport` key and a `mailOptions` key. See the
[nodemailer docs](https://github.com/nodemailer/nodemailer#tldr-usage-example)
for specifics.

`transport` can be a connection string or an object, see examples
[here](https://github.com/nodemailer/nodemailer#set-up-smtp).

`eventAlertSubject` is the email subject for an event-based alert.

`metricsEventAlertSubject` is the email subject for a metric event-based alert.

`mailOptions` controls how emails are delivered. Valid keys include `from` and
`to`. The `subject` is filled in using either `eventAlertSubject` or
`metricsEventAlertSubject`. The `text` will be the formatted event or metrics
event alert.

#### `events`

Think of this as a list of events (StatsD `counters` or `gauges`) that you want
to send alerts immediately as they are sent. A good example of this would be an
exception happening or a user signing up for something. You can use wildcards
in the event name (see [wildcard](https://www.npmjs.org/package/wildcard) for
formatting and matching information).

#### `metrics`

Think of this as a list of aggregate metrics you want to alert on. Typical uses
for this would be alerting when an average time is greater than a set value or
when the rate of counter exceeds some value. You could also alert when the
difference (or `delta`) of a current metric and its previous value exceeds a
certain value. See the [integration test](https://github.com/joshgummersall/statsd-alerting-backend/blob/master/test/integration_test.coffee)
for more information on how to use these alerts. You can also use wildcards
in the metrics name for matching. A wildcard will match any events of a specific
type (i.e. `timer_data` or `counter_rates`).

#### Supported Alert Types

Currently you can alert using [Slack](https://slack.com/), [HipChat](https://www.hipchat.com/),
[Pagerduty](http://www.pagerduty.com/), [Sentry](https://getsentry.com/welcome/)
or logging to stdout. Please ensure that you have the proper configuration
values for whichever alerting sources you specify in your configuration file.

## Example Configuration

```
{
  backends: ["statsd-alerting-backend"],

  alerts: {
    dispatchers: {
      slackDispatcher: {
        type: "slack",
        config: {
          webhook: "<INCOMING_WEBHOOK>",
          username: "statsd-alerts",
          channel: "#alerts",
          template: "Saw a {{name}} event!"
        }
      },

      pagerdutyDispatcher: {
        type: "pagerduty",
        config: {
          key: "<PAGERDUTY_SERVICE_KEY>"
        }
      },

      sentryDispatcher: {
        type: "sentry",
        config: {
          dsn: "<SENTRY_DSN>"
        }
      },

      hipchatDispatcher: {
        type: "hipchat",
        config: {
          key: "<HIPCHAT_API_KEY>",
          from: "statsd-hipchat-alerts".
          room: "<HIPCHAT_ROOM_NAME>",
          color: "red",
          notify: true,
          template: "Saw a {{name}} event!"
        }
      },

      logDispatcher: {
        type: "log",
        config: {
          target: "stdout"
        }
      },

      emailDispatcher: {
        type: "email",
        config: {
          transport: "smtps://user%40gmail.com:pass@smtp.gmail.com",
          eventAlertSubject: "Oops! Event Alert",
          metricsEventAlertSubject: "Oops! Metrics Event Alert",
          mailOptions: {
            from: "user@gmail.com",
            to: "alerts@mycompany.com"
          }
        }
      }
    },

    events: [{
      name: "some.event.*",
      dispatcher: "logDispatcher"
    }],

    metrics: [{
      name: "some.*.timer",
      type: "timer_data",
      key: "mean_90",
      delta: 10,
      dispatcher: "slackDispatcher"
    }, {
      name: "some.event.counter",
      type: "counter_rates",
      gte: 0.2,
      dispatcher: "pagerdutyDispatcher"
    }]
  }
}
```

## Contributing

Feel free to [leave issues here](https://github.com/joshgummersall/statsd-alerting-backend/issues)
or fork the project and submit pull requests. If there is a feature you would like added
just submit an issue describing the feature and I will do my best.
