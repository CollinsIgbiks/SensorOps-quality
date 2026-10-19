# SensorOps: Quality Pipeline

> ðŸš§ **In progress.** Part 3 of [SensorOps](docs/DESIGN.md), an end-to-end IoT monitoring platform.

An automated test pipeline for the [SensorOps Telemetry API](https://github.com/CollinsIgbiks/sensorops-api): API tests with REST-Assured, UI tests with Playwright (Page Object Model), load testing with k6 and security scanning with OWASP ZAP, all running in GitHub Actions with Allure reports published to GitHub Pages.

## The problem it solves
Every change to the API should be checked automatically before it can break anything. A fast smoke suite runs on every push; full regression, performance and security runs happen nightly.

## Architecture
See [the architecture diagram in the design doc](docs/DESIGN.md#2-architecture-and-data-flow).

## Tech
Java 25 Â· JUnit 5 Â· REST-Assured Â· Playwright for Java Â· k6 Â· OWASP ZAP Â· Allure Â· GitHub Actions Â· Jira

## How to run it
_Coming in Milestone 7._

## Tests and CI
_Badges and the Allure report link will be added in Milestones 7â€“8._

## Results
_Real numbers will be added as each milestone is completed._

## What I'd do next
_To be written._

## Licence
[MIT](LICENSE)
