# A CI/CD pipeline without stored secrets

This pipeline gets every secret it needs from Idira Secrets Manager at run
time. GitHub stores none of them.

Each run proves who it is with a short-lived token that GitHub issues to the
job. Idira checks the token and gives the run only the secrets of its own
environment. Idira records every fetch.

Work in progress. The full description comes when the demo is complete.

Idira Secrets Manager was called Conjur before. The old name stays in the
`cyberark/conjur-action` action and in the `CONJUR_*` variables.

Every secret value in this demo is fake. The pipeline log prints the values
on purpose, only to prove that they arrived.
