---
layout: post
title: "Self-hosted Github Actions #1 – Hello world!"
date: 2022-02-14 00:00:00
categories:
 - blog
---

## Setting up actionsflow

Create a repo from the template https://github.com/actionsflow/actionsflow-workflow-default

Clone the repo

Start the docker container with the repo attached as a volume:
`docker run -it -v /var/run/docker.sock:/var/run/docker.sock -v <path_to_repo>:/data -p 3000:3000 --name actionsflow actionsflow/actionsflow`

## Define workflow

Create `workflows/test-repo.yml`:
```yml
on:
  webhook:
    config:
        skipSchedule: true
jobs:
  print:
    name: Print
    runs-on: ubuntu-latest
    steps:
      - name: Print outputs
        env:
          method: ${{on.webhook.outputs.method}}
          body: ${{toJson(on.webhook.outputs.body)}}
          headers: ${{toJson(on.webhook.outputs.headers)}}
        run: |
          echo method: $method
          echo headers $headers
          echo body: $body
```

Webhook is now available at `http://localhost:3000/webhook/test-repo/webhook`

## Prepare project repo

Set up git remote according to https://jasonmurray.org/posts/2020/selfhostedgit/

Go to remote's `<repo>/hooks/`

Facilitate `post-receive` hook https://git-scm.com/docs/git-receive-pack#_post_receive_hook

Create `push-webhook.sh`:
```bash
#!/bin/bash

WEBHOOK_URL="https://maxx23.duckdns.org/actionsflow/webhook/test-repo/webhook/"
read oldref newref refname

curl -v \
        -X POST \
        -H "Content-Type: application/json" \
        -d "{\"oldref\": \"${oldref}\", \"newref\": \"${newref}\", \"refname\": \"${refname}\"}" \
        $WEBHOOK_URL
```

Create `post-receive`:
```bash
#!/bin/bash

./push-webhook.sh 2>&1 | tee -a "$(date +'post-receive_%Y-%m-%d').log"
```