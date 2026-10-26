# Tough Day 2 — Adobe's AEM load/stress-testing CLI

[Docs (6.5)](https://experienceleague.adobe.com/en/docs/experience-manager-65/content/implementing/developing/testing/tough-day) · jar from [Maven Central `com.adobe.qe.toughday2`](https://repo1.maven.org/maven2/com/adobe/qe/toughday2/). Tough Day 2 is the tool behind the "Load Testing" row in [pre-launch-testing-types.md](pre-launch-testing-types.md): it hammers an AEM instance with a weighted mix of authoring-style actions to find its limits. **Requires Java 8** (the tool's requirement, independent of what the instance runs on).

```
java -jar toughday2.jar --host=localhost
```

That default run uses the bundled **`toughday` suite**: create pages + live copies (with rollouts), GET homepage, QueryBuilder queries, create asset hierarchies, delete assets — **15% writes / 85% reads**. Defaults worth knowing: `--port=4502`, `--user=admin --password=admin`, `--duration=1d` (yes, a day — set it), `--timeout=180` (per-test seconds), `--installsamplecontent=true` (**the default run installs Tough Day sample content onto the target** — templates under `/conf/toughday-templates/…`; flip to false on anything you care about).

## The two warnings that decide where you run it

1. **There is no clean-up step.** Everything the suite creates (pages, live copies, asset trees) stays. Adobe's instruction: run against a **cloned staging instance and drop it afterwards** — never production.
2. `count` (max executions per test) is approximate under concurrency — parallel threads overshoot it proportionally to thread count. Don't build assertions on exact counts.

Cloud tie-ins: an AEMaaCS **sandbox cannot host a load test** (no auto-scaling — [cloud-manager-program-types.md](../cloud-service/cloud-manager-program-types.md)); on AMS the Cloud Manager pipeline's performance gate is its own stage-side test driven by KPIs ([cloud-manager-ams-program-setup-kpis.md](../cloud-service/cloud-manager-ams-program-setup-kpis.md)) — Tough Day is the tool you run yourself, outside the pipeline.

## Configuration model

Three layers, lowest to highest precedence: **built-in defaults < YAML file < command-line args**. `--dryrun=true` parses and merges everything, prints the effective config, and exits — use it before any long run.

```yaml
globals:
  host: localhost
  port: 4502
  user: admin
  password: admin
  duration: 30m
  suite: toughday
  installsamplecontent: false
tests:
  - add: CreatePageTreeTest        # add a test (class name)
    properties: { name: MyPages, weight: 5, count: 100 }
  - config: UploadAsset            # reconfigure a test already in the suite
    properties: { title: NewAsset }
  - exclude: DeleteAsset           # drop a suite test
publishers:
  - add: CSVPublisher
  - add: ConsolePublisher
```

Same grammar on the CLI: `--add <Test> prop=val`, `--config <Test> prop=val`, `--exclude <Test>`. Running the same test class twice requires distinct `name=` values. Per-test knobs: `weight` (probability multiplier in the random pick), `count` (max executions), `timeout` (overrides global), plus class-specific properties (`--help <TestClass>` lists them, e.g. `template=` on `CreatePageTreeTest`).

## Run modes and output

- **`--runmode=normal concurrency=<threads> waittime=<ms>`** — fixed thread pool; each thread waits `waittime` ms between executions. Models "N concurrent users".
- **`--runmode=constantload load=<n>`** — starts n test executions per interval regardless of how long they take. Models a fixed arrival rate; the honest mode for "can it sustain X req/s" questions (normal mode's rate degrades as the instance slows, hiding the backlog — closed vs open workload model).

Metrics per test (12): timestamp, passed, failed, and — successful runs only — min/max/median/average/std-dev/90p/99p/99.9p durations + real throughput. Publishing: `--publishmode type=simple` (cumulative since start, the default) vs `type=intervals interval=10s` (windowed — what you want for spotting degradation over time). Publishers `CSVPublisher` + `ConsolePublisher` are both on by default.

Logs land in a `logs/` dir next to the jar: `toughday.log` (global) + `toughday_<testname>.log` — they **append across runs**, so separate runs by directory if you'll compare.

Discovery: `--help_full` (everything), `--help_tests` (all test classes), `--help_publish`, `--help --suite=<name>`, `--help --tag=<tag>`, `--help <TestClass>`.

Source: Adobe 6.5 Tough Day page. Docs-based, not run locally.
