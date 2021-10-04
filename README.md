# zendesk-integration

The zendesk github integration consist of 3 github actions.

- [issue_created.yml](blob/main/.github/workflows/issue_created.yml)
- [issue_commented.yml](blob/main/.github/workflows/issue_commented.yml)
- [zendesk_commented.yml](blob/main/.github/workflows/zendesk_commented.yml)

In addition, within zendesk there needs to be a [Webhook](https://developer.zendesk.com/api-reference/event-connectors/webhooks/webhooks/#create-or-clone-webhook) and a [Trigger](https://developer.zendesk.com/api-reference/ticketing/business-rules/triggers/#create-trigger) configured.

## Github actions

### Issue created

Once an issue is created it send the issue to zendesk,
creating (or using) the user `<Github UserName>` with email `<Github UserName>@users.noreply.github.com`.
This mail adress format is also used by github when a user sets the privacy option that the mail should be hidden.
Once the ticket got created a comment to the issue will be added to inform the user about the tech support contact.

### Issue commented

Every comment on an issue gets send to zendesk. It will look up the connected ticket and author and attaches the
comment to the ticket, There is no additional comment added on the issue to avoid noise.

### Zendesk commented

Once a comment on zendesk was added to a ticket from github it will send a webhook to github which triggers a
github action to sync the zendesk comment back to the issue.


## Zendesk setup

### Webhook

To be able to receive comments from zendesk and sync them back to the github issue a webhook needs to be created within zendesk.
This can either be done through their interface or through the api. The following data is required:

```json
{
  "webhook": {
    "name": "Github Issue Integration",
    "subscriptions": ["conditional_ticket_events"],
    "endpoint": "https://api.github.com/repos/<GITHUB_ORG>/<GITHUB_REPO>/dispatches",
    "http_method": "POST",
    "request_format": "json",
    "authentication": {
      "type": "bearer_token",
      "add_position": "header",
      "data":{
        "token":"<TOKEN>"
      }
    }
  }
}
```

`<GITHUB_ORG>`, `<GITHUB_REPO>` and `<TOKEN>` needs to be replaced with real data.

### Trigger

The webhook itself does not get triggered automatically in zendesk. For this, a `Trigger` needs to be defined.

```json
{
  "trigger": {
    "title": "Zendesk comments to Github",
    "actions": [
      {
        "field": "notification_webhook",
        "value": [
          "<ZENDESK_WEBHOOK_ID>",
          "{\"event_type\": \"zendesk-comment\", \"client_payload\": { \"ticket\": {\"id\": \"{{ticket.id}}\",\"external_id\": \"{{ticket.external_id}}\"},\"comment\": {\"body\": \"{{ticket.latest_comment.value}}\",\"author\": \"{{ticket.latest_comment.author.name}}\"}}}"
        ]
      }
    ],
    "conditions": {
      "all": [
        {
          "field": "comment_is_public",
          "operator": "is",
          "value": true
        },
        {
          "field": "subject_includes_word",
          "operator": "includes",
          "value": "Github_Issue"
        },
        {
          "field": "comment_includes_word",
          "operator": "not_includes",
          "value": "GITHUB_ISSUE_COMMENT"
        }
      ],
      "any": []
    },
    "description": "#zendesk-github-comments",
  }
}
```

`<ZENDESK_WEBHOOK_ID>` needs to be replaced with the real ID of the new generated webhook.
Additional conditions can be set to e.g. limit tickets to specific organizations or categories.

:warning: Only public comments should be syned to github. If `comment_is_public` is left out (or set to `"not_relevant"`)
private comments would also be synced to github and made public.

:warning: The conditions `subject_includes_word` and `comment_includes_word` matches words added by the github actions.
They should not be removed but can be changed to anything - just make sure to also adjust the github actions accordingly.

## Integration

One important notice: This setup requires only one *global* `zendesk_commented` github action (only once) while the
`issue_commented` and `issue_created` github actions needs to be added to every repository where the issues should
get synced to zendesk.

The workflow can also be reused via [Calling a reusable workflow](https://docs.github.com/en/actions/learn-github-actions/reusing-workflows#calling-a-reusable-workflow)
to only have one place for the integration code.

```
on:
  issues:
    types: [opened]
jobs:
  issue_created:
    uses: <GITHUB_ORG>/<GITHUB_REPO>/.github/workflows/issue_created.yml@main
```


```
on:
  issue_comment:
    types: [created]
jobs:
  issue_created:
    uses: <GITHUB_ORG>/<GITHUB_REPO>/.github/workflows/issue_commented.yml@main
```


`<GITHUB_ORG>` and `<GITHUB_REPO>` needs to be replaced with real data.
