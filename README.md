# gotestfmt: go test output for humans

Are you tired of scrolling through endless Golang test logs in GitHub Actions (or other CI systems)?

![An animation showcasing that gotestfmt transforms a text log into an interactive log with folding sections.](https://debugged.it/projects/gotestfmt/gotestfmt.svg)

Then this is the tool for you. Run it locally, or in any CI system with the following command line like this:

```bash
set -euo pipefail
go test -json -v ./... 2>&1 | tee /tmp/gotest.log | gotestfmt
```

Tadam, your tests will now show up in a beautifully formatted fashion. Plug it into your CI, and you're done.

**Note:** Please always save the original log. You will need it if you have to file a bug report for gotestfmt.

**⚠️ With version 2.0 gotestfmt switched to supporting only JSON output. Please add the `-json` flag to your `go test` command line!**

## Setting it up in your CI system

We have support for several CI systems, and you can also customize the output to match your system. Gotestfmt detects the CI system based on environment variables. If it can't detect the CI system it will try to create a generic colored test output. You can force the CI output with the `-ci github|gitlab|...` option.

- [GitHub Actions](#github-actions)
- [GitLab CI](#gitlab-ci)
- [CircleCI](#circleci)
- [Add your own](#add-your-own-ci)

### GitHub Actions

For GitHub Actions we provide [gotestfmt-action](https://github.com/haveyoudebuggedit/gotestfmt-action), making it easy to use. Here's how you can set it up:

```yaml
jobs:
  build:
    name: Test
    runs-on: ubuntu-latest
    steps:
      # Checkout your project with git
      - name: Checkout
        uses: actions/checkout@v2

      # Install Go on the VM running the action.
      - name: Set up Go
        uses: actions/setup-go@v2
        with:
          go-version: 1.16

      # Install gotestfmt on the VM running the action.
      - name: Set up gotestfmt
        uses: haveyoudebuggedit/gotestfmt-action@v2
        with:
          # Optional: pass GITHUB_TOKEN to avoid rate limiting.
          token: ${{ secrets.GITHUB_TOKEN }}

      # Run tests with nice formatting. Save the original log in /tmp/gotest.log
      - name: Run tests
        run: |
          set -euo pipefail
          go test -json -v ./... 2>&1 | tee /tmp/gotest.log | gotestfmt

      # Upload the original go test log as an artifact for later review.
      - name: Upload test log
        uses: actions/upload-artifact@v2
        if: always()
        with:
          name: test-log
          path: /tmp/gotest.log
          if-no-files-found: error
```

Gotestfmt provides specialized output for GitHub Actions based on the presence of the `GITHUB_WORKFLOW` environment variable. You can also set gotestfmt to run in GitHub Actions mode by providing the `-ci github` option.

In GitHub Actions mode gotestfmt will look for the rendering templates in the `.gotestfmt/github` and `.gotestfmt` folders, which can be [customized](#add-your-own-ci).

### GitLab CI

There are multiple ways to run gotestfmt in GitLab CI. You can simply download it from the [releases section](https://github.com/haveyoudebuggedit/gotestfmt/releases) and use it that way, but we would recommend creating a custom container image to run the tests as follows:

```Dockerfile
# Include gotestfmt as a base image for building
FROM ghcr.io/haveyoudebuggedit/gotestfmt:latest AS gotestfmt

# Use the golang base image
FROM golang
# Copy gotestfmt into the golang image
COPY --from gotestfmt /gotestfmt /usr/local/bin/
```

You can then run the tests within this image with the following command:   

```bash
go test -json -v ./... | /usr/local/bin/gotestfmt
```

To put it all together, you can use the following `.gitlab-ci.yaml`:

```yaml
docker-build:
  image: docker:latest
  stage: build
  services:
    - docker:dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - |
      docker build -t gotestfmt .
      docker run \
        -v $(pwd):/source \
        -v /tmp:/tmp |
        -e GITLAB_CI=${GITLAB_CI} \
        gotestfmt \
        /bin/sh -c "cd /source; go test -json -v ./... 2>&1 | tee /tmp/gotest.log | /usr/local/bin/gotestfmt"
  artifacts:
    paths:
      - /tmp/gotest.log
    expire_in: 1 week
  rules:
    - if: $CI_COMMIT_BRANCH
      exists:
        - Dockerfile
```

You can, of course, customize this to your liking. We also recommend mirroring the `gotestfmt` image to your local registry to avoid rate limiting errors. See the [GitLab blog](https://about.gitlab.com/blog/2020/10/30/mitigating-the-impact-of-docker-hub-pull-requests-limits/) on how to do this.

Gotestfmt detects running in GitLab CI based on the `GITLAB_CI` environment variable. You can also force gotestfmt to run in GitLab CI mode by passing the `-c gitlab` option.

### CircleCI

There is no special template for CircleCI since it doesn't support advanced features like log folding. You can set up Circle CI by using the `gotestfmt` container directly:

```yaml
version: 2
jobs:
  test:
    docker:
      - image: circleci/golang:1.16
    steps:
      - checkout
      - setup_remote_docker:
          version: 19.03.13
      - run:
          name: Run tests
          command: go test -json -v ./... 2>&1 | tee /tmp/gotest.log | docker run -i ghcr.io/haveyoudebuggedit/gotestfmt:latest
      - store_artifacts:
          path: /tmp/gotest.log
          destination: gotest.log
workflows:
  version: 2
  build-workflow:
    jobs:
      - test
```

### Add your own CI

You can, of course, customize the output to match your CI system. This can be done by creating a folder named `.gotestfmt` in your project and adding the files below. You can find the default templates in the [.gotestfmt](.gotestfmt) folder in this repository.

When running on a well-known CI system, such as GitHub Actions, gotestfmt will detect that and look in the specific subfolder. If you think a specific CI system should have a custom template, please send us a pull request to this repository.

#### downloads.tpl

This file contains the output fragment showing the package downloads in the Go template format. It has the following variables available:

| Variable | Type | Description |
|----------|------|-------------|
| `.Failed` | `bool` | Indicates an overall failure. |
| `.Packages` | `[]Package` | A list of packages that have been processed. |
| `.StartTime` | `*time.Time` | The time the first download line was seen. May be empty. |
| `.EndTime` | `*time.Time` | The time the last download line was seen. May be empty. |

The `Package` items have the following format:

| Variable | Type | Description |
|----------|------|-------------|
| `.Package` | `string` | Name of the package. (e.g. `github.com/haveyoudebuggedit/gotestfmt`) |
| `.Version` | `string` | Version of the package. (e.g. `v1.0.0`) |
| `.Failed` | `bool` | If the package download has failed. |
| `.Reason` | `string` | Text explaining the failure. |

#### package.tpl

This template is the output format for the results of a single package and the tests in it. If multiple packages are tested, this template is called multiple times in a row. It has the following fields:

| Variable | Type | Description |
|----------|------|-------------|
| `.Name`   | `string` | Name of the package under test.
| `.Result` | `string` | Result of all tests in this package. Can be `PASS`, `FAIL`, or `SKIP`. |
| `.Duration` | `time.Duration` | Duration of all test runs in this package. |
| `.Coverage` | `*float64` | If coverage data was provided, this indicates the code coverage percentage. |
| `.Output` | `string` | Additional output from failures. (e.g. syntax error indications) |
| `.TestCases` | `[]TestCase` | A list of test case results. |
| `.Reason` | `string` | Text explaining the failure. Empty in most cases. |
| `.StartTime` | `*time.Time` | A pointer to a time object when the package was first seen in the output. May be nil. |
| `.EndTime` | `*time.Time` | A pointer to the time object when the package was last seen in the output. May be nil. |

Test cases have the following format:

| Variable | Type | Description |
|----------|------|-------------|
| `.Name` | `string` | Name of the test case. May contain slashes (`/`) if subtests are run. |
| `.Result` | `string` | Result of the test. Can be `PASS`, `FAIL`, or `SKIP`. |
| `.Duration` | `time.Duration` | Duration of all test runs in this package. |
| `.Coverage` | `float64` | If coverage data was provided, this indicates the code coverage percentage. Contains a negative number if no coverage data is available. |
| `.Output` | `string` | Log output from the test. |
| `.StartTime` | `*time.Time` | A pointer to a time object when the test case was first seen in the output. May be nil. |
| `.EndTime` | `*time.Time` | A pointer to the time object when the test case was last seen in the output. May be nil. |

## Architecture

Gotestfmt takes the output from `go test`, parses it and reformats it with the templates located in the `.gotestfmt` directory, or baked into the application.

This application has 3 main pieces: the tokenizer, the parser, and the renderer. All of them run in separate goroutines and pipeline data using channels.

The **tokenizer** takes the raw output from `go test` and turns it into a stream of events that can be consumed.

The **parser** takes the tokens from the tokenizer and interprets them, constructing logical units for test cases, packages, and package downloads.

Finally, the **renderer** takes the two streams from the parser and renders them into human-readable text templates, which are then streamed out to the main application for writing.

## Building

If you wish to build `gotestfmt` for yourself you'll need at least Go 1.16. You can then build it by running `go build cmd/gotestfmt`.

## License

This project is licensed under the [Unlicense](LICENSE.md), you are free to do with it as you please. It has no external dependencies.