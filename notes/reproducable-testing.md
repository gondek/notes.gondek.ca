# Reproducible Testing Environments

_Written: March 17, 2025._

This page contains some ways to increase the reproducibility of tests between different machines (e.g. dev machines vs build machines). 

**Summary/TLDR:** Block unintentional network calls to 3rd parties (such as feature flag services and observability agents) and standardize the environment variables that tests get executed with.

## Tests in Docker and CI

If my team has tests that will eventually run in a container-oriented build pipeline, I find it's not much additional effort to wrap Docker's execution of tests in a shell script. Then, we can add some extra configuration to increase reproducibility. As a result, developers invoking the script locally have a very high chance of matching the execution in the build pipeline, allowing developers to debug any issues quicker (as opposed to waiting for the build pipeline to run again or shelling into build machines). 

It also makes the build pipeline file (e.g. `.circleci/config.yml` for CircleCI) more concise as that pipeline's step only has to invoke the script (e.g. `command: ./scripts/run_tests_docker.sh` for CircleCI) instead of listing all the separate commands inline. As the build pipeline configuration files can get quite big, I find it's easier to reason about them when some functionality is split into separate shell scripts.

Here is an example script with some explanations:

```shell
#!/usr/bin/env bash
set -euo pipefail

# Ensure commands run relative to the repository root for convenience.
# In this example, the script file is one subdirectory below the root.
REPO_ROOT=$( cd "$(dirname "$0")/.." ; pwd -P )
cd "$REPO_ROOT"

# Unconditionally/always build the image. Assuming the Dockerfile is
# written in a way to take advantage of layering and caching, this 
# step will run quite fast on hosts which have previously built parts
# of the same image (be sure to enable it in your build environment). 
# The corresponding Dockerfile is shown later in this post.
IMAGE_TAG="my-project-name-tests:latest"
docker build --target tests --tag "$IMAGE_TAG" .

# Define a working directly in the container where we will mount files.
CONTAINER_WORKDIR="/opt/document-parser"

# This is the core test command we will execute, including configuration
# to get coverage information. In this example, we have a Python project
# using pytest.
TEST_COMMAND="\
  poetry run coverage run -m pytest . -v --color=yes \
  --junitxml=$CONTAINER_WORKDIR/test/test-results/junit/junit.xml \
  && poetry run coverage html -d $CONTAINER_WORKDIR/test/test-results/htmlcov
"

# Use docker to run the tests. Explanations of the flags/parameters are below.
docker run \
  --rm \ 
  --network none \
  --volume "$REPO_ROOT/test:$CONTAINER_WORKDIR/test" \
  --volume "$REPO_ROOT/src:$CONTAINER_WORKDIR/src" \
  "$IMAGE_TAG" \
  bash -c "$TEST_COMMAND"

# '--rm': Automatically delete the container+volumes after finishing to 
# avoid taking up space on the host from repeatedly running tests. This 
# is especially useful on local developer machines where the script
# may be run several times in quick succession.

# '--network none': Disable networking for the container. This is useful to 
# find unintentional requests to 3rd parties which tests should not
# make. 
# Example 1: Triggering a code path which uses a 3rd party service to return
# a result. The test should be modified to avoid making such calls
# so that it is deterministic. This could be done with a mock implementation
# of a service interface (if using dependency injection) or mocking the library 
# used to make network requests (e.g. pytest-httpx if using httpx as an HTTP
# client in Python).
# Example 2: Triggering a code path that relies on a feature flag service
# that makes a network request to find the state of the feature flag. The 
# test should ideally be rewritten to avoid calling components that use 
# feature flags, or use dependency-injected configuration, or mock the feature
# flag service.
# Example 3: Triggering a code path that records errors or metrics that make
# a network request to a 3rd party (e.g. Sentry or Datadog). The test
# environment should disable these 3rd party SDKs completely, making them
# no-ops, or mocked SDKs can be injected.

# '--volume ...': Mount the source code and test cases into the container.
# We mount them dynamically instead of adding them in a layer in the Dockerfile 
# for convenience. Since both the source code and tests change frequently when
# developing, this shaves a bit of time from the docker build command as
# we no longer need an additional layer/container to be built with that code.
# It also avoids polluting the docker runtime with orphaned image layers.
# There is a small risk that doing this may introduce some inconsistencies.
# (e.g. if there is a .gitignore'd configuration file under /src on a dev's 
# local machine that isn't present on the build machine, then it could affect
# the outcome of the tests). If this is a concern, the Dockerfile could be
# modified to include the source directly in the image.

# "$IMAGE_TAG": Use the image we just build in the previous step.

# 'bash -c "$TEST_COMMAND"': Execute the test command we specified earlier
# inside the container.

# Lack of '--env` or `--env-file`: By omitting the loading of any .env file
# or other environment variables, the container will only have any system
# default variables from the base image. If tested components rely on
# "app-level" env vars, they can be modified to accept those values as a 
# parameter, or the specific tests can monkeypatch the environment
# temporarily (and reset it after the tests finish). 
# I find this is one of the most common reasons tests vary between build 
# pipelines and local machines: without environment variable isolation, 
# code can run differently between a dev's machine and the build machines.
# For example, a developer may have configured loading of vars from a 
# '.env' file when testing the service locally, which they modify over time. 
# Then the build servers have been configured with some default variables 
# that might be used in end-to-end testing, but also accidentally leak into
# unit tests (e.g. 'contexts' in CircleCI automatically inject collections 
# of variables into the build machine hosts). 
# If a dev's '.env' file and the build machine config drift, tests 
# might run differently, even though we only intended those variables to 
# affect general usage of the service or end-to-end tests.
```

For reference, here is an example `Dockerfile` to complement the above script:

```dockerfile
## BASE stage
## ============================================================
FROM python:3.11-slim AS base
WORKDIR /opt/my-service-name/

