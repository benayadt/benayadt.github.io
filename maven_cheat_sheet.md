[Go Back](README.md)

- Maven Surefire Plugin
  Maven plugin used during the _test_ phase of the build lefecycle to execute unit tests.
  It generates reports in 2 formats (_.xml and _.txt).
  It has only one goal: _surfire:test_

- Maven Failsafe Plugin
  Maven plugin to run integration tests.
  Failsafe because its sysnonym to surefire, and if it fails, it fails safely
  The Maven lifecycle has four phases for running integration tests:

  1. _pre-integration-test_ for setting up the integration test environment.
  2. _integration-test_ for running the integration tests.
  3. _post-integration-test_ for tearing down the integration test environment.
  4. _verify_ for checking the results of the integration tests.

  If you use the Surefire Plugin for running tests, then when you have a test failure, the build will stop at the integration-test phase and your integration test environment will not have been torn down correctly.

- Skip
  -DskipTests: Skip running any tests
  -DskipITs: Skip running only integration tests
  -Dmaven.test.skip=true: Skip compiling tests
