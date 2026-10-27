# Single Bet Placement - QA Assignment

Test plan, bug reports and a small automation framework for the single bet feature on https://qae-assignment-tau.vercel.app.

The assignment asks for Python + Pytest by default. I used **Java + TestNG**, and the recruiter confirmed that's fine.

## What's in here

- [docs/01-test-plan.md](docs/01-test-plan.md) - 6 scenarios with priority and risk rationale, plus the unclear parts of the spec
- [docs/02-execution-and-bug-reports.md](docs/02-execution-and-bug-reports.md) - results of the top 3 scenarios, exploratory notes, 17 bugs
- [docs/03-strategy-and-recommendations.md](docs/03-strategy-and-recommendations.md) - why these tests, what stays manual, what to do next
- [docs/evidence/](docs/evidence/) - screenshots
- `src/` - the framework and the 2 tests

The most important bugs:
- The receipt payout is always stake x 2 (BUG-01).
- The API accepts negative stakes and adds money (BUG-02).
- The balance can go negative through the API (BUG-03).
- Clicking Place Bet fast several times creates several bets (BUG-04).

## Stack

- Java 17, Maven
- Selenium 4.51 - drivers are handled by Selenium Manager, so no WebDriverManager or manual chromedriver download
- TestNG 7.12
- RestAssured 5.5 with Jackson 2. I didn't use RestAssured 6 because it moves to Jackson 3, which has new package names
- Allure 2 for the report (TestNG's own HTML report is generated too)
- BigDecimal for anything with money, because doubles give wrong results (3 * 3.3 != 9.9)
- Maven Wrapper, so you don't need Maven installed

### Why Allure

TestNG's default report only tells you which test failed and shows the stack trace. For this project that isn't enough, because most failures are real app bugs and you need to see what happened without re-running anything. Allure gives me that out of the box:

- **Request/response for every API call.** `AllureRestAssured` attaches the full HTTP exchange, so for BUG-02 you can see the `-5` stake going in and the 200 with the higher balance coming back.
- **Screenshot and page source when a UI test fails**, attached by `BaseUiTest`.
- **Each check as a step.** `SoftChecks` logs every check, so you see the 12 that passed as well as the 4 that failed, not just the first assert.
- **Failure categories.** `categories.json` splits failures into "Known bugs", "New failures" and "Possibly fixed bugs". With known bugs in the app, the suite is red on purpose, and this is how you tell at a glance whether anything *new* broke.
- **Environment info** (URL, browser, headless, Java) at the top of the report, so it's clear where the run happened.

It also plugs into CI easily - the workflow builds one report from the API and UI jobs and uploads it as an artifact.

The trade-off is that you need `./mvnw allure:serve` (or `allure:report`) to view it instead of just opening a file, and it adds the AspectJ agent to the test run for `@Step`. I think that's worth it here. The plain TestNG report is still generated in `target/surefire-reports/` if you just want a quick pass/fail.

## Project layout

```
src/main/java/com/betting/qa
  config/TestConfig.java        settings: -D property, then env var, then config.properties
  api/BettingApiClient.java     RestAssured client, adds x-user-id, logs to Allure
  api/ApiEndpoints.java
  api/model/                    records for requests/responses
  domain/Selection.java         HOME/DRAW/AWAY and how they map to the UI
  data/TestData.java            picks the next upcoming match
  ui/driver/DriverFactory.java
  ui/pages/                     BasePage, MatchListPage, BetSlip, BetReceiptModal
  util/Money.java               parsing "€1,118.50", payout calculation
  util/SoftChecks.java          soft asserts that can be tagged with a bug id

src/test/java/com/betting/qa
  base/BaseApiTest.java         creates the API client
  base/BaseUiTest.java          browser per test, screenshot on failure
  api/StakeValidationApiTest.java
  ui/PlaceSingleBetE2ETest.java
  listeners/AllureEnvironmentListener.java

src/test/resources
  suites/all.xml, api.xml, ui.xml
  allure/categories.json
```

The framework code is in `src/main` and the tests are in `src/test`. Tests don't have locators, URLs or HTTP setup in them, so when the UI or API changes, the fix is in one place.

## Requirements

- JDK 17 or newer. If you have several on a Mac: `export JAVA_HOME=$(/usr/libexec/java_home -v 17)`
- Chrome
- A user id for the test app
- Internet access (first run downloads Maven, the dependencies and chromedriver)

Maven doesn't need to be installed - the project has the Maven Wrapper (`mvnw` / `mvnw.cmd`), which downloads Maven 3.9.11 on first use. If you already have Maven 3.9+ you can use `mvn` instead of `./mvnw` in all commands below.

## Running

```bash
export USER_ID=<your-user-id>     # or pass -DuserId=<your-user-id> on each run

./mvnw clean test                 # API tests, then the UI test
./mvnw allure:serve               # open the report
```

On Windows use `mvnw.cmd` and `set USER_ID=<your-user-id>`.

If you got the project as a zip and `./mvnw` says "permission denied", run `chmod +x mvnw` once.

Other options:

```bash
./mvnw clean test -Dsuite=api                    # only API, no browser
./mvnw clean test -Dsuite=ui -Dheadless=false    # watch the browser
./mvnw clean test -DbaseUrl=https://other-env    # different environment
```

To build a static report even when tests fail:

```bash
./mvnw clean test -Dmaven.test.failure.ignore=true
./mvnw allure:report     # -> target/site/allure-maven-plugin/index.html
```

TestNG's report is in `target/surefire-reports/index.html`.

### Settings

You can set each one with `-D`, an env var, or in `src/main/resources/config.properties` (checked in that order).

| Key | Env var | Default |
|---|---|---|
| userId | USER_ID | none, required |
| baseUrl | BASE_URL | https://qae-assignment-tau.vercel.app |
| browser | BROWSER | chrome |
| headless | HEADLESS | true |
| uiTimeoutSeconds | UI_TIMEOUT_SECONDS | 10 |
| apiTimeoutMillis | API_TIMEOUT_MILLIS | 15000 |

## The two tests

**PlaceSingleBetE2ETest** selects odds on the next upcoming match, enters €10, places the bet, and checks the slip, the receipt, the saved balance (via the API) and the header balance. 16 checks.

**StakeValidationApiTest** is one data-driven test with 10 stake cases: min, max, below/above, zero, negative, 3 decimals, string, null, and more than the balance. Each case also checks the balance didn't change (rejected) or went down by exactly the stake (accepted).

Both classes have a comment at the top explaining why I picked them.

### Current result: 8 pass, 3 fail

The tests check the spec, so they fail where the app has bugs:

```
StakeValidationApiTest [Negative -5.00]            BUG-02
StakeValidationApiTest [50.00 with only 20 left]   BUG-03
PlaceSingleBetE2ETest                              BUG-01, BUG-05, BUG-06, BUG-15
```

I didn't want to make them pass by removing assertions. `SoftChecks` lets a check be tagged with a bug id, and the failure message starts with one of these:

- `[KNOWN DEFECT]` - only tagged checks failed, nothing new
- `[UNEXPECTED FAILURE]` - an untagged check failed, look at this first
- `[POSSIBLY FIXED]` - a tagged check passed, the bug might be fixed

`allure/categories.json` uses those prefixes to group the failures in the report.

## A few decisions

- The UI test uses the API to reset the balance and check the saved balance afterwards. The browser is only for the actual clicks. That's faster and less flaky, and it's how BUG-06 was found (the header showed a different balance than the API).
- No hardcoded match ids. Lots of the list is in the past, and bets on those are still accepted (BUG-08).
- No `Thread.sleep`, only explicit waits.
- Tests run one at a time because they share one user and the API rejects parallel bets from the same user with 409.
- The user id isn't in the repo. Allure attaches request headers to the report, so don't share `target/` publicly (it's gitignored).

## CI

`.github/workflows/tests.yml` runs the API tests, then the UI test in headless Chrome, and builds one Allure report. Add a repo secret `USER_ID` first.

## Problems I ran into

- **"This version of ChromeDriver only supports Chrome version ..."** - an old chromedriver on the PATH (I had one from Homebrew). The pom sets `SE_SKIP_DRIVER_IN_PATH=true` so Selenium Manager ignores it. If you run from the IDE, add that env var to the run config.
- **"release version 17 not supported"** - Maven is using an older JDK. Set JAVA_HOME to a JDK 17+.
- **Lots of 409 errors** - something else is using the same user id at the same time (another tab or another run).
