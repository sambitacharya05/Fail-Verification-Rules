# Automated failure triage: initial research findings

Scope: the analysis logic and rules for deciding why a failed automated test failed, for a Python + Robot Framework + SeleniumLibrary + AppiumLibrary stack (about 17,000 tests a day, 100+ failures a day). Target verdict: category (environment, product bug, flaky, maintenance period, script issue, data issue), confidence, evidence, next action (rerun, file bug, quarantine, ignore). System architecture is out of scope.

Research date: 2026-11-11. Sources are primary wherever one could be reached: official docs, library source code on GitHub, the W3C spec, and the papers themselves (arXiv, publisher, or author copies).

**Labelling convention used throughout**

- **[V]** Verified fact, cited inline to the source that owns it.
- **[I]** Inference: my reasoning from the cited facts. Not stated by the source.
- **[P]** Proposal: a draft rule or design suggestion for you to accept, change or reject.
- **[U]** Unverified: I could not confirm it from a primary source, and I say so.

---

## 1. Summary (key takeaways)

1. **The strongest single signal is history plus rerun, but it is not proof.** Google defines a flaky result as "both a passing and a failing result with the same code" and reports that about 84% of observed pass-to-fail transitions involved a flaky test [V] ([Micco 2016](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html)). An immediate same-environment rerun under-detects flakiness: Maven's same-JVM rerun flagged only 23% of failures that isolated reruns confirmed as flaky [V] ([DeFlaker, ICSE 2018](https://www.cs.cornell.edu/~legunsen/pubs/BellETAL18DeFlaker.pdf)).
2. **Rerunning hides real bugs if "flaky" means "ignore".** At Chromium, flaky-test predictors reached 99.2% precision but would have caused about 76% of regression faults to be missed, and flaky tests revealed more than a third of all regression faults [V] ([Haben et al. 2023](https://arxiv.org/abs/2302.10594)). So classify the *failure*, not the *test*, and never auto-ignore on a "known flaky test" label alone.
3. **UI tests flake mostly from waiting.** Async Wait caused 45.1% of 235 flaky UI tests (52.0% web, 32.5% mobile). Platform issues are more common on mobile (21.7% vs 10.5%) [V] ([Romano et al., ICSE 2021](https://arxiv.org/abs/2103.02669)). Google found that WebDriver and Android-emulator tests are far more often flaky than average (10.45% to 25.46% of such tests, vs 1.65% overall), largely because they are large tests [V] ([Listfield 2017](https://testing.googleblog.com/2017/04/where-do-our-flaky-tests-come-from.html)).
4. **Most error types are ambiguous on their own.** `NoSuchElement`, "did not appear in 5 seconds", and timeouts can each mean a locator problem, a real UI regression, or a slow environment. Only a few signatures are close to deterministic: session-not-created, browser or driver crash, invalid selector, unknown keyword, unknown variable, and certificate errors. Section 3 catalogues them with ambiguity notes [V for messages, I for category mapping].
5. **Existing tools converge on about five buckets plus "To Investigate".** ReportPortal uses Product Bug, Automation Bug, System Issue, No Defect, To Investigate [V] ([RP docs](https://reportportal.io/docs/work-with-reports/InvestigationOfFailure/)). Mozilla Treeherder uses fixed by commit, intermittent, infra, expected fail, plus unclassified states [V] ([Treeherder fixture](https://github.com/mozilla/treeherder/blob/master/treeherder/model/fixtures/failure_classification.json)). **No tool has a "maintenance period" category, and none has "data issue" as a top-level category.** You will have to model both yourself (ReportPortal custom sub-types are one route).
6. **Industrial classifiers reach roughly 60–90% accuracy, rarely more without history.** Examples: Huawei CAM 58.3%/65.8% offline, about 72% after deployment, which was about 3x better than the regex rules it replaced [V] ([Jiang et al., ICSE 2017](https://arxiv.org/abs/1703.00768)). NCChecker reached macro F1 71.9% over bug/environment/script/third-party [V] ([Gao et al., FSE 2024](https://arxiv.org/abs/2405.02922)). Mozilla's classifier reached 73% average precision, with some suites at 100% [V] ([Lampel et al., FSE 2021](https://www.se.cs.uni-saarland.de/publications/docs/LJA+21.pdf)). Plan for a sizeable human-review share, especially at first.
7. **Most predictive features in the literature:** similarity to historically labelled failures (log or stack text), test history (flake rate, failure rate), run duration, telemetry such as CPU load, OS and platform, and coverage of changed code. Test smells and test-code tokens alone were weak [V] (Fair: run duration ranked first; FlakeFlagger: execution time, coverage, changed-line coverage, third-party library use; Lampel: runtime, CPU load, OS version). See section 6.
8. **Human triage order from first-party sheriff docs:** first check whether many tests or bots fail together (infra or a common culprit). Then check the change list ("blamelist") for an obvious culprit. Then check the flakiness history. Focus on the *first* failure in a suite, because later failures can be collateral [V] ([Chromium sheriffing doc](https://chromium.googlesource.com/chromium/src/+/refs/tags/78.0.3890.0/docs/sheriff.md)). Section 7 turns this into a decision path [P].
9. **Jira/Confluence matching must be two-stage.** Jira text search no longer applies fuzzy, proximity or boost operators, and reserved characters need escaping [V] ([Atlassian text search](https://support.atlassian.com/jira-software-cloud/docs/search-for-issues-using-the-text-field/)). Use JQL or CQL only to *retrieve candidates*, then score similarity locally on normalised failure fingerprints [P]. Mozilla matches intermittents at the test-path level, not the error-message level [V] ([Mozilla sheriffing](https://firefox-source-docs.mozilla.org/testing/sheriffed-intermittents/index.html)).
10. **Confidence: use explicit thresholds and keep a default "To Investigate" bucket.** ReportPortal auto-assigns only when the model probability is at least 50%, and shows suggestions only above 40% [V] ([RP auto-analysis](https://reportportal.io/docs/analysis/AutoAnalysisOfLaunches/), [ML suggestions](https://reportportal.io/docs/analysis/MLSuggestions/)). LLM verbalised confidence tends to be overconfident [V] ([Xiong et al., ICLR 2024](https://arxiv.org/abs/2306.13063)). Gate each automatic action separately, and calibrate against your own labelled history [P].

---

## 2. Signals available in the stack

### 2.1 Robot Framework result model (output.xml / listener)

**Statuses** [V] ([RF User Guide, "Test and suite statuses"](https://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html#test-and-suite-statuses); [result.xsd](https://github.com/robotframework/robotframework/blob/master/doc/schema/result.xsd))
- Status values in output.xml: `PASS`, `FAIL`, `SKIP`, `NOT RUN` (enumeration `StatusValue` in result.xsd, RF 7.0+ schema). Per the listener docs, SKIP and NOT RUN are new in RF 4.0, and keywords that were not executed get `NOT RUN`.
- FAIL causes listed by the guide: a keyword fails (raises an exception or is called incorrectly), syntax errors, or an empty test. "If a suite setup fails, tests in that suite are marked failed without running them. If a suite teardown fails, tests are marked failed retroactively."
- **Skip-on-failure:** the `--skiponfailure <tag>` option, or the reserved tag `robot:skip-on-failure` (RF 5.0+), marks failed tests as SKIP. The guide says it is meant for tests that are "not yet ready" or test functionality that is not yet ready. Also `--skip`, `robot:skip`, and the BuiltIn `Skip` / `Skip If` keywords.
- **Continuable failures:** `robot:continue-on-failure` and `robot:recursive-continue-on-failure` let a test keep running after a failure. All failures are then listed in a message that starts `Several failures occurred:` [V] (User Guide, "Continue on failure").
- **Stopping:** `--exitonfailure` (-X) and the `robot:exit-on-failure` tag (RF 7.2+) stop the whole run [V].

**Status messages that RF itself generates** [V] ([robot/running/status.py](https://github.com/robotframework/robotframework/blob/master/src/robot/running/status.py), [timeouts/timeout.py](https://github.com/robotframework/robotframework/blob/master/src/robot/running/timeouts/timeout.py), [BuiltIn.py](https://github.com/robotframework/robotframework/blob/master/src/robot/libraries/BuiltIn.py), [namespace.py](https://github.com/robotframework/robotframework/blob/master/src/robot/running/namespace.py), [variables/notfound.py](https://github.com/robotframework/robotframework/blob/master/src/robot/variables/notfound.py))

| Message prefix / template | Meaning |
|---|---|
| `Setup failed:\n…` / `Teardown failed:\n…` / `…\n\nAlso teardown failed:\n…` | Test-level setup or teardown failed |
| `Suite setup failed:\n…` / `Parent suite setup failed:\n…` / `Parent suite teardown failed:\n…` | Failure inherited from a suite fixture. The test body may never have run |
| `Test execution stopped due to a fatal error.` / `Failure occurred and exit-on-failure mode is in use.` / `Error occurred and exit-on-error mode is in use.` | Collateral: test was not run because the run was stopped |
| `{Kind} timeout {value} exceeded.` (for example `Test timeout 5 minutes exceeded.`) | RF test or keyword timeout |
| `Keyword '{name}' failed after retrying {…}. The last error was: {err}` | `Wait Until Keyword Succeeds` exhausted its retries |
| `No keyword with name '{name}' found.` / `Multiple keywords with name '{name}' found` | Test-code (script) error |
| `Variable '{name}' not found.` | Test-code or variable-file error |
| `Several failures occurred:\n\n1) …` | Continuable failures; more than one cause is possible |

**Rerun and merge** [V] (User Guide, "Re-executing failed test cases" and "Merging results"; [result/merger.py](https://github.com/robotframework/robotframework/blob/master/src/robot/result/merger.py))
- `robot --rerunfailed output.xml` (-R) reruns failed tests. `--rerunfailedsuites` (-S) reruns *all* tests of failed suites, "useful when the tests in a test suite depends on each other".
- `rebot --merge original.xml rerun.xml`: "tests from the latter test runs replace tests in the original", except that skipped tests in later runs are ignored (RF 4.1+).
- The merged test message is generated as HTML. It starts `*HTML* … Test has been re-executed and results merged.` and then contains `New status:` / `New message:` and `Old status:` / `Old message:` sections (source: `_create_merge_message` in merger.py). **A rerun-pass therefore leaves a parseable trace in output.xml**: final status PASS, plus an old status FAIL with the original message.

**Timestamps and keyword-level detail** [V] (result.xsd)
- Every `<suite>`, `<test>`, `<kw>` and control structure has a `<status status="…" start="xs:dateTime" elapsed="xs:float">` element (RF 7.0+ schema; older versions use a different schema, see the [schema folder](https://github.com/robotframework/robotframework/tree/master/doc/schema)).
- `<msg time="…" level="TRACE|DEBUG|INFO|WARN|ERROR|FAIL|SKIP" html="…">` log messages are nested inside the keyword that logged them. `<kw>` has `name`, `owner` (library or resource name) and `type` (`SETUP` or `TEARDOWN` when applicable). `<test>` has `<tag>` and `<timeout value>`. A top-level `<errors>` block holds execution errors, such as failed library imports, which "do not fail tests" by default [V] (User Guide, "Stopping on parsing or execution error").
- [I] This lets you find the *innermost failing keyword* (the deepest `<kw>` with `status="FAIL"`), its owning library (SeleniumLibrary, AppiumLibrary, BuiltIn, or a user resource file), the time of failure, and the elapsed time. Elapsed time is a useful feature (see Fair in section 6), for example failing after exactly the wait timeout vs failing instantly.

**Listener API** [V] (User Guide, "Listener interface")
- Versions 2 and 3. Version 3 is the default since RF 7.0. It receives the real running and result model objects (`start_test(data, result)`, `end_test(data, result)`, keyword-level methods from RF 7.0, `log_message(message)`).
- Listener v2 `end_test` attributes include `starttime`, `endtime`, `elapsedtime` (ms), `status` (PASS/FAIL/SKIP), `message`, `tags`, `source`, `lineno`. `log_message` gives `message`, `level`, `timestamp`, `html`.
- [I] A listener can capture extra evidence at failure time (current URL, page source, app activity, device logs) before teardown closes the browser. This is useful because screenshot and page source are "not always" available today.

### 2.2 SeleniumLibrary and AppiumLibrary behaviour

**SeleniumLibrary (v6.9.0 source)** [V] ([__init__.py](https://github.com/robotframework/SeleniumLibrary/blob/master/src/SeleniumLibrary/__init__.py), [waiting.py](https://github.com/robotframework/SeleniumLibrary/blob/master/src/SeleniumLibrary/keywords/waiting.py), [elementfinder.py](https://github.com/robotframework/SeleniumLibrary/blob/master/src/SeleniumLibrary/locators/elementfinder.py), [librarycomponent.py](https://github.com/robotframework/SeleniumLibrary/blob/master/src/SeleniumLibrary/base/librarycomponent.py), [screenshot.py](https://github.com/robotframework/SeleniumLibrary/blob/master/src/SeleniumLibrary/keywords/screenshot.py))
- Defaults: `timeout=5 s` for `Wait …` keywords, `implicit_wait=0 s`, `page_load_timeout=5 min`, `run_on_failure="Capture Page Screenshot"`.
- **Run-on-failure only fires when a SeleniumLibrary keyword fails.** `run_keyword` wraps only the library's own keywords and calls `failure_occurred()` on exception. A failure in `BuiltIn.Should Be Equal`, or in a user keyword's own logic, does not trigger a screenshot. If the screenshot itself fails, the library logs a WARN: `Keyword '…' could not be run on failure: …`. Screenshot files default to `selenium-screenshot-{index}.png`.
  - [I] This explains part of "screenshot not always available". A WARN that says the screenshot could not be taken is itself evidence of a dead browser session (environment).
- Message templates (`<TIMEOUT>` is replaced by a time string such as "5 seconds"):
  - `{Element} with locator '{locator}' not found.` (element lookup)
  - `Element '{locator}' did not appear in <TIMEOUT>.` / `did not disappear in` / `not visible after` / `still visible after` / `was not enabled in` / `did not get text '{text}' in`
  - `Text '{text}' did not appear in <TIMEOUT>.` / `Location did not become '{expected}' in <TIMEOUT>.`
  - `Page should have contained {tag} '{locator}' but did not.` / `Page should have contained text '{text}' but did not.`
  - `Element '{locator}' should have contained text '…' but …` / `Title should have been '…' but was '…'.` / `Location should have been '…' but was '…'.` / `Alert not found in …`
  - `No browser is open.`
- Wait loop detail: the wait keywords catch `ElementNotFound` and `StaleElementReferenceException` while polling (logging "Suppressing StaleElementReferenceException from Selenium."). The final message is the last not-found error or the timeout message. [I] So a raw `StaleElementReferenceException` in a final message usually comes from a non-wait keyword (click, input) acting on an element that re-rendered.

**AppiumLibrary (v3.2.1 source)** [V] ([__init__.py](https://github.com/serhatbolsu/robotframework-appiumlibrary/blob/master/AppiumLibrary/__init__.py), [_waiting.py](https://github.com/serhatbolsu/robotframework-appiumlibrary/blob/master/AppiumLibrary/keywords/_waiting.py), [_element.py](https://github.com/serhatbolsu/robotframework-appiumlibrary/blob/master/AppiumLibrary/keywords/_element.py), [keywordgroup.py](https://github.com/serhatbolsu/robotframework-appiumlibrary/blob/master/AppiumLibrary/keywords/keywordgroup.py), [_runonfailure.py](https://github.com/serhatbolsu/robotframework-appiumlibrary/blob/master/AppiumLibrary/keywords/_runonfailure.py))
- Defaults: `timeout=5`, `run_on_failure='Capture Page Screenshot'`, `sleep_between_wait_loop=0.2`. A metaclass decorator wraps every public keyword and runs the on-failure keyword on any exception. A failure of that keyword is reported as `Keyword '…' could not be run on failure: …`.
- Messages: `Element locator '{locator}' did not match any elements.` / `… did not match any elements after {timeout}` / `Element '{locator}' did not appear in <TIMEOUT>` / `Element '{locator}' was not visible in {timeout}` / `Text '{text}' did not appear in <TIMEOUT>` / `Page should have contained element '{locator}' but did not` / `Element '{locator}' should be enabled/disabled/visible …` / `No application is open` / `Element locator with prefix '{prefix}' is not supported`.

### 2.3 WebDriver, Selenium, ChromeDriver and Appium errors

- **W3C WebDriver error codes** [V] ([W3C WebDriver, section 6.6 Errors](https://www.w3.org/TR/webdriver2/#errors)), with HTTP status: element click intercepted (400), element not interactable (400), insecure certificate (400), invalid argument (400), invalid cookie domain (400), invalid element state (400), invalid selector (400), invalid session id (404), javascript error (500), move target out of bounds (500), no such alert (404), no such cookie (404), no such element (404), no such frame (404), no such window (404), no such shadow root (404), script timeout (500), session not created (500), stale element reference (404), detached shadow root (404), timeout (500), unable to set cookie (500), unable to capture screen (500), unexpected alert open (500), unknown command (404), unknown error (500), unknown method (405), unsupported operation (500).
- **Selenium's own "likely cause" guidance** for each exception [V] ([selenium.dev, Understanding Common Errors](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/)) is summarised in the table below. For example, for NoSuchElement: wrong place, element not yet in the DOM, or locator changed. For SessionNotCreated: browser/driver version mismatch, OS permissions, or missing driver. For WebDriverException: browser crashed or closed mid-command, driver couldn't connect, or no more specific class.
- **ChromeDriver adds non-W3C status strings** [V] ([chromedriver status.cc](https://chromium.googlesource.com/chromium/src/+/main/chrome/test/chromedriver/chrome/status.cc)): `chrome not reachable`, `disconnected`, `tab crashed`, `target frame detached`, `no such execution context`, `aborted by navigation`.
- **Appium base driver** [V] ([errors.ts](https://github.com/appium/appium/blob/master/packages/base-driver/lib/protocol/errors.ts), [driver.ts](https://github.com/appium/appium/blob/master/packages/base-driver/lib/basedriver/driver.ts), [proxy.ts](https://github.com/appium/appium/blob/master/packages/base-driver/lib/jsonwp-proxy/proxy.ts)): `A new session could not be created. Details: …`; `An unknown server-side error occurred while processing the command.`; `A session is either terminated or not started`; `New Command Timeout of {n} seconds expired. Try customizing the timeout using the 'newCommandTimeout' desired capability` (default 60 s); `Could not proxy command to the remote server. Original error: …`.
- **UiAutomator2 driver** [V] ([core.ts](https://github.com/appium/appium-uiautomator2-driver/blob/master/lib/uiautomator2-server/core.ts)): `'{method} {url}' cannot be proxied to UiAutomator2 server because the instrumentation process is not running (probably crashed). Check the server log and/or the logcat output for more details`.

---

## 3. Signal catalogue

Columns: signal → likely category or categories (in rough order of likelihood) → why → ambiguity and disambiguation → source. The message text is **[V]**. The category mapping is **[I]**, informed by the cited "likely cause" descriptions and research. Categories: ENV = environment, PROD = product bug, FLAKY, MAINT = maintenance period, SCRIPT = test code or locator, DATA = test data.

| # | Signal (message, exception or condition) | Likely category | Why | Ambiguity / how to disambiguate | Source |
|---|---|---|---|---|---|
| 1 | `session not created` / `SessionNotCreatedException` / Appium `A new session could not be created.` | ENV | Browser/driver mismatch, driver missing, grid or device capacity | Rarely PROD. SCRIPT if capabilities are wrong (bad `invalid argument` details). Check whether every test on the same node or device failed the same way | [W3C](https://www.w3.org/TR/webdriver2/#errors), [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/), [Appium errors.ts](https://github.com/appium/appium/blob/master/packages/base-driver/lib/protocol/errors.ts) |
| 2 | `chrome not reachable`, `disconnected`, `WebDriverException` (browser crashed mid-command) | ENV | Browser process died or driver lost connection | `tab crashed` repeatedly on the same page can be a PROD memory or renderer problem. Look at the cross-test pattern | [status.cc](https://chromium.googlesource.com/chromium/src/+/main/chrome/test/chromedriver/chrome/status.cc), [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/) |
| 3 | `invalid session id` / Appium `A session is either terminated or not started` | ENV or SCRIPT | Session deleted: crash, grid timeout, or test called close/quit early | SCRIPT if the test (or a previous test sharing the session) called `Close Browser` / `close()` / `quit()`. Check earlier keywords in output.xml | [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/) |
| 4 | UiAutomator2 `instrumentation process is not running (probably crashed)` / `Could not proxy command to the remote server` | ENV (or PROD if the app under test crashed) | Device-side automation server died | Check logcat or app crash logs. An app crash with a stack in the app's package points to PROD | [UIA2 core.ts](https://github.com/appium/appium-uiautomator2-driver/blob/master/lib/uiautomator2-server/core.ts), [proxy.ts](https://github.com/appium/appium/blob/master/packages/base-driver/lib/jsonwp-proxy/proxy.ts) |
| 5 | Appium `New Command Timeout of N seconds expired` | ENV or SCRIPT | No command reached Appium for N seconds | SCRIPT if the test sleeps or waits longer than newCommandTimeout. ENV if the client or network hung | [driver.ts](https://github.com/appium/appium/blob/master/packages/base-driver/lib/basedriver/driver.ts) |
| 6 | `insecure certificate` | ENV | Certificate expired, self-signed or mismatched on the test environment | Almost never a test issue | [W3C](https://www.w3.org/TR/webdriver2/#errors), [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/) |
| 7 | Connection refused or HTTP connection errors to the grid or Appium URL (Python-level, for example urllib3 errors) | ENV | Infrastructure unreachable | [U] Exact Python exception text not verified for your versions. Collect real samples | [I] |
| 8 | `invalid selector` / `Element locator with prefix '…' is not supported` | SCRIPT | Malformed locator. Deterministic | Very low ambiguity | [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/), [AppiumLibrary elementfinder](https://github.com/serhatbolsu/robotframework-appiumlibrary/blob/master/AppiumLibrary/locators/elementfinder.py) |
| 9 | `No keyword with name '…' found.` / `Variable '…' not found.` / `Multiple keywords with name …` / library import errors in `<errors>` | SCRIPT (variable not found can be DATA if variable files are environment data) | Test-code error. Deterministic | Missing library import can be ENV (package not installed on the runner) | [namespace.py](https://github.com/robotframework/robotframework/blob/master/src/robot/running/namespace.py), [notfound.py](https://github.com/robotframework/robotframework/blob/master/src/robot/variables/notfound.py), [User Guide](https://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html) |
| 10 | `invalid argument` | SCRIPT (or ENV for missing upload file on the remote node) | Wrong value type, non-absolute URL, bad capability | Read the message detail | [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/) |
| 11 | `no such element` / SeleniumLibrary `… with locator '…' not found.` / AppiumLibrary `… did not match any elements` | SCRIPT, PROD, ENV, FLAKY (**highly ambiguous**) | Selenium lists: wrong page or step, element not in DOM yet, locator changed | Check (a) page source or screenshot: is it the expected page? an error page? (b) is the element present under a different attribute? (c) did the app build or the test code change since the last pass? (d) does a rerun pass? | [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/), [elementfinder.py](https://github.com/robotframework/SeleniumLibrary/blob/master/src/SeleniumLibrary/locators/elementfinder.py) |
| 12 | `Element '…' did not appear in 5 seconds.` / `not visible after` / `was not enabled in` / `Text '…' did not appear in` | FLAKY or ENV (slow), SCRIPT (wrong wait), PROD (feature broken) | Default SeleniumLibrary timeout is only 5 s | Compare elapsed time with history for the same step. A cluster across many tests at the same time points to ENV. Rerun-pass points to FLAKY or ENV. Consistent failure after an app change points to PROD or SCRIPT | [waiting.py](https://github.com/robotframework/SeleniumLibrary/blob/master/src/SeleniumLibrary/keywords/waiting.py), [Romano 2021](https://arxiv.org/abs/2103.02669) |
| 13 | `stale element reference` | FLAKY (async re-render) or SCRIPT (holds element across navigation) | DOM refreshed, navigated, or window or frame switched | Rarely PROD. If deterministic after an app change, the script needs re-locating | [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/) |
| 14 | `element click intercepted` | FLAKY (animation or overlay timing), SCRIPT, ENV (cookie or consent banner, environment-specific popup), PROD (unexpected modal or error dialog) | Another element received the click | The screenshot is decisive: what overlays the target? | [W3C](https://www.w3.org/TR/webdriver2/#errors), [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/) |
| 15 | `element not interactable` / `invalid element state` / `move target out of bounds` | SCRIPT or FLAKY; PROD if the control is wrongly disabled | Hidden element, wrong element matched, viewport too small, disabled control | Check visibility in page source. Check window size (ENV config) | [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/) |
| 16 | `timeout` (page load, default 300 s), Selenium `TimeoutException`, RF `Test timeout … exceeded.` | ENV (slow or hung environment), PROD (performance regression or hang), SCRIPT (unrealistic timeout) | An operation did not finish in time | Many tests timing out at once points to ENV. A single test whose duration has trended upward points to PROD (performance). Check server-side health if available | [W3C](https://www.w3.org/TR/webdriver2/#errors), [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/), [timeout.py](https://github.com/robotframework/robotframework/blob/master/src/robot/running/timeouts/timeout.py) |
| 17 | `Keyword '…' failed after retrying … The last error was: …` | FLAKY or ENV, then classify the last error | The test already retried internally (`Wait Until Keyword Succeeds`) | The retry hides intermittency inside one run. Classify the "last error" with the rest of this table | [BuiltIn.py](https://github.com/robotframework/robotframework/blob/master/src/robot/libraries/BuiltIn.py) |
| 18 | `unexpected alert open` | PROD (unexpected dialog) or SCRIPT (unhandled expected dialog) | A modal blocked the command | Read the alert text if logged | [W3C](https://www.w3.org/TR/webdriver2/#errors) |
| 19 | `no such frame` / `no such window` / `no such alert` | SCRIPT or FLAKY | Frame or window not loaded yet, wrong handle, alert not yet open | PROD if the window should exist and the app no longer opens it | [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/) |
| 20 | `javascript error` / `script timeout` | SCRIPT (user JS) or PROD (page JS state) | Error while executing JavaScript supplied by the test | Check whether the test uses `Execute JavaScript` | [W3C](https://www.w3.org/TR/webdriver2/#errors) |
| 21 | Assertion mismatches: `Page should have contained text '…' but did not.`, `… should have contained text '…' but …`, `Title should have been '…' but was '…'.`, BuiltIn `Should Be Equal` messages | PROD, DATA, or SCRIPT (stale expected value) | The application state differs from expectation | DATA if the expected value is data-dependent (balances, names, counts). SCRIPT if requirements changed intentionally. PROD otherwise. Requirements and Jira are needed. Screenshot and page source show the actual state | [SeleniumLibrary element.py](https://github.com/robotframework/SeleniumLibrary/blob/master/src/SeleniumLibrary/keywords/element.py) |
| 22 | `No browser is open.` / `No application is open` | SCRIPT (missing setup or ordering) or ENV (earlier crash) | The keyword ran without an active session | Look for an earlier session crash in the same suite | [SeleniumLibrary __init__.py](https://github.com/robotframework/SeleniumLibrary/blob/master/src/SeleniumLibrary/__init__.py), [_applicationmanagement.py](https://github.com/serhatbolsu/robotframework-appiumlibrary/blob/master/AppiumLibrary/keywords/_applicationmanagement.py) |
| 23 | WARN `Keyword 'Capture Page Screenshot' could not be run on failure: …` | ENV (session already dead) | Screenshot failed because the browser or app is gone | Combine with the primary error | [SeleniumLibrary __init__.py](https://github.com/robotframework/SeleniumLibrary/blob/master/src/SeleniumLibrary/__init__.py) |
| 24 | `Suite setup failed:` / `Parent suite setup failed:` | Same as the setup's own error. Often ENV or DATA (login, seed data) | All tests in the suite inherit one failure | Classify once at the suite and propagate. Avoid N duplicate verdicts | [status.py](https://github.com/robotframework/robotframework/blob/master/src/robot/running/status.py) |
| 25 | `Teardown failed:` with test body PASS | ENV or SCRIPT (cleanup) | Cleanup failed after the checks passed | Usually lower priority. PROD only if the teardown verifies product state | [status.py](https://github.com/robotframework/robotframework/blob/master/src/robot/running/status.py) |
| 26 | `Test execution stopped due to a fatal error.` / exit-on-failure messages | Collateral (not an independent cause) | The test did not run because the run stopped | Attribute to the first fatal failure | [status.py](https://github.com/robotframework/robotframework/blob/master/src/robot/running/status.py) |
| 27 | Merged message `Test has been re-executed and results merged.` with new PASS and old FAIL | FLAKY or transient ENV | Same test, same code, different outcome | Google's flaky definition (same code, pass and fail). DeFlaker shows reruns in a dirty environment under-detect flakiness, so a rerun-*fail* does not prove a deterministic failure | [merger.py](https://github.com/robotframework/robotframework/blob/master/src/robot/result/merger.py), [Micco 2016](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html), [DeFlaker](https://www.cs.cornell.edu/~legunsen/pubs/BellETAL18DeFlaker.pdf) |
| 28 | `Several failures occurred:` | Multiple | Continuable failures | Classify each numbered failure separately (see open questions) | [User Guide](https://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html) |
| 29 | Page source or screenshot shows an HTTP 5xx error page, "Service Unavailable", gateway error, or a maintenance banner | ENV, or MAINT if inside a declared window | The back end is unavailable | PROD if a single feature endpoint returns 500 while the rest of the environment is healthy | [I] |
| 30 | App or API message about data state: "not found", "already exists", "duplicate", "insufficient", "expired", "locked", "no records" | DATA (or PROD) | Test preconditions on data are not met | PROD if the data verifiably exists and the app mis-reports it. Check the data-setup keywords' status | [I] |
| 31 | Many unrelated tests fail with the same normalised signature in the same time window | ENV, or one PROD or framework culprit | A shared dependency failed | Chromium: "If a LOT of tests failed, look for common symptoms … either a CL in the blamelist or some framework/service flake culprit" | [Chromium sheriff doc](https://chromium.googlesource.com/chromium/src/+/refs/tags/78.0.3890.0/docs/sheriff.md) |
| 32 | The test (or its resource or locator file) changed since its last pass, and the failure is in the changed keyword | SCRIPT | A recent edit is the most likely culprit | Same logic as "blamelist" culprit finding | [Chromium sheriff doc](https://chromium.googlesource.com/chromium/src/+/refs/tags/78.0.3890.0/docs/sheriff.md), [Memon et al. 2017](https://research.google/pubs/taming-google-scale-continuous-testing/) |

---

## 4. Existing taxonomies and tools, mapped to the six categories

### 4.1 What each tool does

**ReportPortal** [V]
- Defect type groups ([docs](https://reportportal.io/docs/work-with-reports/InvestigationOfFailure/)): **To Investigate** (no investigation yet), **Product Bug**, **Automation Bug** (the automation test is not correct), **System Issue** (system-level issue, such as an app crash), **No Defect** (investigated, not a defect). Up to 70 custom defect types can be placed under these groups.
- Auto-Analysis ([docs](https://reportportal.io/docs/analysis/AutoAnalysisOfLaunches/)):
  - Only logs at level ERROR and above (≥ 40000) of "To investigate" items are sent. Logs of previously analysed items are stored in OpenSearch, one document per log.
  - The Analyzer extracts error message, stack trace, numbers, exceptions, URLs and paths into separate fields. Small logs (1–2 lines, ≤ 100 words) are merged.
  - OpenSearch returns the top 10 similar logs per log. "Message similarity is a compulsory condition", and launch name, exceptions and so on boost the score.
  - Candidates are grouped by defect type, and an **XGBoost model (~30 features: cosine similarities of text fields plus a TF-IDF Random Forest probability)** scores each group. The highest group is assigned only if its probability is **≥ 50%**. Comment and bug-tracker link are copied from the best match, and an "AA" label is set.
  - Base for analysis is configurable: all previous launches, launches with the same name, only the previous launch, only the current launch, and so on. There is an option to analyse the longest retry instead of the latest.
- Models ([HowModelsAreRetrained](https://github.com/reportportal/docs/blob/develop/docs/analysis/HowModelsAreRetrained.md)): global models are trained on six projects. Project models retrain after enough labelled items: error-message model after >100 labelled items and 100 new; auto-analysis model after >300 and 100 new. A custom model is kept only if it beats the global one.
- ML Suggestions ([docs](https://reportportal.io/docs/analysis/MLSuggestions/)): XGBoost, ~40 features. Candidates with probability ≤ 40% are filtered out. Bands: SAME = 100%, HIGH = 70–99.9%, LOW = 40–69.9%. Near-duplicates at ≥ 98% similarity are de-duplicated, and at most 5 are shown.
- Pattern Analysis ([docs](https://reportportal.io/docs/analysis/PatternAnalysis/)): user-defined STRING or REGEX rules label matching items ("PA"). They run before Auto-Analysis and **do not themselves set the defect type**.
- Unique Error Analysis ([docs](https://reportportal.io/docs/analysis/UniqueErrorAnalysis/)): groups failures by identical error, with an option to exclude numbers so that errors differing only by numbers merge.
- Flaky widget ([docs](https://reportportal.io/docs/dashboards-and-widgets/FlakyTestCasesTableTop50/)): a test is listed if its status switched between Passed and Failed at least once over the last N launches (default 30, range 2–100), using only the last retry per launch. "% of switches" = switches / (executions − skipped − 1).

**Allure** [V]
- Categories ([docs](https://allurereport.org/docs/categories/), [categories file](https://allurereport.org/docs/how-it-works-categories-file/)): Allure 2 uses a JSON array of `{name, messageRegex, traceRegex, matchedStatuses, flaky}`, matched top to bottom with the first match winning. Allure 3 uses rules with matchers (`statuses`, `flaky`, `labels`, `message`, `trace`, `transitions`, `environments`), and the `flaky` and `transitions` matchers require history.
- Defaults: "Product errors" = failed, "Test errors" = broken. Statuses ([docs](https://allurereport.org/docs/test-statuses/)): failed means unexpected behaviour in the system under test; broken means failed because of a test defect.
- **Robot Framework caveat** [V]: the allure-robotframework adapter maps every non-pass, non-skip RF status to `FAILED` ([utils.py `get_allure_status`](https://github.com/allure-framework/allure-python/blob/master/allure-robotframework/src/listener/utils.py)). [I] So for RF, Allure's built-in failed/broken split carries no information. Only custom regex categories do.

**Mozilla Treeherder** [V] ([failure_classification.json](https://github.com/mozilla/treeherder/blob/master/treeherder/model/fixtures/failure_classification.json); [sheriffing docs](https://firefox-source-docs.mozilla.org/testing/sheriffed-intermittents/index.html))
- Classifications: not classified, fixed by commit, expected fail, intermittent, infra, new failure not classified, autoclassified intermittent, intermittent needs bugid.
- Sheriffs decide "regression or intermittent", annotate intermittents to a **single tracking bug per test path**, and needinfo the owner at high frequency ("currently 30 failures a week").

**Chromium (LUCI)** [V] ([Haben et al. 2021, section 2](https://arxiv.org/abs/2111.03382); [sheriff doc](https://chromium.googlesource.com/chromium/src/+/refs/tags/78.0.3890.0/docs/sheriff.md))
- A failing test is rerun 2–5 times in the same build. If any rerun passes it is labelled *flaky* (does not break the build). If all fail, it is *unexpected* (a legitimate failure).
- The sheriff doc notes "deterministic" here means "failed twice in a row", "so a deterministic failure can still be caused by a flake". Infra failures show as purple bots.

**Datadog Test Optimization** [V] ([flaky test management](https://docs.datadoghq.com/tests/flaky_test_management/), [flaky management](https://docs.datadoghq.com/tests/flaky_management/))
- Flaky means pass and fail "across multiple test runs for the same commit". Tags: Flaky, New Flaky, Known Flaky.
- States: Active, Quarantined (runs but does not fail CI), Disabled (skipped), Fixed (auto after 30 days without flaking). Policies: quarantine or disable by time, branch, or 7-day failure-rate thresholds. "Attempt to Fix" retries 20 times by default.
- AI-assigned flaky categories (needs at least one failed execution with error message and stack): Concurrency, Randomness, Floating Point, Unordered Collection, Too Restrictive Range, Timeout, Order Dependency, Resource Leak, Asynchronous Wait, IO, Network, Time, Environment Dependency, Unknown.

**Buildkite Test Engine** [V] ([monitors](https://buildkite.com/docs/pipelines/configure/tests/workflows/monitors), [flaky management](https://buildkite.com/docs/test-engine/flaky-test-management))
- Monitors:
  - *Passed on retry*: at least one pass and one fail on the same commit SHA. Recovers after 7 days or 100 executions without such events.
  - *Transition count*: pass/fail changes over a configurable window, scored.
  - *Probabilistic flakiness*: Bayesian score, which the docs attribute to Meta.
  - *Duration threshold*: p50 over a window.
- Quarantine means changing the test state to muted or skipped.

**Launchable** [V] ([unhealthy tests](https://help.launchableinc.com/features/unhealthy-tests/))
- Flakiness score from 0 to 1: "the probability that a test fails but eventually passes if you run it repeatedly" against the same build. It requires retries and is recomputed daily.
- Also lists "Most Failed Tests" and "Never Failing Tests".

**Industrial research taxonomies** [V]
- *Huawei CAM* ([Jiang et al., ICSE 2017](https://arxiv.org/abs/1703.00768)): Obsolete test, Product code defect, Configuration error, Test script defect, Device anomaly, Environment issue, Third-party software problem. Distribution in dataset 1: 58.2% product, 15.5% obsolete, 11.6% script, 9.9% configuration. In dataset 2: 46.7% script, 28.1% product, 13.5% third-party.
- *NCChecker* ([Gao et al., FSE Companion 2024](https://arxiv.org/abs/2405.02922)): bug related (63.7%), environmental (23.2%), test script (11.7%), third-party library (1.4%) over 7,159 labelled failed logs from a telecom company.
- *Westermo* ([Strandberg et al., ISSTA 2020](https://arxiv.org/abs/2005.06826)): verdicts pass, fail, **invalid** ("the execution could not be completed", typically unexpected behaviour followed by an unhandled exception in the test framework). The 2026 Westermo report says practitioners check whether a failure originates in "product software, the test framework, the test environment or their interaction" ([Jansson et al. 2026](https://arxiv.org/abs/2609.21843)).
- *Ericsson microservices* ([Sarika et al. 2023](https://arxiv.org/abs/2306.07653)): developers manually sort failures into cluster, artifactory, microservice, CI/CD tests, or environment issues (as summarised in the search snippet for the paper; [U] I did not read the full category definitions).

### 4.2 Mapping onto your six categories [I]/[P]

| Source category | ENV | PROD | FLAKY | MAINT | SCRIPT | DATA | Unresolved |
|---|---|---|---|---|---|---|---|
| ReportPortal | System Issue | Product Bug | (no group; custom sub-type, usually under System Issue or Automation Bug) | No Defect (or a custom SI sub-type) | Automation Bug | (custom sub-type under AB or SI) | To Investigate |
| Allure (custom categories only, for RF) | user-defined | user-defined | `flaky` flag (history) | user-defined | user-defined | user-defined | uncategorised |
| Treeherder | infra | fixed by commit (if the commit was product code) | intermittent, autoclassified intermittent, intermittent needs bugid | – | fixed by commit (if the commit was test code) | – | not classified, new failure not classified |
| Chromium LUCI | purple / infra | unexpected (consistent) | flaky (passed on retry) | tree state "Maintenance" exists for the CQ, not for tests | unexpected (consistent) | – | – |
| Datadog | Network, IO, Environment Dependency (as flaky sub-causes) | – | all categories (Async Wait, Concurrency, …) | – | Order Dependency, Too Restrictive Range, Randomness (test-design causes) | – | Unknown |
| Huawei CAM | Device anomaly, Environment issue, Configuration error (env config), Third-party software | Product code defect | – | – | Test script defect, Obsolete test, Configuration error (test config) | – | – |
| NCChecker | Environmental, Third-party library | Bug related | – | – | Test script | – | – |
| Romano UI root causes | Environment: Platform issue, Layout difference | – | Async Wait: network resource loading, resource rendering, animation timing | – | Test Runner API: DOM selector issue, incorrect interaction. Test script logic: resource load order, order dependency, randomness, time | – | – |
| Westermo | test environment, (invalid verdict) | product software | intermittent (q-score) | – | test framework | – | – |

Observations:
- [V] No surveyed tool or paper has a **maintenance window** category.
- [I] **Data issues** are folded into "System Issue", "Automation Bug" or "Configuration error". Your six-way split is finer than any public taxonomy. Expect less prior art, and plan to collect labelled examples for MAINT and DATA early.
- [P] If you adopt ReportPortal, a natural encoding is the five groups plus custom sub-types. For example: SI/Environment, SI/Maintenance window, AB/Script-Locator, AB/Test data (or SI/Test data), and a "Flaky" sub-type under whichever group your team treats as owner.

---

## 5. Flaky test findings

### 5.1 Root causes

- **Luo et al., "An Empirical Analysis of Flaky Tests", FSE 2014** [V] ([PDF](https://mir.cs.illinois.edu/lamyaa/publications/fse14.pdf), DOI 10.1145/2635868.2635920)
  - Dataset: 201 commits that likely fix flaky tests in 51 open-source projects, 161 precisely classified.
  - Ten root-cause categories: **Async Wait (74/161 = 45%)**, **Concurrency (20%)**, **Test Order Dependency (12%)**, then Resource Leak, Network, Time, IO, Randomness, Floating Point Operations, Unordered Collections.
  - Other findings: 78% of flaky tests are flaky from when they are first written. 96% are platform-independent. 54% of Async Wait fixes use waitFor-style synchronisation. 47% of order-dependency cases depend on external resources.
  - The paper notes the common practice is rerunning a suspected failure, which is costly and can miss low-probability flakes.
- **Romano et al., "An Empirical Analysis of UI-based Flaky Tests", ICSE 2021** [V] ([arXiv 2103.02669](https://arxiv.org/abs/2103.02669))
  - Dataset: 235 flaky UI tests from 62 web and Android projects. The abstract argues UI tests are "unsuited for detection techniques involving" reruns, because of their cost and complexity.
  - Four root-cause groups: Async Wait (105, 45.1%), Environment (platform issue 34, layout difference 10), Test Runner API issues (DOM selector 16, incorrect test-runner interaction 24), Test Script Logic (incorrect resource load order 22, test order dependency 12, unordered collections 5, randomness 5, time 1).
  - Async Wait sub-causes: resource rendering 61, animation timing 26, network resource loading 19.
  - Fixes: Fix await mechanism 43, add or increase delay 21, refactor logic 75, remove test 73, disable animations 4, dependency fixes 19.
- **Hashemi, Tahir, Rasheed, "An Empirical Study of Flaky Tests in JavaScript", ICSME 2022** [V] ([arXiv 2207.01047](https://arxiv.org/abs/2207.01047))
  - 452 commits analysed. Top causes: concurrency (async waits, races, deadlocks), then OS-specific behaviour and network instability.
  - More than 80% of flaky tests were fixed to eliminate flakiness. Others were skipped, quarantined or removed.
- **Strandberg et al., "Intermittently Failing Tests in the Embedded Systems Domain", ISSTA 2020** [V] ([arXiv 2005.06826](https://arxiv.org/abs/2005.06826)): at system level, "the test environment itself is often the cause of intermittence". Intermittent tests took more effort to diagnose. The paper ends with a nine-question diagnostic checklist: ranges, testing context, shared resources, resource leaks, network assumptions, randomness, test-framework subsystems, ongoing refactoring, and intermittent product issues.

### 5.2 Scale and impact (Google and others)

- **Micco, Google Testing Blog, 2016** [V] ([post](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html))
  - About 1.5% of all test runs report a flaky result. Almost 16% of tests have some flakiness. About 84% of pass-to-fail transitions involve a flaky test.
  - Mitigations: rerun only failing tests, automatic reruns, a flaky mark that "report[s] a failure only if it fails 3 times in a row" (the post itself names this rule's weakness), and auto-quarantine with an automatically filed bug, which "could easily mask a real race condition".
- **Listfield, Google Testing Blog, 2017** [V] ([post](https://testing.googleblog.com/2017/04/where-do-our-flaky-tests-come-from.html))
  - About 63k of 4.2M tests had a flaky run in a week. By engineer-chosen size: 0.5% small, 1.6% medium, 14% large tests were flaky.
  - Binary size (r² 0.82) and RAM use (r² 0.76) correlate with flakiness. Java WebDriver 10.45%, Python WebDriver 18.72%, Android emulator 25.46% of tests flaky (vs 1.65% overall). "Size is more predictive than tool."
- **Memon et al., "Taming Google-Scale Continuous Testing", ICSE-SEIP 2017** [V] ([Google Research page](https://research.google/pubs/taming-google-scale-continuous-testing/), DOI 10.1109/ICSE-SEIP.2017.16)
  - Of 5.56M test targets, 115,160 both passed and failed. 46,694 of those were flakes, leaving 1.23% of targets that found real breakages or fixes.
  - The authors avoided "rerun tests that failed recently" heuristics because "we would end up mostly re-running flaky tests". Tests closer to the changed code, and code recently changed by more than 3 developers, break more.
- **Haben et al., "The Importance of Discerning Flaky from Fault-triggering Test Failures" (Chromium), 2023** [V] ([arXiv 2302.10594](https://arxiv.org/abs/2302.10594)): flaky-*test* predictors reached 99.2% precision but would miss about 76.2% of regression faults. Failure-level prediction with new features raised MCC from 0.20 to 0.42.
- **Habchi et al., qualitative study of 14 practitioners** [V] ([arXiv 2112.04919](https://arxiv.org/abs/2112.04919)): flakiness also comes from component interactions, the testing infrastructure, and external factors. Key mitigations are guidelines and a stable infrastructure.

### 5.3 Detection techniques

- **DeFlaker, ICSE 2018** [V] ([PDF](https://www.cs.cornell.edu/~legunsen/pubs/BellETAL18DeFlaker.pdf))
  - Marks a newly failing test as flaky if it "did not execute any of the changes". It found 1,874 flaky tests from 4,846 failures with a 1.5% false-alarm rate, and 95.5% recall vs 23% for Maven's rerun detector.
  - In a rerun study of 5,328 new failures: same-JVM reruns detected 1,162. Adding fresh-JVM ("Fork") reruns gave 4,186, and adding machine reboot ("Reboot") gave 5,075.
  - [I] The cleaner the rerun environment, the more failures turn out to be flaky. A rerun on the same polluted node or device is weak evidence.
- **iDFlakies, ICST 2019** [V] ([NSF PAR record](https://par.nsf.gov/biblio/10101224)): reorders and reruns tests to detect flakiness and partially classify it as order-dependent (OD) or not. Of 422 flaky tests, 50.5% were OD.
- **FlakeFlagger, ICSE 2021** [V] ([PDF](https://par.nsf.gov/servlets/purl/10273428))
  - 24 projects rerun 10,000 times each, and some known flaky tests still never flaked.
  - Predictive features: test execution time, coverage of source code, coverage of changed lines, third-party library usage. Test smells and tokens were not strong indicators.
  - 380 false positives vs 4,674 for the prior classifier.
- **Apple (Kowalczyk et al., ICSE-SEIP 2020)** [V] ([ICSE page](https://2020.icse-conferences.org/details/icse-2020-Software-Engineering-in-Practice/2/Modeling-and-Ranking-Flaky-Tests-at-Apple)): models flakiness via *entropy* (randomness of results) and *flipRate* (temporal variation). Reduced flakiness by 44% with less than 1% loss in fault detection. [U] I could not access the exact formulas.
- **Westermo q-score** [V] ([Strandberg 2020](https://arxiv.org/abs/2005.06826)): treat the verdict sequence (pass, fail, invalid) as a Markov chain. q = 1 − diag(N)/sum(N), the fraction of consecutive runs whose verdict changed, measured over a moving window of about 8–15 verdicts. The worked example gives q = 0.57 for an 8-run window.
- **Fair (Chromium), 2021** [V] ([arXiv 2111.03382](https://arxiv.org/abs/2111.03382))
  - False alerts were 81% of 1.19M failures (969,417 vs 225,762).
  - History alone was insufficient: 16% of false alerts had flakeRate = 0 and failRate = 0 over the previous 35 commits.
  - Legitimate failures ran longer (median 1.09 s vs 0.41 s). Random Forest on duration, status and TF-IDF of stack trace, stderr, crash log, command and test source reached MCC up to 95% (98% for GUI tests). Run duration was the most important feature.
- **Failure de-duplication (Alshammari et al. 2024)** [V] ([arXiv 2401.15788](https://arxiv.org/abs/2401.15788)): matching new failures to previously seen flaky or true failures over 230,439 failure messages was extremely effective (100% specificity) on some projects and "entirely ineffective" on others. Flaky symptoms can resemble true failures.

### 5.4 History-based flakiness signals you can compute [P]

From the sources above, the practical signals are listed below. Thresholds are proposals to be tuned on your data.

| Signal | Definition | Typical use | Basis |
|---|---|---|---|
| Rerun-pass in the same run | FAIL then PASS for the same test, same build, same environment | Strong flaky (or transient ENV) indicator for *this* failure | Google, Buildkite "passed on retry", Datadog, Chromium LUCI, RF merge message |
| Flip rate / q-score | Status changes ÷ (runs − 1) over the last W runs (W ≈ 10–30) | Ranks how intermittent a test is | Westermo q-score (W 8–15); ReportPortal "% of switches" (default 30 launches); Apple flipRate |
| Failure rate (7-day) | Fails ÷ runs | Quarantine policy trigger; also detects "always failing" tests, which are *not* flaky | Datadog failure-rate policy; Fair failRate |
| Flakiness score | P(fail and eventually pass on retry) | Prior for "this failure is a flake" | Launchable |
| First failure after a long green streak with no test or app change | Transition without a change | Prior toward flaky or ENV | Memon, DeFlaker |
| First failure that coincides with an app build or test-code change | Transition with a change | Prior toward PROD or SCRIPT | DeFlaker (changed-code coverage), Chromium blamelist |
| Duration anomaly | Elapsed vs historical median for the same test or step | Timeouts and slow environment | Fair (duration), Buildkite duration monitor |

[P] Caveats to encode:
- A test with a high flip rate can still fail for a real bug ([Haben 2023](https://arxiv.org/abs/2302.10594)). Per-failure evidence must override the per-test prior.
- With 17k tests a day, a daily run gives one sample per test per day. A 30-run window is about a month of history.

---

## 6. Automated classification research: features, methods, results

| Work (venue) | Setting | Classes | Method | Most predictive features / insight | Reported result |
|---|---|---|---|---|---|
| Herzig & Nagappan (ICSE-SEIP 2015) [V] ([MSR page](https://www.microsoft.com/en-us/research/publication/empirically-detecting-false-test-alarms-using-association-rules/)) | Windows 8.1, Dynamics AX system and integration tests | false alarm (faulty test or infrastructure) vs real | Association rules over patterns of failing test steps | Which test steps fail together | Mean precision 0.85–0.90 while catching 34–48% of false alarms (>10k classifications per product) |
| Jiang et al., CAM (ICSE 2017) [V] ([arXiv](https://arxiv.org/abs/1703.00768)) | Huawei system and integration testing, >14k alarms | 7 causes (section 4) | IR: similarity of the new test log to historical labelled logs, with log selection and highlighting of differences | Similar historical logs. Shows the diff to the tester | Accuracy 58.3% / 65.8%. About 72% after 2 months deployed, about 3x the earlier regex strategy. About 0.1 s per analysis |
| Amar & Rigby, LogFaultFlagger (ICSE 2019) [V] ([ICSE page](https://2019.icse-conferences.org/details/icse-2019-Technical-Papers/54/Mining-Historical-Test-Logs-to-Predict-Bugs-and-Localize-Faults-in-the-Test-Logs)) | Ericsson base-station testing | product fault vs not; fault-line localisation | Remove lines that also appear in passing logs, TF-IDF, exclusive KNN | Lines unique to failing logs | Finds 89% of faults while flagging <1% of failed log lines. Deployed daily |
| Golagha et al. (ISSTA 2019) [V] ([ISSTA page](https://conf.researchr.org/details/issta-2019/issta-2019-Technical-Papers/22/Failure-Clustering-Without-Coverage)) | Automotive hardware-in-the-loop, 8,743 failing tests, 1,531 faults | clusters (one fault per cluster) | Clustering on non-code features; inspect one representative per cluster | Clustering reduces volume | >60% less analysis time; >80% of faults found via representatives only |
| Lampel et al. (FSE 2021) [V] ([PDF](https://www.se.cs.uni-saarland.de/publications/docs/LJA+21.pdf)) | Mozilla CI, >2M job runs | intermittent (flaky or infra) vs real | Classifiers on job telemetry | Runtime, CPU load, OS version, platform | Precision 73% average (up to 100% for some suites, "good enough for fully automated classification"); recall 82% (up to 94%) |
| Haben et al., Fair (2021) [V] ([arXiv](https://arxiv.org/abs/2111.03382)) | Chromium CI, >1M failures | false alert vs legitimate | Random Forest, TF-IDF of artefacts | Run duration first, then artefact keywords | MCC up to 95% (98% GUI tests) |
| Linnaeus (Ericsson, WAIN 2021) [V] ([arXiv](https://arxiv.org/abs/2103.06927)) | Ericsson CI, logs collected back from the time of failure | team-defined | Reusable ML log-classification pipeline | Limited labelled data handling | The search snippet says it returns a label and a confidence score; [U] the abstract has no metrics |
| Sarika et al. (Ericsson, 2023; venue not verified) [V] ([arXiv](https://arxiv.org/abs/2306.07653)) | Kubernetes microservice tests | cluster / artifactory / microservice / CI-CD tests / environment | SVM, KNN, RF, GBC, MLP compared | Cluster logs | Random Forest gave good accuracy at lower cost; [U] numbers not in the abstract |
| Gao et al., NCChecker (FSE Companion 2024) [V] ([arXiv](https://arxiv.org/abs/2405.02922)) | Telecom company, 7,159 failed logs | bug / environment / script / third-party | Log abstraction, "diff with pass" step, lookup table of critical log events | Log events absent from passing logs; critical event lookup | Macro F1 71.9% (vs CAM 55.0%, LogFaultFlagger 37.8%). Per class F1: bug 83.1, env 63.4, script 61.2, third-party 80.0 |
| FlaXifyer (TELUS, FSE 2026 Industry) [V] ([arXiv](https://arxiv.org/abs/2601.22264)) | 2,458 intermittent job failures | intermittent-failure categories (network, infrastructure, resource exhaustion, non-deterministic tests …) | Few-shot fine-tuned pre-trained language models (12 examples per class) | LogSift highlights influential log lines (74.4% less review effort) | Macro F1 84.3%, Top-2 accuracy 92.0% |
| LogSage (ByteDance, 2025 preprint) [V] ([arXiv](https://arxiv.org/abs/2506.03691)) | CI/CD failures | root cause plus remediation | LLM: token-efficient log filtering, structured diagnostic prompting, RAG over historical fixes | Log pre-filtering is critical | >98% precision on 367 GitHub CI failures; >80% end-to-end precision over 1.07M executions in a 1-year deployment |
| Jansson et al., Westermo (PROFES 2026) [V] ([arXiv](https://arxiv.org/abs/2609.21843)) | Nightly test failures (embedded networking) | RCA report | Tool-augmented LLM, single-agent vs multi-agent | Access to test metadata and logs | Six practitioners rated quality; no consistent advantage for multi-agent. Single-agent faster and cheaper |

**Cross-cutting conclusions**
- [V]/[I] Similarity to labelled history is the workhorse: ReportPortal, CAM, LogFaultFlagger and NCChecker all depend on it. It only works once you have labelled history, which is a cold-start problem for a new system.
- [V] **Diff against passing runs** is repeatedly the most useful log transformation (LogFaultFlagger, NCChecker "diff with pass"). [I] For RF, compare the failing test's keyword tree and messages with its last passing run.
- [V] Huawei's regex-based strategy was about 3x worse than CAM after deployment ([CAM](https://arxiv.org/abs/1703.00768)). [I] Hand-written regex rules alone will plateau. Use them for the high-precision, deterministic signatures, and use similarity or ML for the rest.
- [V] Run-time and telemetry features matter (Fair duration; Lampel CPU load, OS, platform). [I] Record environment metadata per run: browser and driver versions, device ID, OS, grid node, app build.
- [V] Per-class performance is uneven. Environment and script classes are typically harder than "bug related" (NCChecker per-class F1), and accuracy depends heavily on the project (Alshammari 2024).
- [U] I found no peer-reviewed study that benchmarks an LLM on exactly your six-way classification of Robot Framework UI failures. The closest are FlaXifyer (intermittent sub-categories), LogSage (CI root cause) and Westermo (qualitative). Treat LLM accuracy for your case as unknown until measured.

---

## 7. Human triage process and a derived diagnostic path

### 7.1 What first-party triage documentation says [V]

- **Chromium sheriffing loop** ([sheriff.md, v78](https://chromium.googlesource.com/chromium/src/+/refs/tags/78.0.3890.0/docs/sheriff.md))
  - Loop order: (1) Is the tree closed? (2) Are there consistent failures? (3) Other red bots? (4) Bug queue. (5) Proactive flake hunting.
  - For a failed test step:
    - Focus on the first failed test in a suite, "since a failing test will sometimes disturb the state of the test runner".
    - Ignore "flaky failures" first and look at "deterministic" ones, which means "failed twice in a row".
    - If related tests failed together, suspect the blamelist or a shared framework or service flake. If a single test failed, check the blamelist, then the flakiness dashboard. If many tests failed, look for common symptoms in a handful at random.
    - Slow tests time out flakily on slow bots.
    - Purple means infra failure, and a red run after a purple run can come from corrupt disk state.
  - For flaky tests: find where in history the flakes started and revert the culprit, or disable the test narrowly.
- **Chromium gardener priorities** ([gardener.md](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/gardener.md)): keep the tree open, then prevent new failures, then repair existing ones. Revert suspected culprit CLs. If no culprit is found, assign to the test author or most recent significant editor, else a proximal OWNER.
- **Mozilla sheriffs** ([sheriffed intermittents](https://firefox-source-docs.mozilla.org/testing/sheriffed-intermittents/index.html)): decide "regression or intermittent". Match the failure to an existing *single tracking bug per test file path*. File a new one when none exists. Escalate at high frequency (about 30 failures a week).
- **Westermo** ([Strandberg 2020](https://arxiv.org/abs/2005.06826); [Jansson 2026](https://arxiv.org/abs/2609.21843)): practitioners inspect heterogeneous logs from multiple sources and decide whether the origin is product software, test framework, test environment, or their interaction. The nine-question checklist (section 5.1) is a structured human diagnostic.
- **Selenium's per-exception guidance** ([selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/)), for example for NoSuchElement: first confirm you are on the expected page and that previous steps succeeded, then the wait strategy, then the locator.
- [U] I did not find a primary, published, step-by-step triage procedure written *specifically* for Selenium or Appium UI suites by a major vendor. The flow below is derived from the general CI-triage sources above.

### 7.2 Derived diagnostic path (PROPOSAL)

> **[P] This is a draft decision flow derived from the sources above. It is not a published procedure.** Each step either emits a verdict (and stops), or adds evidence and continues. Steps 1–4 are run- or cluster-level and should be evaluated once per run before per-test analysis, so that one infrastructure incident does not produce 300 separate investigations.

0. **Normalise the failure** (per failed test):
   - Extract the innermost failing keyword and its library, the final message, and the setup/body/teardown position.
   - Extract the failure timestamp and elapsed time, environment metadata, screenshot and page-source presence, and any rerun or merge record.
   - Build a *fingerprint*: exception or message template with locators, numbers, IDs, timestamps and URLs masked (as ReportPortal's "exclude numbers" option does).
1. **Maintenance window check.** If the failure timestamp (or the run's start or end) overlaps a declared maintenance window for the target environment or a dependency → **MAINT**. Supporting evidence: maintenance banner, HTTP 503 page, or a cluster of ENV-type errors inside the window.
2. **Run-level health and clustering** (Chromium "many tests failed" heuristic; ReportPortal unique errors; Golagha clustering):
   - (a) Did session creation or connection fail? → **ENV**.
   - (b) Do many unrelated tests share the same fingerprint in the same time window, environment, node or device? → one cluster verdict, usually **ENV**, sometimes one **PROD** outage. Pick a representative, analyse it, and propagate the verdict to members.
   - (c) Did the failures all happen on one node, device or browser version? → **ENV**.
3. **Fixture attribution.**
   - `Parent suite setup failed` / `Suite setup failed` → analyse the setup failure once and propagate.
   - Teardown-only failure → analyse separately with low severity.
   - Collateral "execution stopped" → attribute to the first fatal failure.
4. **Deterministic test-code signatures** (unknown keyword, variable not found, invalid selector, unsupported locator prefix, import errors) → **SCRIPT**.
5. **Known-issue match** (section 9):
   - Fingerprint, test ID or test path matches an open product bug → **PROD (known)**. Link it and do not file a duplicate.
   - Matches an open automation or test-data ticket → SCRIPT or DATA (known).
   - Matches a known-flaky record → FLAKY candidate. Continue to step 6 to confirm for this failure.
   - Matches only resolved issues → possible regression; flag it.
6. **History and rerun:**
   - Rerun-pass in the same build → **FLAKY** (or transient ENV if the cluster check in step 2 shows co-occurring infra errors).
   - High flip rate with no relevant change → FLAKY prior.
   - Long green history, then a new consistent failure → go to step 7 (change-driven).
   - Always failing → not flaky. A stale test or open bug is likely.
7. **Change correlation** (Chromium "blamelist"; DeFlaker; Memon):
   - Test code, locator file or test data changed since the last pass, and the failing keyword touches the change → **SCRIPT** (or **DATA**).
   - The app build changed and the test did not → PROD or SCRIPT (UI changed). Go to step 8.
8. **Artefact inspection** (screenshot and page source; this is where an LLM or vision step helps most [I]):
   - Error page or blank page → ENV (or PROD if only this feature fails).
   - Expected page, but the target element exists with changed attributes → **SCRIPT** (locator maintenance), unless the requirement says the element must keep that ID or text.
   - Expected page, element missing or wrong value shown, requirement unchanged → **PROD**.
   - Data-state message (not found, already exists, insufficient …) → **DATA**.
   - Overlay or modal covering the target → FLAKY, ENV, or PROD depending on what the overlay is.
9. **Assertion semantics:**
   - Expected vs actual differ, and the expected value is derived from test data → DATA vs PROD. Check the data-setup keywords and data freshness.
   - Expected value hard-coded and requirements changed → SCRIPT.
10. **Emit the verdict:** category, confidence, evidence list, next action. If no step produced a verdict above threshold → **To Investigate** (human queue), with the gathered evidence attached.

---

## 8. Draft starter ruleset (PROPOSAL)

> **[P] All rules below are proposals.** Confidence hints are qualitative (H/M/L). A suggested starting mapping is H ≈ 0.9, M ≈ 0.7, L ≈ 0.5, to be calibrated (section 10). The "Source/rationale" column says what each rule rests on. Evaluate rules in precedence order: the first rule that fires with H confidence decides. M and L verdicts accumulate as evidence and can be overridden by later, more specific evidence.

**Precedence tiers:** T1 maintenance → T2 run-level environment and clusters → T3 fixture attribution → T4 deterministic script signatures → T5 known-issue match → T6 rerun and history → T7 change correlation → T8 artefact and assertion analysis → T9 fallback.

| ID | Tier | When (conditions) | Category | Conf. | Next action | Source / rationale |
|---|---|---|---|---|---|---|
| R01 | T1 | Failure time (or the whole test interval) inside a declared maintenance window for the target environment or service | MAINT | H (if the schedule is authoritative) | Ignore; auto-rerun after the window closes | No tool primitive exists; Chromium has a tree "Maintenance" state ([gardener.md](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/gardener.md)). Your definition needed (open question 1) |
| R02 | T1 | Maintenance banner, HTTP 503 or a known "under maintenance" page in page source or screenshot, with no window on record | MAINT (or ENV) | M | Rerun later; ask the environment owner to register the window | [I] |
| R03 | T2 | `session not created` / `SessionNotCreatedException` / Appium `A new session could not be created` | ENV | H | Rerun on another node or device after a health check; alert the infra owner if it affects >1 test | [W3C](https://www.w3.org/TR/webdriver2/#errors), [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/), [Appium](https://github.com/appium/appium/blob/master/packages/base-driver/lib/protocol/errors.ts) |
| R04 | T2 | `chrome not reachable` / `disconnected` / UiAutomator2 `instrumentation process is not running` / `Could not proxy command to the remote server` / WARN "could not be run on failure" right after the primary error | ENV | M–H | Rerun on a clean node or device; collect logcat or driver logs | [status.cc](https://chromium.googlesource.com/chromium/src/+/main/chrome/test/chromedriver/chrome/status.cc), [UIA2](https://github.com/appium/appium-uiautomator2-driver/blob/master/lib/uiautomator2-server/core.ts), [proxy.ts](https://github.com/appium/appium/blob/master/packages/base-driver/lib/jsonwp-proxy/proxy.ts) |
| R05 | T2 | `invalid session id` and no `Close Browser` / `Close Application` / quit earlier in the test or suite | ENV | M | Rerun | [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/) |
| R06 | T2 | `insecure certificate` | ENV | H | Notify the environment owner; no rerun until fixed | [W3C](https://www.w3.org/TR/webdriver2/#errors) |
| R07 | T2 | ≥ K tests (for example K = 10, or ≥ 5% of the run) from ≥ 3 unrelated suites share one fingerprint within a short window, or all failures are on one node, device or browser version | ENV (cluster) | M (H if the fingerprint is an R03–R06 type) | One incident ticket for the cluster; rerun the cluster after recovery | [Chromium sheriff doc](https://chromium.googlesource.com/chromium/src/+/refs/tags/78.0.3890.0/docs/sheriff.md), [RP unique errors](https://reportportal.io/docs/analysis/UniqueErrorAnalysis/), [Golagha 2019](https://conf.researchr.org/details/issta-2019/issta-2019-Technical-Papers/22/Failure-Clustering-Without-Coverage) |
| R08 | T3 | Message starts `Parent suite setup failed:` or `Suite setup failed:` | Inherit the setup verdict | = setup | Analyse the setup once; propagate; avoid duplicate tickets | [status.py](https://github.com/robotframework/robotframework/blob/master/src/robot/running/status.py), [User Guide](https://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html#test-and-suite-statuses) |
| R09 | T3 | Test body passed; only `Teardown failed:` | Classify the teardown error (often ENV or SCRIPT) | M | Low priority; no product bug unless the teardown asserts product state | [status.py](https://github.com/robotframework/robotframework/blob/master/src/robot/running/status.py) |
| R10 | T3 | `Test execution stopped due to a fatal error.` / exit-on-failure messages | Collateral (link to the first failure) | H | No separate action | [status.py](https://github.com/robotframework/robotframework/blob/master/src/robot/running/status.py) |
| R11 | T4 | `No keyword with name`, `Multiple keywords with name`, `Variable '…' not found.`, library import error in `<errors>`, `invalid selector`, `Element locator with prefix … not supported` | SCRIPT | H | File an automation bug to the test owner; do not rerun | RF and library sources (section 2) |
| R12 | T4 | `No browser is open.` / `No application is open` and no earlier session crash in the suite | SCRIPT | M | Fix setup or ordering | SeleniumLibrary / AppiumLibrary sources |
| R13 | T5 | Fingerprint or test path matches an **open** Jira Bug (product) with similarity ≥ S_high | PROD (known) | H | Link to the bug; comment with the occurrence; no new ticket | [Mozilla sheriffing](https://firefox-source-docs.mozilla.org/testing/sheriffed-intermittents/index.html), [RP AA copies the BTS link](https://reportportal.io/docs/analysis/AutoAnalysisOfLaunches/) |
| R14 | T5 | Match an open automation-bug or test-data ticket | SCRIPT or DATA (known) | H | Link; no rerun | as above |
| R15 | T5 | Match only a **resolved** product bug | PROD (possible regression) | M | Human review; consider reopening | [I] |
| R16 | T5 | Test is on the known-flaky or quarantine list | FLAKY prior | M (do **not** jump to H) | Continue to T6 to confirm for this failure; never auto-ignore on this alone | [Haben 2023](https://arxiv.org/abs/2302.10594), [Micco 2016](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html) |
| R17 | T6 | Rerun in the same build passed (RF merge message: new PASS, old FAIL) and no ENV cluster at that time | FLAKY | M–H | Record a flake; quarantine if the flake rate over 30 runs is ≥ Q (for example 10%); file a stabilisation task | [merger.py](https://github.com/robotframework/robotframework/blob/master/src/robot/result/merger.py), [Buildkite](https://buildkite.com/docs/pipelines/configure/tests/workflows/monitors), [Datadog](https://docs.datadoghq.com/tests/flaky_management/) |
| R18 | T6 | Rerun passed **and** an ENV cluster or infra errors co-occurred | ENV (transient) | M | No test action; count toward environment stability metrics | [Habchi 2021](https://arxiv.org/abs/2112.04919), [Lampel 2021](https://www.se.cs.uni-saarland.de/publications/docs/LJA+21.pdf) |
| R19 | T6 | Rerun failed with the same fingerprint | Not flaky for this failure; go to T7/T8 | – | Continue | Chromium "deterministic" = failed twice ([sheriff doc](https://chromium.googlesource.com/chromium/src/+/refs/tags/78.0.3890.0/docs/sheriff.md)); caveat from [DeFlaker](https://www.cs.cornell.edu/~legunsen/pubs/BellETAL18DeFlaker.pdf) (same-environment reruns under-detect) |
| R20 | T6 | q-score or flip rate over the last 10–15 runs ≥ 0.3, no test or app change, timing-type error (rows 12–14, 17 in section 3) | FLAKY | M | Rerun once in a fresh session; quarantine candidate | [Strandberg 2020](https://arxiv.org/abs/2005.06826), [RP flaky widget](https://reportportal.io/docs/dashboards-and-widgets/FlakyTestCasesTableTop50/) |
| R21 | T6 | Test failed in every one of the last N runs (N ≥ 5) with the same fingerprint | Not flaky: PROD (open bug), SCRIPT (stale test) or DATA | M | Route by T5/T7/T8; if unlinked, human triage | Datadog excludes 100%-failing tests from "Fixed" ([docs](https://docs.datadoghq.com/tests/flaky_management/)) |
| R22 | T7 | Test, resource or locator file changed since the last pass, and the failing keyword or locator is in the diff | SCRIPT | M–H | Assign to the commit author | Chromium blamelist and culprit revert ([sheriff doc](https://chromium.googlesource.com/chromium/src/+/refs/tags/78.0.3890.0/docs/sheriff.md), [gardener.md](https://chromium.googlesource.com/chromium/src/+/HEAD/docs/gardener.md)) |
| R23 | T7 | Test-data files or seed scripts changed since the last pass, and the failure involves those records | DATA | M | Fix or refresh data; rerun | [I] |
| R24 | T8 | Locator-not-found or element wait failure (section 3, rows 11–12), page source shows the expected page, and a uniquely matching element exists with a changed id, name or text; app build changed | SCRIPT (locator maintenance) | M | File an automation bug with a suggested new locator; if the change was unintended, PROD | [selenium.dev NoSuchElement](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/) |
| R25 | T8 | Locator-not-found, and page source or screenshot shows an error, blank or login page | ENV (or PROD if only this feature's endpoint errors) | M | Check environment health; rerun | [I] |
| R26 | T8 | Locator-not-found or assertion failure, expected page, element or value missing, requirement (Confluence or Jira) still specifies it, app build changed, test unchanged | PROD | M | File a product bug (human confirmation recommended at first) | [I]; [Memon 2017](https://research.google/pubs/taming-google-scale-continuous-testing/) (breakages near the changed code) |
| R27 | T8 | Failure message, app message or page text contains data-state terms (not found, already exists, duplicate, insufficient, expired, locked), or a data-setup keyword (API, DB seeding) failed | DATA | M | Refresh or recreate the data; rerun | [I] |
| R28 | T8 | `StaleElementReference`, `element click intercepted`, `element not interactable` with no app or test change and rerun-pass or flip history | FLAKY (async wait or animation) | M | Rerun; stabilisation task (explicit waits) | [Luo 2014](https://mir.cs.illinois.edu/lamyaa/publications/fse14.pdf), [Romano 2021](https://arxiv.org/abs/2103.02669) |
| R29 | T8 | `element click intercepted` and the screenshot shows an environment-specific overlay (cookie banner, environment notice) | ENV (or SCRIPT: test should handle it) | M | Handle the overlay in setup | [selenium.dev](https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/) |
| R30 | T8 | `Test timeout … exceeded.` or `timeout` on an isolated test whose duration trend is rising | PROD (performance) or SCRIPT | L | Human review with the duration chart | [Fair duration](https://arxiv.org/abs/2111.03382), [Buildkite duration monitor](https://buildkite.com/docs/pipelines/configure/tests/workflows/monitors) |
| R31 | T8 | `unexpected alert open` | PROD or SCRIPT | L | Human review; read the alert text | [W3C](https://www.w3.org/TR/webdriver2/#errors) |
| R32 | T8 | `Several failures occurred:` | Classify each sub-failure; overall verdict = highest-severity category (proposal: PROD > DATA > SCRIPT > ENV > FLAKY > MAINT), with all others listed | per part | Per part | [User Guide](https://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html) (open question 4) |
| R33 | – | Test status SKIP (including `robot:skip-on-failure`) | Not a failure; report separately | H | None (count "not ready" tests) | [User Guide](https://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html#test-and-suite-statuses) |
| R34 | T9 | Nothing above reached threshold | To Investigate | – | Human queue with collected evidence; the human label feeds back into history | [ReportPortal To Investigate](https://reportportal.io/docs/work-with-reports/InvestigationOfFailure/) |

**Category → default next action** [P]

| Category | Default next action |
|---|---|
| MAINT | Ignore, then auto-rerun after the window |
| ENV | Rerun after a health check or on a different node or device; incident ticket if clustered |
| FLAKY | Rerun once in a fresh session; quarantine when the flake rate exceeds the threshold; stabilisation ticket |
| SCRIPT | File an automation bug to the test owner; no rerun |
| DATA | Refresh or repair the data, then rerun; ticket to the data owner if recurring |
| PROD | Link to the known bug, or file a new bug (human confirmation in the early phase) |
| To Investigate | Human triage |

---

## 9. Known-bug matching via Jira and Confluence

### 9.1 API facts [V]

**Jira Cloud search**
- Use `GET` or `POST /rest/api/3/search/jql` ("enhanced search"). The older `GET`/`POST /rest/api/3/search` is marked **"Deprecated … Endpoint is currently being removed"** ([Jira Cloud REST v3, Issue search](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-search/)).
- Parameters: `jql`, `nextPageToken` (token pagination), `maxResults`, `fields`, `expand`, `properties`, `fieldsByKeys`, `failFast`, `reconcileIssues`.
- Recent updates might not be visible immediately; `reconcileIssues` gives read-after-write consistency.
- Scopes: `read:jira-work` (classic). Results are limited to issues the caller can browse.
- `POST /rest/api/3/search/approximate-count` also exists.

**JQL text search** ([JQL operators](https://support.atlassian.com/jira-software-cloud/docs/jql-operators/); [text-field search](https://support.atlassian.com/jira-software-cloud/docs/search-for-issues-using-the-text-field/); [JQL fields](https://support.atlassian.com/jira-software-cloud/docs/jql-fields/))
- `~` (CONTAINS) works on text fields. `text ~ "…"` searches Summary, Description, Environment, Comments and all text custom fields.
- Exact phrases need inner escaped quotes: `text ~ "\"exact phrase\""`.
- Boolean AND, OR, NOT, `+` and `-` are supported, and `*` works as a trailing wildcard.
- **"Operators for fuzzy search (~), proximity search, and term boosting (^) no longer have an affect on searches."**
- Some characters are reserved and need quoting.
- Where `resolution` is unavailable, use `statusCategory = Done` to mean resolved.

**Confluence Cloud search**
- `GET /wiki/rest/api/search?cql=…` with `cursor` pagination (`_links.next`), `limit`, `excerpt`, `cqlcontext`. Scope `search:confluence`. User-specific CQL fields are no longer supported on this endpoint ([Confluence REST v1, Search](https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-search/)).
- CQL `text` is a master field over title, content body and labels. `space`, `type`, `label`, `title`, `ancestor`, `lastmodified` are available ([CQL fields](https://developer.atlassian.com/cloud/confluence/cql-fields/)).

### 9.2 Similarity techniques from research [V]

- **Runeson et al., ICSE 2007** (Sony Ericsson; [dblp](https://dblp.org/rec/conf/icse/RunesonAN07), DOI 10.1109/ICSE.2007.32): NLP (vector space) duplicate detection. About 2/3 of duplicates "can possibly be found".
- **Wang et al., ICSE 2008** ([PDF](https://www.cs.utsa.edu/~xwang/papers/icse08.pdf)): adding *execution information* to text found 67–93% of duplicates in Firefox, vs 43–72% with natural language alone.
- **ReBucket, ICSE 2012** (Microsoft Windows Error Reporting; [DOI](https://doi.org/10.1109/ICSE.2012.6227111)): call-stack similarity to bucket duplicate crash reports. Average F-measure about 0.88 on five products.
- **ReportPortal**: OpenSearch TF-IDF retrieval, then XGBoost on cosine similarities, then thresholds 50% (auto) and 40% (suggest). Separate fields for message, stack trace, exceptions, URLs, paths and numbers ([docs](https://reportportal.io/docs/analysis/AutoAnalysisOfLaunches/)).
- **Mozilla**: match intermittents at the **test path** level ("single tracking bug"), not per error message ([docs](https://firefox-source-docs.mozilla.org/testing/sheriffed-intermittents/index.html)).
- **Alshammari 2024**: failure-message de-duplication effectiveness varies widely by project ([arXiv](https://arxiv.org/abs/2401.15788)).

### 9.3 Proposed matching approach [P]

1. **Fingerprint every failure**:
   - Exception or error type (W3C code or library template, for example `SeleniumLibrary.ElementNotFound`).
   - The message template with masked values (locators → `<LOC>`, numbers → `<N>`, URLs → `<URL>`, timestamps or IDs → `<ID>`).
   - The innermost failing keyword name and the test path.
   - Hash this as a *stable fingerprint*, and keep the unmasked text for similarity.
2. **Write the fingerprint into Jira** when a bug is filed, for example as a label `fv-fp-<hash>` or a custom field, and include the test path. [I] Future exact matches then become a cheap JQL equality query instead of fuzzy text search.
3. **Candidate retrieval with JQL** (examples; field names depend on your Jira configuration):
   - Exact: `labels = "fv-fp-1a2b3c" AND statusCategory != Done`
   - By test path: `text ~ "\"tests/checkout/payment.robot\"" AND issuetype = Bug ORDER BY updated DESC`
   - By key terms: `project in (APP, QA) AND issuetype = Bug AND statusCategory != Done AND text ~ "\"Payment declined\""` (escape reserved characters; avoid raw stack traces).
4. **Local re-ranking** of the top N candidates:
   - TF-IDF or BM25 cosine on summary and description vs the failure message.
   - Stack or keyword-path similarity (ReBucket-style: the same innermost keywords in the same order).
   - Optional semantic embeddings.
   - Combine with priors: same component or feature, same environment, created after the first occurrence.
5. **Thresholds:** S_high → auto-link (rule R13). S_mid → suggest to a human (like ReportPortal suggestions). Below → no match.
6. **Confluence requirements** (when available): CQL `type = page AND space = REQ AND text ~ "\"<feature name>\""` to fetch the requirement text for assertion-semantics checks (rules R24/R26). [I] Requirements are still being compiled. Treat "no requirement found" as *missing evidence*, not as evidence of a script issue.

---

## 10. Confidence and human-in-the-loop

### 10.1 How tools and papers express confidence [V]

- **ReportPortal**: model probability. Auto-assign at ≥ 50%. Suggestions shown at > 40%, grouped SAME (100%), HIGH (70–99.9%), LOW (40–69.9%). The "AA" label marks automatic decisions, so humans can filter and audit them. "To Investigate" is the default for anything not analysed ([AA](https://reportportal.io/docs/analysis/AutoAnalysisOfLaunches/), [ML suggestions](https://reportportal.io/docs/analysis/MLSuggestions/)).
- **Launchable**: a flakiness score from 0 to 1 interpreted as a probability ([docs](https://help.launchableinc.com/features/unhealthy-tests/)).
- **Buildkite**: transition-count score with alarm and recover thresholds; Bayesian probabilistic flakiness score ([docs](https://buildkite.com/docs/pipelines/configure/tests/workflows/monitors)).
- **Datadog**: policy thresholds on 7-day failure rate, time and branch drive quarantine or disable ([docs](https://docs.datadoghq.com/tests/flaky_management/)).
- **Lampel et al.**: per-suite precision decides whether full automation is acceptable. Some suites reached 100% precision, which is "good enough for fully automated classification" ([PDF](https://www.se.cs.uni-saarland.de/publications/docs/LJA+21.pdf)).
- **Herzig & Nagappan**: chose a high-precision, partial-recall operating point (precision 0.85–0.90, catching 34–48% of false alarms) ([MSR](https://www.microsoft.com/en-us/research/publication/empirically-detecting-false-test-alarms-using-association-rules/)).
- **FlaXifyer**: reports Top-2 accuracy (92.0%), which supports showing a ranked shortlist to humans ([arXiv](https://arxiv.org/abs/2601.22264)).
- **LLMs**: verbalised confidence "tend[s] to be overconfident". Consistency across multiple samples helps but does not solve it ([Xiong et al., ICLR 2024](https://arxiv.org/abs/2306.13063)).
- **Risks of auto-acting**: Google notes that auto-quarantine "could easily mask a real race condition" ([Micco 2016](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html)), and ignoring flaky-test failures would miss about 76% of regression faults at Chromium ([Haben 2023](https://arxiv.org/abs/2302.10594)).

### 10.2 Guidance (PROPOSAL)

1. **Separate the confidence of the verdict from the risk of the action.** Suggested minimum confidence per action:

| Action | Risk if wrong | Suggested minimum |
|---|---|---|
| Rerun | Low (costs compute only) | M |
| Link to an existing bug | Low to medium | H plus fingerprint or test-path match |
| File a new product bug | Medium (noise for developers) | H, and human confirmation in the first months |
| Quarantine | High (can mask bugs) | Flake history + rerun-pass + no linked open product bug; time-boxed with an auto-filed ticket (Google, Datadog patterns) |
| Ignore (MAINT only) | High | Only with an authoritative maintenance schedule (R01) |

2. **Derive confidence from evidence, not from the LLM's self-report.** Base it on the rule tier that fired, the agreement of independent signals (for example error signature + history + rerun), and the similarity score for known-issue matches. If an LLM is used, sample it several times and use agreement as a signal. Record the evidence list with every verdict.
3. **Calibrate on your own labels.** Run in *shadow mode* first: the agent proposes, and humans decide as today. Measure precision per category and per action. Then set thresholds so each automatic action meets a target precision (for example ≥ 95% for auto-link and auto-rerun). Repeat as models or rules change. ReportPortal's own approach, keeping a retrained model only if it beats the baseline, is a sensible gate.
4. **Keep a permanent To Investigate bucket.** With a top-2 shortlist and evidence, a human can decide quickly. Feed human decisions back as labelled history: ReportPortal retrains after about 100–300 new labels.
5. **Cluster before you triage.** Representative-based review cut analysis time by more than 60% in Golagha et al. With 100+ failures a day, expect a small number of clusters to cover most failures. [I] Your actual cluster distribution is unknown.
6. **Measure regressions missed, not only triage time saved.** Track how many failures auto-labelled FLAKY, ENV or MAINT later turn out to be product bugs (Haben 2023 argues for this evaluation).

---

## 11. Open questions for you

These are decisions or facts the sources cannot settle.

1. **Maintenance period definition.**
   - What counts: planned downtime of the test environment, of a downstream dependency, deployments to the test environment, or database refreshes?
   - Where does the schedule live (calendar, Confluence page, ITSM tool, a status API)? Is it authoritative and in which timezone?
   - Partial maintenance: one service down, the rest up.
   - Should tests that *start* before a window and fail inside it count as MAINT?
2. **Run history.**
   - How many past runs are retained, and in what form (output.xml archive, ReportPortal, a database)?
   - Is test identity stable across renames (needed for flip rate and the per-test history in rules R17–R21)?
   - Do you rerun failed tests automatically today (`--rerunfailed` + `rebot --merge`)? On the same node or device, or a fresh one?
3. **Version and change metadata.** Is each run tagged with the app build, test-code commit, test-data version, environment ID, browser and driver versions, and device ID or OS? Rules R22–R26 and the change-correlation step depend on this.
4. **Multiple causes.** How should a test with several failures (`Several failures occurred`) or cascading failures be reported: one primary category, or a list? Is the severity ordering PROD > DATA > SCRIPT > ENV > FLAKY > MAINT acceptable?
5. **Flaky vs transient environment.** Both pass on rerun. Do you want FLAKY reserved for test-side nondeterminism (async waits, order dependency) and ENV for infra hiccups? This decides who owns the fix.
6. **Script vs product when the UI changed.** Who decides whether a UI change was intended? Until requirements exist in Confluence, is there another source of truth (release notes, Jira stories in the build)?
7. **Data issue boundary.** Is test data owned per test (created in setup) or shared in the environment? Data refreshed by someone else's job looks like ENV or PROD unless you track it.
8. **Jira conventions.**
   - Which projects and issue types hold product bugs vs automation bugs vs data or environment tickets?
   - Can the agent create issues and labels automatically, or only suggest them?
   - Is there an existing "known flaky" or quarantine list?
9. **Automation risk appetite.** Which actions may run without a human (rerun, link, quarantine, ignore)? What precision target is acceptable per action?
10. **Evidence capture.** Can you enable screenshot and page-source capture on *every* failure, including BuiltIn assertion failures? SeleniumLibrary's run-on-failure only covers its own keywords. Can you add a listener or teardown that also saves the URL, page source, browser console logs, Appium logcat or syslog?
11. **Environment telemetry.** Is there a health endpoint or monitoring for the application environments, the Selenium Grid and the device farm, so that ENV can be confirmed directly instead of inferred?
12. **Parallelism and ordering.** Do you use parallel execution (for example pabot) or shared accounts across tests? Order-dependency and shared-data collisions become likely flaky and DATA causes.
13. **Labelled ground truth.** Are past manual triage decisions recorded anywhere? They are needed to calibrate thresholds and to seed similarity-based matching.
14. **Data handling for AI.** Can logs, screenshots and page source be sent to an external LLM, or must analysis stay on-premises? Do they contain personal or production data?

---

## 12. References

**Robot Framework and libraries**
- Robot Framework User Guide 7.5.1. https://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html
- RF output schema (result.xsd). https://github.com/robotframework/robotframework/blob/master/doc/schema/result.xsd
- RF source: result/merger.py, running/status.py, running/timeouts/timeout.py, libraries/BuiltIn.py, running/namespace.py, variables/notfound.py. https://github.com/robotframework/robotframework/tree/master/src/robot
- SeleniumLibrary source (v6.9.0). https://github.com/robotframework/SeleniumLibrary/tree/master/src/SeleniumLibrary
- AppiumLibrary source (v3.2.1). https://github.com/serhatbolsu/robotframework-appiumlibrary/tree/master/AppiumLibrary
- allure-robotframework status mapping. https://github.com/allure-framework/allure-python/blob/master/allure-robotframework/src/listener/utils.py

**WebDriver, Selenium, Appium**
- W3C WebDriver, section 6.6 Errors. https://www.w3.org/TR/webdriver2/#errors
- Selenium, Understanding Common Errors. https://www.selenium.dev/documentation/webdriver/troubleshooting/errors/
- ChromeDriver status.cc. https://chromium.googlesource.com/chromium/src/+/main/chrome/test/chromedriver/chrome/status.cc
- Appium base-driver errors.ts. https://github.com/appium/appium/blob/master/packages/base-driver/lib/protocol/errors.ts
- Appium base-driver driver.ts (newCommandTimeout). https://github.com/appium/appium/blob/master/packages/base-driver/lib/basedriver/driver.ts
- Appium base-driver jsonwp-proxy/proxy.ts. https://github.com/appium/appium/blob/master/packages/base-driver/lib/jsonwp-proxy/proxy.ts
- Appium UiAutomator2 driver uiautomator2-server/core.ts. https://github.com/appium/appium-uiautomator2-driver/blob/master/lib/uiautomator2-server/core.ts

**Test analytics tools**
- ReportPortal, Investigation of failure (defect types). https://reportportal.io/docs/work-with-reports/InvestigationOfFailure/
- ReportPortal, Auto-Analysis. https://reportportal.io/docs/analysis/AutoAnalysisOfLaunches/
- ReportPortal, ML Suggestions. https://reportportal.io/docs/analysis/MLSuggestions/
- ReportPortal, Pattern Analysis. https://reportportal.io/docs/analysis/PatternAnalysis/
- ReportPortal, Unique Error Analysis. https://reportportal.io/docs/analysis/UniqueErrorAnalysis/
- ReportPortal, How models are retrained. https://github.com/reportportal/docs/blob/develop/docs/analysis/HowModelsAreRetrained.md
- ReportPortal, Flaky test cases table widget. https://reportportal.io/docs/dashboards-and-widgets/FlakyTestCasesTableTop50/
- Allure, Categories. https://allurereport.org/docs/categories/
- Allure, Categories file. https://allurereport.org/docs/how-it-works-categories-file/
- Allure, Test statuses. https://allurereport.org/docs/test-statuses/
- Datadog, Flaky Test Management. https://docs.datadoghq.com/tests/flaky_test_management/
- Datadog, Flaky management. https://docs.datadoghq.com/tests/flaky_management/
- Buildkite, Workflow monitors. https://buildkite.com/docs/pipelines/configure/tests/workflows/monitors
- Buildkite, Flaky test management. https://buildkite.com/docs/test-engine/flaky-test-management
- Launchable, Unhealthy tests. https://help.launchableinc.com/features/unhealthy-tests/
- Mozilla Treeherder, failure classifications. https://github.com/mozilla/treeherder/blob/master/treeherder/model/fixtures/failure_classification.json
- Mozilla, Sheriffed intermittent failures. https://firefox-source-docs.mozilla.org/testing/sheriffed-intermittents/index.html
- Chromium, Gardening. https://chromium.googlesource.com/chromium/src/+/HEAD/docs/gardener.md
- Chromium, Sheriffing (v78 snapshot, detailed procedures). https://chromium.googlesource.com/chromium/src/+/refs/tags/78.0.3890.0/docs/sheriff.md

**Flaky test research**
- Luo, Hariri, Eloussi, Marinov. An Empirical Analysis of Flaky Tests. FSE 2014. https://mir.cs.illinois.edu/lamyaa/publications/fse14.pdf (DOI 10.1145/2635868.2635920)
- Romano, Song, Grandhi, Yang, Wang. An Empirical Analysis of UI-based Flaky Tests. ICSE 2021. https://arxiv.org/abs/2103.02669
- Hashemi, Tahir, Rasheed. An Empirical Study of Flaky Tests in JavaScript. ICSME 2022. https://arxiv.org/abs/2207.01047
- Micco. Flaky Tests at Google and How We Mitigate Them. Google Testing Blog, 2016. https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html
- Listfield. Where do our flaky tests come from? Google Testing Blog, 2017. https://testing.googleblog.com/2017/04/where-do-our-flaky-tests-come-from.html
- Memon et al. Taming Google-Scale Continuous Testing. ICSE-SEIP 2017. https://research.google/pubs/taming-google-scale-continuous-testing/ (DOI 10.1109/ICSE-SEIP.2017.16)
- Bell et al. DeFlaker: Automatically Detecting Flaky Tests. ICSE 2018. https://www.cs.cornell.edu/~legunsen/pubs/BellETAL18DeFlaker.pdf (DOI 10.1145/3180155.3180164)
- Lam, Oei, Shi, Marinov, Xie. iDFlakies. ICST 2019. https://par.nsf.gov/biblio/10101224
- Alshammari, Morris, Hilton, Bell. FlakeFlagger: Predicting Flakiness Without Rerunning Tests. ICSE 2021. https://par.nsf.gov/servlets/purl/10273428
- Kowalczyk et al. Modeling and Ranking Flaky Tests at Apple. ICSE-SEIP 2020. https://2020.icse-conferences.org/details/icse-2020-Software-Engineering-in-Practice/2/Modeling-and-Ranking-Flaky-Tests-at-Apple
- Strandberg, Ostrand, Weyuker, Afzal, Sundmark. Intermittently Failing Tests in the Embedded Systems Domain. ISSTA 2020. https://arxiv.org/abs/2005.06826
- Habchi, Haben, Papadakis, Cordy, Le Traon. A Qualitative Study on the Sources, Impacts, and Mitigation Strategies of Flaky Tests. https://arxiv.org/abs/2112.04919
- Haben, Habchi, Papadakis, Cordy, Le Traon. Discerning Legitimate Failures From False Alerts: A Study of Chromium's CI (Fair). https://arxiv.org/abs/2111.03382
- Haben et al. The Importance of Discerning Flaky from Fault-triggering Test Failures: A Case Study on the Chromium CI. https://arxiv.org/abs/2302.10594
- Alshammari, Ammann, Hilton, Bell. 230,439 Test Failures Later: An Empirical Evaluation of Flaky Failure Classifiers. https://arxiv.org/abs/2401.15788

**Failure classification research**
- Herzig, Nagappan. Empirically Detecting False Test Alarms Using Association Rules. ICSE-SEIP 2015. https://www.microsoft.com/en-us/research/publication/empirically-detecting-false-test-alarms-using-association-rules/
- Jiang, Li, Yang, Xuan. What Causes My Test Alarm? ICSE 2017. https://arxiv.org/abs/1703.00768
- Amar, Rigby. Mining Historical Test Logs to Predict Bugs and Localize Faults in the Test Logs. ICSE 2019. https://2019.icse-conferences.org/details/icse-2019-Technical-Papers/54/Mining-Historical-Test-Logs-to-Predict-Bugs-and-Localize-Faults-in-the-Test-Logs
- Golagha, Lehnhoff, Pretschner, Ilmberger. Failure Clustering Without Coverage. ISSTA 2019. https://conf.researchr.org/details/issta-2019/issta-2019-Technical-Papers/22/Failure-Clustering-Without-Coverage
- Lampel, Just, Apel, Zeller. When Life Gives You Oranges: Detecting and Diagnosing Intermittent Job Failures at Mozilla. ESEC/FSE 2021. https://www.se.cs.uni-saarland.de/publications/docs/LJA+21.pdf (DOI 10.1145/3468264.3473931)
- Catovic et al. Linnaeus: A highly reusable and adaptable ML based log classification pipeline. WAIN 2021. https://arxiv.org/abs/2103.06927
- Sarika, Badampudi, Josyula, Usman. Automating Microservices Test Failure Analysis using Kubernetes Cluster Logs. 2023. https://arxiv.org/abs/2306.07653
- Gao, Xue, Hu, Shang, Xia. Easy over Hard: A Simple Baseline for Test Failures Causes Prediction. FSE Companion 2024. https://arxiv.org/abs/2405.02922
- Aïdasso, Bordeleau, Tizghadam. Predicting Intermittent Job Failure Categories for Diagnosis Using Few-Shot Fine-Tuned Language Models. FSE 2026 Industry. https://arxiv.org/abs/2601.22264
- Xu et al. LogSage: An LLM-Based Framework for CI/CD Failure Detection and Remediation with Industrial Validation. 2025. https://arxiv.org/abs/2506.03691
- Jansson, Strandberg, Sörensen, Enoiu, Afzal. Supporting Industrial Test-Failure Analysis with LLM-Based Systems: An Experience Report. PROFES 2026. https://arxiv.org/abs/2609.21843
- Xiong et al. Can LLMs Express Their Uncertainty? ICLR 2024. https://arxiv.org/abs/2306.13063

**Jira, Confluence, duplicate detection**
- Jira Cloud REST API v3, Issue search. https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-search/
- JQL operators. https://support.atlassian.com/jira-software-cloud/docs/jql-operators/
- Search using the text field. https://support.atlassian.com/jira-software-cloud/docs/search-for-issues-using-the-text-field/
- JQL fields. https://support.atlassian.com/jira-software-cloud/docs/jql-fields/
- Confluence Cloud REST API v1, Search. https://developer.atlassian.com/cloud/confluence/rest/v1/api-group-search/
- CQL fields. https://developer.atlassian.com/cloud/confluence/cql-fields/
- Runeson, Alexandersson, Nyholm. Detection of Duplicate Defect Reports Using NLP. ICSE 2007. https://dblp.org/rec/conf/icse/RunesonAN07 (DOI 10.1109/ICSE.2007.32)
- Wang, Zhang, Xie, Anvik, Sun. An Approach to Detecting Duplicate Bug Reports using Natural Language and Execution Information. ICSE 2008. https://www.cs.utsa.edu/~xwang/papers/icse08.pdf
- Dang, Wu, Zhang, Zhang, Nobel. ReBucket: A Method for Clustering Duplicate Crash Reports Based on Call Stack Similarity. ICSE 2012. https://doi.org/10.1109/ICSE.2012.6227111

**Verification notes**
- Read in full or in relevant part (PDF or source): Luo 2014, Romano 2021, Memon 2017, DeFlaker, iDFlakies (abstract and results), FlakeFlagger (abstract and contributions), Fair, Strandberg 2020, CAM, NCChecker, Lampel 2021 (abstract), and all RF, SeleniumLibrary, AppiumLibrary, Appium and ChromeDriver sources cited.
- Verified from abstract or landing page only: Hashemi 2022, Haben 2023, Alshammari 2024, Habchi 2021, Herzig 2015, Amar & Rigby 2019, Golagha 2019, Kowalczyk 2020, Linnaeus, Sarika 2023, FlaXifyer, LogSage, Westermo 2026, Xiong 2024, Runeson 2007, Wang 2008, ReBucket 2012. The Runeson, Wang and ReBucket figures came from search-result excerpts of the abstracts, not the full PDFs.
- Not verified: exact Python-level connection error strings for your Selenium and urllib3 versions; Apple's entropy and flipRate formulas; the full category definitions in Sarika 2023.
