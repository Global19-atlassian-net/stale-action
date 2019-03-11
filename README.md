![Build Status](https://action-badges.now.sh/probot/stale-action)

# Stale-Action

This action is adapted from Probot's [Stale app](https://github.com/probot/stale), built by [@bkeepers](https://github.com/bkeepers), which itself was onspired by @parkr's [auto-reply](https://github.com/parkr/auto-reply#optional-mark-and-sweep-stale-issues) bot that runs @jekyllbot.

It uses [Actions-Toolkit](https://github.com/JasonEtco/actions-toolkit) by [@JasonEtco](https://github.com/JasonEtco) and is built to be scheduled by triggering a `repository_dispatch` HTTP event from an external service, allowing you to customize how often to scan for stale content.

## Usage

1. Create `.github/stale.yml` based on the following template.
2. Create a `main.workflow` based off the [`example.workflow`](./example.workflow) in this repository, which will start a scan in response to a [`repository_dispatch`](https://developer.github.com/actions/creating-workflows/triggering-a-repositorydispatch-webhook/) event.

A `.github/stale.yml` file is required to enable the plugin. The file can be empty, or it can override any of these default settings:

```yml
# Configuration for probot-stale - https://github.com/probot/stale

# Number of days of inactivity before an Issue or Pull Request becomes stale
daysUntilStale: 60

# Number of days of inactivity before an Issue or Pull Request with the stale label is closed.
# Set to false to disable. If disabled, issues still need to be closed manually, but will remain marked as stale.
daysUntilClose: 7

# Issues or Pull Requests with these labels will never be considered stale. Set to `[]` to disable
exemptLabels:
  - pinned
  - security
  - "[Status] Maybe Later"

# Set to true to ignore issues in a project (defaults to false)
exemptProjects: false

# Set to true to ignore issues in a milestone (defaults to false)
exemptMilestones: false

# Set to true to ignore issues with an assignee (defaults to false)
exemptAssignees: false

# Label to use when marking as stale
staleLabel: wontfix

# Comment to post when marking as stale. Set to `false` to disable
markComment: >
  This issue has been automatically marked as stale because it has not had
  recent activity. It will be closed if no further activity occurs. Thank you
  for your contributions.

# Comment to post when removing the stale label.
# unmarkComment: >
#   Your comment here.

# Comment to post when closing a stale Issue or Pull Request.
# closeComment: >
#   Your comment here.

# Limit the number of actions per hour, from 1-30. Default is 30
limitPerRun: 30

# Limit to only `issues` or `pulls`
# only: issues

# Optionally, specify configuration settings that are specific to just 'issues' or 'pulls':
# pulls:
#   daysUntilStale: 30
#   markComment: >
#     This pull request has been automatically marked as stale because it has not had
#     recent activity. It will be closed if no further activity occurs. Thank you
#     for your contributions.

# issues:
#   exemptLabels:
#     - confirmed
```

### YAML Validation
Thanks to the ability to integrate an Action with [Check Runs](https://developer.github.com/v3/checks/runs/), this also makes use of [Joi](https://github.com/hapijs/joi) and [line-column](https://github.com/io-monad/line-column) to validate your YAML file against the [schema](./lib/schema.js). If one of the values in your `stale.yml` values is invalid, you should get an annotation on the commit that looks something like this:

![image](https://user-images.githubusercontent.com/13207348/54105958-eb638e80-4391-11e9-96b9-192ba69cfd52.png)

Right now it will only send annotations for values that don't match the schema. If you have bad YAML syntax, the config will fail to load and you should see relevant errors in the Actions logs.


### Creating a `.workflow` file
The `main.workflow` file contains multiple `on` events. The main one that triggers a scan for stale content is `on = "repository_dispatch"`. The others are events that listen for activity on issues and pull requests to unmark them if they were previously marked as stale.

```hcl
workflow "Run Stale!" {
  on = "repository_dispatch"
  resolves = ["probot/stale-action@master"]
}

action "probot/stale-action@master" {
  uses = "probot/stale-action@master"
  secrets = ["GITHUB_TOKEN"]
}

workflow "On issue comments" {
  on = "issue_comment"
  resolves = ["probot/stale-action@master - issue_comment"]
}

action "probot/stale-action@master - issue_comment" {
  uses = "probot/stale-action@master"
  secrets = ["GITHUB_TOKEN"]
}

workflow "On pull request" {
  on = "pull_request"
  resolves = ["probot/stale-action@master - PR"]
}

action "probot/stale-action@master - PR" {
  uses = "probot/stale-action@master"
  secrets = ["GITHUB_TOKEN"]
}
```

The `repository_dispatch` event can be triggered manually whenever you want to kick off a scan. An example with `curl` would look something like this, and require a token with write access to the repo you want to trigger:

```sh
curl -X POST https://api.github.com/repos/:owner/:repo/dispatches \
  -H 'Accept: application/vnd.github.everest-preview+json'
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: application/json' \
  -d '{ "event_type": "stale" }'
```

There are a number of ways to setup a scheduled trigger for the repository dispatch event. The easiest one I found was an [Azure job scheduler](https://azure.microsoft.com/en-us/services/scheduler/) setup to fire the authenticated HTTP trigger once a day.

If you use Azure and want to deploy something similar, here's the important bits you'll need in an Azure Resource Group Template. For the Authorization token you'll need to use a Personal Access Token that has write access to the repository you want to trigger:

<details>

```json
{
  "comments": "Generalized from resource...",
  "type": "Microsoft.Scheduler/jobCollections/jobs",
  "name": "Stale Action HTTP trigger",
  "apiVersion": "2016-03-01",
  "scale": null,
  "properties": {
      "startTime": "2019-02-27T07:27:30.678Z",
      "action": {
          "request": {
              "uri": "https://api.github.com/repos/:owner/:repo/dispatches",
              "method": "POST",
              "headers": {
                  "Authorization": "Bearer <token>",
                  "Accept": "application/vnd.github.everest-preview+json",
                  "Content-Type": "application/json",
                  "User-Agent": "Stale from Azure"
              },
              "body": "{\n  \"event_type\": \"stale\"\n}"
          },
          "type": "HTTPS",
          "retryPolicy": {
              "retryType": "fixed",
              "retryInterval": "30.00:00:00",
              "retryCount": 3
          }
      },
      "recurrence": {
          "frequency": "day",
          "count": 7,
          "interval": 1
      },
      "state": "completed"
  }
}
```

</details>

## License

[ISC](LICENSE) Copyright © 2019 Tommy Byrd
