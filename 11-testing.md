# Part 11: Tests



## Testing purposes

* Discover as many problems as possible as early as possible
* Make it less scary to change the code
* Tests run on pull requests on the build machines
    * Compile checks and basic windows test always run
    * Extensive platform tests run on request



## Test types

* Unit tests
    * Test an isolated piece of functionality
    * Asserts
    * Run really fast (seconds for entire codebase)
* Integration tests
    * Test that everything works together
    * Anything that needs to load project data from disk, etc
    * Runs pretty slow (minutes per platform)
* Unit tests are really awesome
* Nobody can be bothered to write integration tests
    * Because the system is bad?
    * Because it takes forever and is a PITA to run them?
    * Because it is just a lot of work?



## Unit test system

* Tests are just functions called with an `ITestRunner` class
    * Switches for whether tests should run
    * `test_runner.h`
* Hierarchy of test functions
    * `test_foundation_collection.cpp`
* Compiled as a part of the regular engine
    * Run with `--run-unit-tests --test-for-hours --test-disk --test-network`
    * Results reported at end of run



### Regression test system

* Ruby script: `run_regression_tests.rb`
    * Sync a project from git
    * Compile and run it on a specific platform
    * Send Lua commands using the console connection
    * Check the results of those Lua commands
* Tests in ruby files `character.rb`
* No automatic image comparison/rendering tests
    * Some work is being done
