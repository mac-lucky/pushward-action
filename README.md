# PushWard action

Send [PushWard](https://pushward.app) push notifications from a workflow, show a job as a Live Activity on your Lock Screen, update home screen widgets, send email, and stop a job until someone taps a button on their phone.

It runs [pushward-cli](https://github.com/mac-lucky/pushward-cli), so anything the CLI can do works here through the `command` input.

## Setup

Copy an integration key (`hlk_...`) from the PushWard app and save it as a repository secret, say `PUSHWARD_TOKEN`. A key just for CI is better than your default one: give it only the scopes the workflow uses, and restrict its activity slugs to `gha-*`.

## Notify when a job fails

```yaml
- uses: mac-lucky/pushward-action@v1
  if: failure()
  with:
    token: ${{ secrets.PUSHWARD_TOKEN }}
    title: ${{ github.workflow }} failed
    body: ${{ github.repository }} on ${{ github.ref_name }}
    level: time-sensitive
```

Tapping it opens the run. Notifications from one repository are grouped into a thread.

## A job as a Live Activity

```yaml
steps:
  - uses: mac-lucky/pushward-action@v1
    with:
      token: ${{ secrets.PUSHWARD_TOKEN }}
      command: activity start --template steps --step 1/3 --step-labels Build,Test,Deploy
      text: Building

  - run: make build

  - uses: mac-lucky/pushward-action@v1
    with:
      token: ${{ secrets.PUSHWARD_TOKEN }}
      command: activity update --step 2/3
      text: Testing

  - run: make test

  - uses: mac-lucky/pushward-action@v1
    with:
      token: ${{ secrets.PUSHWARD_TOKEN }}
      command: activity update --step 3/3
      text: Deploying

  - run: make deploy

  - uses: mac-lucky/pushward-action@v1
    if: always()
    with:
      token: ${{ secrets.PUSHWARD_TOKEN }}
      command: activity end
      status: ${{ job.status }}
```

The card is named after the workflow, shows `repo / job` underneath and links to the run. `activity end` shows a green check, red cross or grey stop for a few seconds, then ends the card. If the job failed before the start step ran, the end step only logs a warning.

The slug defaults to `gha-<owner>-<repo>-<run_id>-<job>`. Matrix legs share the job id, so give each leg its own:

```yaml
    with:
      slug: gha-${{ github.run_id }}-${{ strategy.job-index }}
```

## Wait for an answer

```yaml
- id: ask
  uses: mac-lucky/pushward-action@v1
  with:
    token: ${{ secrets.PUSHWARD_TOKEN }}
    title: Deploy ${{ github.ref_name }} to production?
    body: ${{ github.event.head_commit.message }}
    actions: |
      deploy=Deploy
      skip=Skip
    wait: 30m

- if: steps.ask.outputs.answer == 'deploy'
  run: ./deploy.sh
```

If nobody answers in time the step fails. With `fail-on-error: false` it passes instead, `answer` is empty and `status` is `pending`. The runner is billed the whole time it waits, so for anything longer than a few minutes a protected environment with required reviewers is the cheaper gate.

The same works with an approval card on the Lock Screen: `command: activity start --template approval`, `actions` as its options, then `command: activity wait` with `wait: 30m`.

## Update a widget

```yaml
- uses: mac-lucky/pushward-action@v1
  with:
    token: ${{ secrets.PUSHWARD_TOKEN }}
    command: widget update
    slug: coverage
    fields: content.value=${{ steps.coverage.outputs.percent }}
```

Create the widget once, from the app or with `command: widget create --template gauge --min 0 --max 100 --unit %`.

## Send an email

```yaml
- uses: mac-lucky/pushward-action@v1
  with:
    token: ${{ secrets.PUSHWARD_TOKEN }}
    command: email send --to ops@example.com --subject "Nightly report"
    text: ${{ steps.report.outputs.summary }}
```

Email only goes to addresses you have verified in the app.

## Inputs

`token` is the only required input. `command` defaults to `notify` and takes any pushward command with its flags; the [CLI README](https://github.com/mac-lucky/pushward-cli#readme) lists them. The other inputs are shortcuts for the common flags: `title`, `body`, `subtitle`, `level`, `url`, `slug`, `name`, `template`, `text`, `progress`, `icon`, `color`, `status`, `wait`, `actions` and `json`. `fields` takes typed `key=value` lines with dotted keys, for anything without a shortcut. An input the command has no use for is ignored with a warning.

Keep untrusted text (branch names, commit messages, PR titles) in those inputs, never in `command`. `command` is split like a shell line, so a crafted value there can add flags.

A failed call fails the step with an error annotation. `fail-on-error: false` turns that into a warning.

## Outputs

| Output | Value |
|---|---|
| `response` | the API response as JSON |
| `id` | notification id |
| `slug` | activity or widget slug |
| `answer` | id of the tapped action, or the chosen option of an approval card |
| `answer-text` | text typed with the answer |
| `status` | notification delivery (`all`, `partial`, `none`), activity state (`ongoing`, `ended`) or answer status (`pending`, `answered`) |

## Notes

This is a Docker container action, so it runs on Linux runners only. On macOS or Windows runners, install the CLI (`brew install mac-lucky/tap/pushward`, or a release archive) and call it from a `run` step.

If the PushWard GitHub bridge also watches the repository, each run gets two cards, the bridge's and this one. Use one or the other.

`@v1` follows the newest 1.x release. Every release pins an exact pushward-cli image, so pinning `@v1.2.3` or a commit SHA also pins the CLI.

## License

MIT