RUN set -eux \
  # ... other commands (like apt-get) ommitted for brevity
  && pip install --upgrade pip \
  && pip install --upgrade poetry

COPY pyproject.toml poetry.lock poetry.toml /opt/my-service-name/

# Install only runtime dependencies. We exclude test dependencies as 
# not all other images that inherit from 'base' will need them.
RUN poetry install --only main

## tests stage/target (inherits from base)
## ============================================================
FROM base AS tests
# Need to install dev+test dependencies (as '--only main' was done in the 'base' stage)
RUN poetry install
USER document-parser
# Note: no copying of source or test code. That is done via volumes in the test script.

## deployment stage/target  (inherits from base)
## ============================================================
FROM base AS deployment
COPY ./src /opt/document-parser/src
# Note: no copying of test code, and the test/dev dependencies were omitted in the 'base' stage
```

Since the shell script always runs a `docker build` against the `tests` target, and also dynamically mounts the source code and tests, we can be quite confident about the correctness of the test execution. For example, if we upgrade our project dependencies (in this example, via changing `pyproject.toml` and its associated files), the docker layer would be invalidated, causing a rebuild of the later steps. If nothing changes, then the build would run instantly since all the layers would be the same.

## Tests on a Local Machine (e.g. via an IDE)

Running tests in Docker can hamper local development depending on the OS and language ecosystem. 

For example, getting PyCharm to run tests in Docker containers can be tough (including being able to select individual test cases by pressing the "Run" icon beside them in the editor gutter). It gets even trickier when attempting to attach the PyCharm debugger to a container (e.g. setting up `pydevd_pycharm` which I have found unreliable in the past).

For that reason, I set up test frameworks so they can also be run directly on the host in addition to being run in containers. Then they can be easily integrated with IDE features. However, this could affect the reproducibility of the tests, which the developer might not notice until the tests are run in Docker in the build pipeline.

To recapture some reproducibility of Docker-based test execution, I often add some root-level fixtures/hooks for some common causes of inconsistent test results.

**.env Files**

My teams have often used `.env` files to make it easier to manage configuration of our apps via environment variables. This is useful when running the app locally, but can easily change the branching of code in tests.

If we're using something like `pydantic_settings.BaseSettings` which supports autoloading `.env` files via the `env_file` option, one option is to disable that during test execution: 

```python
@pytest.fixture(autouse=True, scope="session")
def bypass_dotenv_file():
    """
    When running tests outside a container (on a developer's host), we want to prevent
    accidental use of environment variables via the dotenv file (.env). This way the developer 
    will have a consistent experience between their host and when running the same tests in 
    Docker/CircleCI.
    """
    MyAppSettingsClass.model_config["env_file"] = None  # disables .env file loading

    settings = MyAppSettingsClass()

    # Self-test / future-proofing: We assert that a specific value that all developers should 
    # have set in their .env file is blank. If this assertion fails, it means our way of 
    # disabling env file loading is not working and needs to be fixed so this fixture 
    # continues working properly.
    assert not settings.some_common_value
