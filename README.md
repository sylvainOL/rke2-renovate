# rke2 and renovate

Simple repo to reproduce issue having retrieving the last value of rke2 version

Here's the `config.js` file I'm using if interesting (on a private gitlab instance...):

```js
module.exports = {
  "platform": "gitlab",
  "dependencyDashboardAutoclose": true,
  "repositoryCache": "enabled",
  "autodiscover": true,
  "extends": [
    "github>whitesource/merge-confidence:beta"
  ],
  "ignorePrAuthor": true,
  "onboardingConfig": { "extends": ["config:base", ":disableDependencyDashboard"] },
  repositories: ["sylvainOL/test-project"],
  hostRules: [
    {
      matchHost: 'docker.io',
      username: '******',
      password: process.env.DOCKER_HUB_PASSWORD,
    },
  ],
};
```

gitlab ci file:

```yaml
---
stages:
  - build
  - test
  - package-build
  - package-test
  - infra
  - deploy
  - acceptance
  - publish
  - infra-prod
  - production

###
# Variables
##

variables:
  ###
  # Variables for imported jobs
  ##

  # linting part
  PATH_EXCLUSION: './ci'
  JSON_PATH_EXCLUSION: "$PATH_EXCLUSION"
  MARKDOWN_PATH_EXCLUSION: "$PATH_EXCLUSION"
  MD_LINK_CHECK_PATH_EXCLUSION: "$PATH_EXCLUSION"
  SHELL_PATH_EXCLUSION: "$PATH_EXCLUSION"
  SPELLING_PATH_EXCLUSION: "$PATH_EXCLUSION"
  YAML_PATH_EXCLUSION: "$PATH_EXCLUSION"

  # to-be-continuous/semantic-release
  GITLAB_TOKEN: $BOT_TOKEN
  SEMREL_AUTO_RELEASE_ENABLED: "true"
  SEMREL_TAG_FORMAT: "$${version}"

  # renovate part
  RENOVATE_IMAGE: "renovate/renovate:latest"
  RENOVATE_BASE_DIR: $CI_PROJECT_DIR/renovate
  RENOVATE_ENDPOINT: $CI_API_V4_URL

  RENOVATE_LOG_FILE: renovate-log.ndjson
  RENOVATE_AUTODISCOVER_FILTER: '${CI_PROJECT_ROOT_NAMESPACE}/**'
  RENOVATE_LOG_FILE_LEVEL: trace
  LOG_LEVEL: debug
  RENOVATE_TOKEN: $BOT_TOKEN

workflow:
  rules:
    # exclude merge requests
    - if: $CI_MERGE_REQUEST_ID
      when: never
    # run the pipeline in other cases
    - when: always

.renovate-base:
  image: $RENOVATE_IMAGE
  # Cache downloaded dependencies and plugins between builds.
  # To keep cache across branches add 'key: "$CI_JOB_NAME"'
  # TODO (if necessary): define cache policy here
  cache:
    key: ${CI_COMMIT_REF_SLUG}-renovate
    paths:
      - renovate/cache/renovate/**


# validator job
renovate-validator:
  extends: .renovate-base
  stage: build
  # force no dependency
  dependencies: []
  script:
    - renovate-config-validator

# dependency check job: on manual or schedule (dry-run otherwise)
renovate-depcheck:
  extends: .renovate-base
  stage: test
  # force no dependency
  dependencies: []
  variables:
    # dry-run by default
    RENOVATE_DRY_RUN: "true"
  script:
    - renovate $RENOVATE_EXTRA_FLAGS
  artifacts:
    when: always
    expire_in: 1d
    paths:
      - "$RENOVATE_LOG_FILE"
  rules:
    # not dry run on manual or schedule
    - if: '$CI_PIPELINE_SOURCE == "schedule" || $CI_PIPELINE_SOURCE == "web"'
      variables:
        RENOVATE_DRY_RUN: "false"
    - if: $RENOVATE_TOKEN

include:
  - remote: "https://gitlab.com/Orange-OpenSource/lfn/ci_cd/\
             gitlab-ci-templates/-/raw/master/markdown.gitlab-ci.yml"
  - remote: "https://gitlab.com/Orange-OpenSource/lfn/ci_cd/\
             gitlab-ci-templates/-/raw/master/markdown-link-check.gitlab-ci.yml"
  - remote: "https://gitlab.com/Orange-OpenSource/lfn/ci_cd/\
             gitlab-ci-templates/-/raw/master/yaml.gitlab-ci.yml"
  - remote: "https://gitlab.com/Orange-OpenSource/lfn/ci_cd/\
             gitlab-ci-templates/-/raw/master/json.gitlab-ci.yml"
  - remote: "https://gitlab.com/Orange-OpenSource/lfn/ci_cd/gitlab-ci-templates\
             /-/raw/master/git.signed.gitlab-ci.yml"
  - project: 'to-be-continuous/gitleaks'
    ref: '1.3.0'
    file: '/templates/gitlab-ci-gitleaks.yml'
  - project: 'to-be-continuous/semantic-release'
    ref: '2.3.0'
    file: '/templates/gitlab-ci-semrel.yml'

###
# Overrides
##

.lint-rules:
  rules:
    # do not allow merge request
    - if: '$CI_MERGE_REQUEST_ID'
      when: never
    # do not launch on schedules
    - if: '$CI_PIPELINE_SOURCE != "schedule"'

markdown:
  extends: .lint-rules
  stage: build
  needs: []

markdown-link-check:
  extends: .lint-rules
  stage: build
  needs: []

yaml:
  extends: .lint-rules
  stage: build
  needs: []

json:
  extends: .lint-rules
  stage: build
  needs: []

git.signed:
  extends: .lint-rules
  stage: build
  needs: []
```