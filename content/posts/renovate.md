+++
date = '2026-03-27T15:14:43+01:00'
title = 'Renovate in Gitlab'
[params]
    description = ''
+++

# Renovate Bot

...

## Gitlab Templates

```yaml
stages:
- test
- deploy
- rotate-token

include:
  - project: 'renovate-bot/renovate-runner'
    file: 
      - '/templates/renovate.gitlab-ci.yml'
      - '/templates/renovate-config-validator.gitlab-ci.yml'

renovate-config-validator:
  stage: test
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

## Token Rotation

After each run, the Renovate token is being rotated.

The following tokens are required:
* TOKEN_WITH_MAINTAINER_PERMISSIONS
* RENOVATE_TOKEN

| NAME                                | SCOPES                                | ROLES      | CONTEXT             | DESCRIPTION                         |
|-------------------------------------|---------------------------------------|------------|---------------------|-------------------------------------|
| `RENOVATE_TOKEN`                    | `api`, `self_rotate`, `read_registry` | DEVELOPER  | group token | Access repositories, create MR, etc |
| `TOKEN_WITH_MAINTAINER_PERMISSIONS` | `api`, `self_rotate`                  | MAINTAINER | project token       | Update tokens                       |


```yaml
rotate-token:
  image: alpine/curl:8.12.0
  stage: rotate-token
  when: always
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
  variables:
    GROUP_ID: "<GROUP_ID>"
  before_script:
    - apk add --no-cache jq
  script:
    - |-
      set -eu
      function send_request() {
        curl --silent --show-error --fail-with-body "$@"
      }
      
      function rotate_token() {
        # TOKEN_SCOPE = { "groups", "projects" }
        # ID = { GROUP_ID, PROJECT_ID }
        local OLD_TOKEN="$1"
        local TOKEN_SCOPE="$2"
        local ID="$3"
        local EXPIRY_DATE=$(date -I -d@"$(( `date +%s`+10*24*60*60))")
        local PAYLOAD="{\"expires_at\":\"$EXPIRY_DATE\"}"

        send_request --request POST --header "Authorization: Bearer $OLD_TOKEN" "https://gitlab.com/api/v4/$TOKEN_SCOPE/$ID/access_tokens/self/rotate" --json "$PAYLOAD" \
          | jq -r .token
      }
      
      function update_project_variable() {
        local PROJECT_ID="$1"
        local TOKEN_WITH_MAINTAINER_PERMISSIONS="$2"
        local NEW_VALUE="$3"
        local VARIABLE_NAME="$4"
        local PAYLOAD="{\"value\":\"$NEW_VALUE\",\"masked\":true,\"masked_and_hidden\":true,\"protected\":false}"
      
        send_request --request PUT --header "Authorization: Bearer $TOKEN_WITH_MAINTAINER_PERMISSIONS" "https://gitlab.com/api/v4/projects/$PROJECT_ID/variables/$VARIABLE_NAME" --json "$PAYLOAD"
      }
      
      function main() {
        echo "[*] Rotating project level access token (TOKEN_WITH_MAINTAINER_PERMISSIONS)..."
        local ROTATED_TOKEN_WITH_MAINTAINER_PERMISSIONS
        ROTATED_TOKEN_WITH_MAINTAINER_PERMISSIONS=$(rotate_token "$TOKEN_WITH_MAINTAINER_PERMISSIONS" "projects" "$CI_PROJECT_ID")
        echo "[+] renovate-bot project access token rotated (TOKEN_WITH_MAINTAINER_PERMISSIONS)."
      
        echo "[*] Updating TOKEN_WITH_MAINTAINER_PERMISSIONS variable with new value..."
        update_project_variable "$CI_PROJECT_ID" "$ROTATED_TOKEN_WITH_MAINTAINER_PERMISSIONS" "$ROTATED_TOKEN_WITH_MAINTAINER_PERMISSIONS" "TOKEN_WITH_MAINTAINER_PERMISSIONS"
        echo "[+] TOKEN_WITH_MAINTAINER_PERMISSIONS variable updated."
        
        echo "================================================================"
      
        echo "[*] Rotating group level token (RENOVATE_TOKEN)..."
        local ROTATED_RENOVATE_TOKEN
        ROTATED_RENOVATE_TOKEN=$(rotate_token "$RENOVATE_TOKEN" "groups" "$GROUP_ID")
        echo "[+] renovate-bot group access token rotated (RENOVATE_TOKEN)."
      
        echo "[*] Updating group level variable with new value (RENOVATE_TOKEN)..."
        update_project_variable "$CI_PROJECT_ID" "$ROTATED_TOKEN_WITH_MAINTAINER_PERMISSIONS" "$ROTATED_RENOVATE_TOKEN" "RENOVATE_TOKEN"
        echo "[+] RENOVATE_TOKEN variable updated."
      }
      
      main

```

## Renovate Configuration

File: config.js

```js
module.exports = {
    "$schema": "https://docs.renovatebot.com/renovate-schema.json",
    "description": "The default configuration for Renovate",
    "platform": "gitlab",
    "gitAuthor": "Renovate Bot <renovate-bot@renovate.io>",
    "ignorePrAuthor": true,
    "extends": [
        "gitlab>path/repo" // default.json
    ],
    "commitMessagePrefix": "RNVT: ",
    "onboardingCommitMessage": "RNVT: Onboarding",
    "autodiscover": true,
    "autodiscoverFilter": [
        // ...
    ],
    "onboarding": true,
    "onboardingConfig": {
        "extends": [
            "gitlab>path/repo"
        ]
    },
    "automerge": false,
    "semanticCommits": "enabled",
    "dependencyDashboardOSVVulnerabilitySummary": "all",
    "branchPrefix": "renovate/",
    "osvVulnerabilityAlerts": true,
    "addLabels": [
        "renovate"
    ],
    "vulnerabilityAlerts": {
        "labels": [
            "security"
        ]
    }
}
```

File: default.json

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "description": "The default onboarding configuration for Renovate",
  "extends": [
    "config:best-practices",
    ":disableDependencyDashboard",
    "mergeConfidence:all-badges",
    "security:minimumReleaseAgeNpm",
    "group:allNonMajor"
  ],
  "packageRules": [
    {"matchCategories": ["python"], "addLabels": ["lang: python"]},
    {"matchCategories": ["java"], "addLabels": ["lang: java"]},
    {"matchCategories": ["golang"], "addLabels": ["lang: go"]},
    {"matchCategories": ["js"], "addLabels": ["lang: nodejs"]},
    {
      "managers": ["maven"],
      "packageNames": ["com.google.guava:guava"],
      "allowedVersions": "/.*-jre$/"
    },
    {
      "matchManagers": ["terraform"],
      "matchDepTypes": ["module"],
      "groupName": "terraform modules",
      "matchUpdateTypes": ["minor", "patch"]
    }
  ],
  "baseBranchPatterns": ["master"],
  "reviewersFromCodeOwners": true,
  "assigneesFromCodeOwners": true,
  "minimumReleaseAge": "14 days"
}
```

# References

* https://gitlab.com/renovate-bot/renovate-runner
* https://docs.renovatebot.com/key-concepts/minimum-release-age/