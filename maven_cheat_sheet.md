[<< Go back](README.md)

# Maven test plugins

## Maven Surefire Plugin

Maven plugin used during the **test** phase of the build lefecycle to execute unit tests.  
It generates reports in 2 formats (_.xml and _.txt).  
It has only one goal: **surfire:test**.

## Maven Failsafe Plugin

Maven plugin to run integration tests.  
Failsafe because its sysnonym to surefire, and if it fails, it fails safely.  
The Maven lifecycle has four phases for running integration tests:

1. **pre-integration-test** for setting up the integration test environment.
2. **integration-test** for running the integration tests.
3. **post-integration-test** for tearing down the integration test environment.
4. **verify** for checking the results of the integration tests.

If you use the Surefire Plugin for running tests, then when you have a test failure, the build will stop at the integration-test phase and your integration test environment will not have been torn down correctly.

## Skip tests

-DskipTests: Skip running any tests  
-DskipITs: Skip running only integration tests  
-Dmaven.test.skip=true: Skip compiling tests