```

An additional step of calling `os.environ.clear()` could work in some cases, but is likely too aggressive (e.g. it could affect `pytest`'s output mode or how Python module loading works). Also using it in a session-scoped fixture could allow some tests to accidentally pollute it.

A function-scoped fixture that restores the env after each test finishes is gentler, but still has a small risk of affecting "system-level" config options:
```python
@pytest.fixture(autouse=True, scope="function")
def remove_env_vars():
    with mock.patch.dict('os.environ', clear=True):
        # environment variables are cleared inside this block
        yield # yield execution to the test case code
        # once the test case is done, control is transferred back
    # exiting the `with` block restores the environment
```

Then to test individual modules that rely on environment variables, they can be inverted to use dependency injection, or additional calls to `mock.patch.dict` or `monkeypatch.setenv` can be made in the test case setup code.

**Disabling Feature Flags**

Accidentally coupling tests to the state of feature flags can happen when the feature flag service is used directly in a component (instead of having the state dependency injected) and some older existing cases test the same component but haven't been updated to mock/override the newly added feature flags.

Ideally, tests could be run in a way that prevents the code from initialing the feature flag service in the first place. If that's not possible, I like to patch the feature flag SDK so that it fails loudly when a feature flag access has not been mocked. This can either be done by disabling the SDK's initialization logic, or monkey-patching it to fail all feature flag state requests.

For example, if using the global SDK-scoped LaunchDarkly client (`ldclient.get()`) and using `variation_detail` to fetch the flag state, the following fixture could work:
```python
@pytest.fixture(autouse=True, scope="session")
def disable_launchdarkly():
    """
    Forces all tests which trigger a feature flag lookup to mock it by
    causing an exception to be thrown by default.
    """
    import ldclient

    ld_client_mock = MagicMock()
    ld_client_mock.variation_detail.side_effect = Exception(
        "A test case is triggering a LaunchDarkly feature flag lookup via the SDK."
        "Please ensure the feature flag is mocked, or switch it to use a dependency injected state."
    )

    with mock.patch.object(ldclient, "get") as mock_get:
        mock_get.return_value = ld_client
        yield
```

**Disabling Observability Libraries and Agents**

Services like Sentry and NewRelic are great for observability. However, I disable them in tests since the network requests they issue and their hooking into 3rd party libraries can create noise. One downside of this is that if the SDKs are being used incorrectly or are buggy themselves, test cases won't catch that. If that's a concern, a separate integration test might be useful.

Ideally, tests could be run in a way that prevents the code from initialing these SDKs in the first place. If that's not possible, then I like to patch the SDKs.

At the time of writing, to disable the Sentry SDK in Python, it's enough to patch the `init` method:

```python
@pytest.fixture(autouse=True, scope="session")
def disable_sentry():
    """
    Because Sentry automatically hooks into other 3rd party libraries and can send
    network requests, it's best we prevent it from initializing because it causes noise 
    in the test output.
    """
    import sentry_sdk
    
    with mock.patch.object(sentry_sdk, "init"):
        yield
```
