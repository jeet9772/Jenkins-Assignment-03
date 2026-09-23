# CI/CD Assignment – Multi-Language Jenkins Pipelines

**Submitted by:** Jeetandra

Set up CI checks for three repositories (Python, Go, Java), each with its own set of Jenkins Freestyle jobs — running security/dependency scans, linting, unit tests with coverage, publishing reports inside Jenkins, archiving artifacts, and sending Slack + Email alerts on every failure and recovery.

## Environment

* Jenkins `2.568.3` running on an AWS EC2 instance (`ubuntu@ip-172-31-47-183`), reachable at `18.207.171.62:8080`
* `JAVA_HOME` set to `/usr/lib/jvm/java-8-openjdk-amd64` for the Java jobs

## Repositories

| Language | Repository                                     | Branch    |
| -------- | ----------------------------------------------- | --------- |
| Python   | `OT-MICROSERVICES/attendance-api`               | `main`    |
| Go       | `OT-MICROSERVICES/employee-api`                 | `main`    |
| Java     | `opstree/spring3hibernate`                      | `master`  |

Each repo got 4 Freestyle jobs, all following the same naming pattern:

* `<name>-credential-scan`
* `<name>-dependency-scan`
* `<name>-lint`
* `<name>-unit-test-coverage`

12 jobs in total, all visible on the Jenkins dashboard:

![Jenkins dashboard showing all 12 jobs](screenshots/jenkins-dashboard-all-jobs.png)

At the time of this screenshot, coverage stood at **70.43%** for `py-attendance-unit-test-coverage`, **46.89%** for `go-employee-unit-test-coverage`, and **6.90%** for `java-spring3-unit-test-coverage` — a good reminder of how far apart three "started around the same time" pipelines can end up on test coverage.

## Checks run per job type

| Job              | Tooling                                         | Key artifact(s)                          |
| ---------------- | ------------------------------------------------ | ----------------------------------------- |
| `credential-scan`  | gitleaks                                          | `gitleaks-report.json`                    |
| `dependency-scan`  | Trivy (Python/Java), govulncheck (Go)             | `trivy-report.json` / `govulncheck-report.txt` |
| `lint`             | go vet + gofmt (Go), Maven Checkstyle (Java)      | `checkstyle-result.xml`, `gofmt-report.txt` |
| `unit-test-coverage` | Maven Surefire + JaCoCo (Java), language-native test runners (Go/Python) | JUnit XML, `jacoco.xml`, coverage summary |

Post-build actions on every job: **Archive the artifacts**, **Publish HTML reports** (where applicable), **Publish JUnit test result report**, and **Record code coverage results** for the coverage jobs.

## Credentials

Two credentials cover both notification channels, stored once in Jenkins and reused across all 12 jobs:

![Jenkins global credentials store](screenshots/jenkins-credentials-store.png)

* `gmail-smtp` — a Gmail App Password so Jenkins can send mail through Gmail's SMTP server
* `slack-bot-token` — the Slack bot token used by the Jenkins Slack Notification plugin

## Slack Integration

Created a Slack app (`jenkins-ci`) in a dedicated workspace, with just the two bot scopes it actually needs to post messages:

![Slack Bot Token Scopes — chat:write and chat:write.public](screenshots/slack-bot-token-scopes.png)

Installed it to the workspace:

![Installing the jenkins-ci Slack app](screenshots/slack-app-install-allow.png)

Added the bot to a dedicated `#jenkins-alert` channel:

![jenkins-ci app added to #jenkins-alert](screenshots/slack-jenkins-alert-channel-setup.png)

Configured the Slack plugin in Jenkins' global settings (workspace, credential, default channel) and confirmed the connection:

![Jenkins global Slack config — Test Connection: Success](screenshots/jenkins-slack-test-connection.png)

Per job, Slack notifications are set to fire on every failure and once the build is back to normal:

![Slack notification options on a job's post-build config](screenshots/py-attendance-slack-notifications-config.png)

## Email Integration

Added the `gmail-smtp` credential and configured Extended E-mail Notification against `smtp.gmail.com:465` (SSL), sending failure/recovery alerts to a dedicated inbox. Example of a recovery email landing correctly:

![Email notification: py-attendance-dependency-scan Build #4 Fixed](screenshots/email-py-attendance-fixed.png)

## Verifying the alerting actually works

Rather than trust the config on faith, I let real builds fail and watched both channels report it consistently.

### Python — `py-attendance-dependency-scan`

Builds #1–#3 failed, then #4 passed once the underlying issue was fixed. The job's build history shows the failing run:

![py-attendance-dependency-scan build history](screenshots/py-attendance-dependency-scan-build-history.png)

Slack picked up the same failure in real time:

![Slack: py-attendance-dependency-scan #3 Still Failing](screenshots/slack-py-attendance-still-failing.png)

...and the recovery on the next build:

![Slack: py-attendance-dependency-scan #4 Success](screenshots/slack-py-attendance-success.png)

...which matches the "Fixed" email above.

### Go — `go-employee-dependency-scan`

Same pattern: early builds failed, then recovered.

![go-employee-dependency-scan build history](screenshots/go-employee-dependency-scan-build-history.png)

Slack again confirms the recovery:

![Slack: go-employee-dependency-scan #3 Success](screenshots/slack-go-employee-success.png)

### Java — `java-spring3-credential-scan`

The credential scan job cycled through a few failures before settling, and the email plugin correctly reported the final recovery:

![Email: java-spring3-credential-scan Build #5 Fixed](screenshots/email-java-spring3-credential-scan-fixed.png)

`java-spring3-dependency-scan`, by contrast, was still red at the time the dashboard screenshot above was taken — a good example of the dashboard reflecting real, current state rather than a cherry-picked "everything passes" run.

## Summary

| Job                        | Language | Checks run                                    | Notifications                        |
| -------------------------- | -------- | ----------------------------------------------- | -------------------------------------- |
| `py-attendance-*`          | Python   | gitleaks, Trivy, unit tests + coverage, lint     | Slack `#jenkins-alert` + email, on failure & recovery |
| `go-employee-*`            | Go       | gitleaks, govulncheck, go vet/gofmt, unit tests + coverage | Slack `#jenkins-alert` + email, on failure & recovery |
| `java-spring3-*`           | Java     | gitleaks, Trivy, Maven Checkstyle, Surefire + JaCoCo | Slack `#jenkins-alert` + email, on failure & recovery |

All three pipelines are wired to the same Slack channel and email inbox, verified by intentionally letting builds fail and confirming both channels reported the failure and the eventual recovery consistently with each other.
