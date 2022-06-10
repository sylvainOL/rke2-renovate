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
