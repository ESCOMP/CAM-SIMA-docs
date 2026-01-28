# Testing

This page describes the various automated and manual tests that are run for CAM-SIMA whenever the code is modified, as well as instructions for how to add new tests.

## Regression Testing
### Running the regression tests (manual)

**NOTE:  Regression testing on derecho should be done for every PR before merging!**

Users can manually run regression tests on Derecho to ensure that the model builds correctly in various configurations.  The tests can be run with a local copy of CAM-SIMA by using the `test_driver.sh` script under `$CAM-SIMA/test/system`.  To run the tests associated with a particular compiler option one can do the following commands:

!!! Warning "run git-fleximod"
    Make sure you have run `bin/git-fleximod update` before you run the tests!

For running GNU tests*:
```
env CAM_FC=gnu ./test_driver.sh -f
```

For running Intel tests*:
```
env CAM_FC=intel ./test_driver.sh -f
```

*Note: you may also have to include the environment variable `CAM_ACCOUNT` on derecho, which is your account key

!!! Note "test_driver.sh default"
    By default, `test_driver.sh` will compare against the baselines in `/glade/campaign/cesm/community/amwg/sima_baselines/latest_<CAM_FC>`. If you wish to compare against different baselines, specify the path to those baselines with the environment variable `BL_TESTDIR`

!!! Note "Running test_driver.sh with no baseline comparison"
    If you do not wish to compare to baselines, don't use the `BL_TESTDIR` environment variable and use the flag `--no-bl-compare`


Running the script will produce a directory in your scratch space labeled `aux_sima_<CAM_FC>_<timestamp>`, where `<CAM_FC>` is the compiler you chose, and `<timestamp>` is the timestamp (starting with the date) of when the tests were started, along with a job submitted to the local batch system.

Inside the directory you should see an executable labeled `cs.status.*`.  Running that command after the submitted job has finished will display the test results. **Everything should be labeled `PASS`.  Any other label indicates that a test may have failed, and should be investigated.** Expected failures can be found in the `$CAM-SIMA/test/existing-test-failures.txt` file.

#### Inspecting test output
If you have an unexpected FAIL or PEND or DIFF, you will want to investigate further. Start by navigating into the test directory, for example:
```
cd SMS_Ln9.ne5pg3_ne5pg3_mg37.FTJ16.derecho_intel.cam-outfrq_se_cslam.GC.aux_sima_intel_20241223100339
```

If you have familiarity with CAM cases, the setup and directory structure will be familiar.

- A good place to start is the `TestStatus.log` file, which will include almost all test output (including build failures and information on where to look for other failures)
- Build logs for each component are in the `bld` directory. If you look at the filenames and one or more doesn't have the `.gz` end tag, those are likely culprits for having the error.
- Run logs are in the `run` directory. Similar to the build logs, log files that aren't zipped up are candidates for containing errors. The `atm.log.*` and `cesm.log.*` files are likely going to contain the information about the error you're seeking.
- Refer to the procedures for [checking metadata](../conversion/check-metadata.md) and [running CAM-SIMA](../conversion/run-cam-sima.md) for additional debugging help

#### Additional test info
The tests themselves are listed in `<CAM-SIMA>/cime_config/testdefs/testlist_cam.xml`. Any files that need to be included in order for the tests to run properly are located in `<CAM-SIMA/cime_config/testdefs/testmods_dirs/cam/outfrq_XXX`, where `XXX` is the name of the test.  Additional information on the CIME testing system, which is what this testing infrastructure is built on, can be found [online here](https://esmci.github.io/cime/versions/master/html/users_guide/testing.html).

### Archiving baselines
**If your PR changes answers**, then after you have run the tests, merged your PR, and created a tag (see [tag workflow](git-basics.md#tagging-a-commit)), you will need to archive your baselines for the next person.

To do this, navigate to `$CAM-SIMA/test/system` on derecho and run:

```
env CESM_TESTDIR=/glade/derecho/scratch/YourName/aux_sima_yyyymmddsssss CAM_FC=INTEL ./archive_baseline.sh <sima tag>
```
```
env CESM_TESTDIR=/glade/derecho/scratch/YourName/aux_sima_yyyymmddsssss CAM_FC=GNU ./archive_baseline.sh <sima tag>
```

!!! Note "Baseline 'latest' symlink"
    By default, `archive_baseline.sh` will create a symlink to the relevant `latest_<CAM_FC>` directory. If you are archiving old baselines and do not want to create the symlink, use the `--no-symlink` flag.

### Adding a new regression test

The test list can be found here: `$CAM-SIMA/cime_config/testdefs/testlist_cam.xml`

- If you are adding a new machine, compiler or category for an existing test, add a new `<machine>` XML entry to that `<test>` entry
- If you are adding a fully new test, add a *new* `<test>` XML entry with the following structure:
```
<test compset="<COMPSET_NAME>" grid="<GRID_ALIAS>" name="<TEST_TYPE>_<TEST_MOD>" testmods="<RELPATH_TO_TESTMODS_DIR>">
  <machines>
    <machine name="<MACH_NAME>" compiler="<COMPILER>" category="<TEST_CATEGORY>"/>
  </machines>
  <options>
    <option name="comment">COMMENT HERE</option>
    <option name="wallclock">WALLCLOCK_TIME</option>
  </options>
</test>
```

- `<COMPSET_NAME>`: component set alias (or long name) - you can see more about compsets [here](../usage/creating-a-case.md)
- `<GRID_ALIAS>`: model grid/resolution you'd like to run on - you can see more about grids [here](../usage/creating-a-case.md)
    - **Try to use the lowest/coarsest resolution that will still accomplish your testing goals**
- `<TEST_TYPE>`: type of test to be run. You can find the testing types [here](https://esmci.github.io/cime/versions/master/html/users_guide/testing.html#testtype).
- `<TEST_MOD>`: test modifier that changes the default behavior of the test type. More [here](https://esmci.github.io/cime/versions/master/html/users_guide/testing.html#modifiers)
    - **Unless a longer run is necessary to exercise the code you are testing, run for 9 timesteps (_Ln9)**
- `<RELPATH_TO_TESTMODS_DIR>`: relative path to the testmods directory for this run; usually looks something like `"cam/some_directory_name/"`
    - The testmods directory will contain any namelist mods and XML configuration variable changes for the test (`user_nl_cam` and/or `shell_commands`)
    - testmods directories can be found in `$CAM-SIMA/cime_config/testdefs/testmods_dirs/cam/`
!!! Note "standard snapshot test mods"
    All new test mod directories will likely include a mod to CAM_CONFIG_OPTS in `shell_commands` as well as the following mods to `user_nl_cam`:

      - `ncdata` (initial data file - the "before" snapshot)
      - `ncdata_check` (check file - the "after" snapshot)
      - `ncdata_check_err=.true.` (will cause the test to fail if the ncdata check process fails)
      - `min_difference=<some small number>` (if your ncdata_check had roundoff errors, set this to a number slightly above your roundoff)
      - `pver=30` (to match your snapshot file)
      - Some relevant history fields for baseline comparisons - see [history usage](../usage/history.md)
- `<MACH_NAME>`: machine name (options: `derecho`, `izumi`, `casper`)
- `<COMPILER>`: compiler to be used (options: `gnu`, `nag`, `intel`, `nvhpc`)
- `<TEST_CATEGORY>`: group of tests that this test belongs to - the default run by `test_driver.sh` is `aux_sima` (which is run for each PR to CAM-SIMA)
- `WALLCLOCK_TIME`: maximum amount of time that the job will be allowed to run

Here is an example test entry for a 2-timestep smoke test of kessler physics on a coarse MPAS grid, run with both intel and gnu
```
  <test compset="FKESSLER" grid="mpasa480_mpasa480" name="SMS_Ln2" testmods="cam/outfrq_kessler_mpas_derecho_nooutput/">
    <machines>
      <machine name="derecho" compiler="intel" category="aux_sima"/>
      <machine name="derecho" compiler="gnu" category="aux_sima"/>
    </machines>
    <options>
      <option name="wallclock">00:10:00</option>
      <option name="comment">GNU build test for MPAS dycore (with Kessler physics)</option>
    </options>
  </test>
```

## Github continuous integration testing
The following tests/linters are run automatically on pull requests to `develop` in github. You can see previous runs of the CI tests [here](https://github.com/ESCOMP/CAM-SIMA/actions)

### Python unit testing
CAM-SIMA supports two kinds of python unit tests, `doctest` and `unittest` tests, both of which are part of the standard python library.

All `unittest` tests should  be in:

`CAM-SIMA/test/unit/python`

while all files used by the tests should be in:

`CAM-SIMA/test/unit/python/sample_files`

All `unittest` tests are automatically run via Github Actions whenever a Pull Request (PR) is opened, modified, or merged.

All `doctest` tests are also run automatically as long as the scripts they are located in are under `CAM-SIMA/cime_config` or `CAM-SIMA/src/data`.

To manually run all of the unit tests at any time, simply run the following shell script:

`CAM-SIMA/test/run_python_unit_tests.sh`

Finally, when adding new tests, determine if the test can be done in only a few lines with minimal interaction with external files or variables.  If so, then it would likely be best as a `doctest`.  Otherwise it should be a `unittest` test.  Failure to follow this rule of thumb could result in test failures in the Github Actions workflow.  Also remember to add your new tests to the `run_python_unit_tests.sh` script so future users can easily run the tests manually.

#### Updating sample files
If you modified any python files in your code modifications (those in `cime_config` or in `src/data`), you may need to update the sample files to match what is now expected.

1. Run the unit tests as described above
1. If you have failures, quickly check that the diffs are expected

    ```
    diff <tmp file> <sample file>
    ```

1. Copy over the changed files from the `tmp` directory to the `sample_files` directory
1. Rerun the tests to confirm they all pass now.

### Fortran unit testing
CAM-SIMA supports Fortran unit testing via the pFUnit framework, which can be [found here](https://github.com/Goddard-Fortran-Ecosystem/pFUnit).  All of the actual tests are written in `*.pf` files, and can be found in the following locations in CAM-SIMA:

`test/unit/fortran/src/core_utils` (tests for core string utilities)
`test/unit/fortran/src/pio-reader` (tests for PIO NetCDF reader object)
`dynamics/mpas/tests/unit` (tests for procedures used in the MPAS <-> SIMA coupling layer)

There is no official method yet for running the unit tests locally, but you can find the general steps for running the tests in their respective Github Action workflow files:

[core string utility and PIO reader tests](https://github.com/ESCOMP/CAM-SIMA/blob/development/.github/workflows/fortran_unit_tests.yml)

[MPAS/SIMA coupling layer procedure tests](https://github.com/ESCOMP/CAM-SIMA/blob/development/.github/workflows/mpas_dynamical_core_ci.yml#L235)

If you would like to add your own tests to the automatic testing workflow then please reach out to a CAM-SIMA SE so we can help ensure that it is included in the official CMake testing build.

### Static Source Code Analysis

#### Python

Any python script which is added or modified via a PR will automatically be analyzed using `pylint`, and must have a score of 9.5 or greater in order to not be marked as a test failure.  The hidden `pylintrc` file used for the analysis can be found here:

`CAM-SIMA/test/.pylintrc`

Users can also manually run `pylint` against the core python build scripts by running the following shell script:

`CAM-SIMA/test/pylint_test.sh`

Please note that `pylint` is not part of the standard python library, and so it may need to be installed before being able to run the shell script.

### git-fleximod tests
[git-fleximod](git-fleximod.md) CI tests are run on both the oldest supported (currently 3.8) and latest versions of python to confirm:

- `bin/git-fleximod update` works on the existing `.gitmodules` file
- All module `url`s and `fxDONOTUSEurl`s match (e.g. a temporary fork was not committed)
- All module `fxtag`s exist and are in sync with submodule hashes
    - Also confirms that no `fxtag` is a branch
- Spare checkout files exist

If a test fails:

- View the run on github (either on the PR itself or in the [actions](https://github.com/ESCOMP/CAM-SIMA/actions) tab)
- View the output under `Run $GITHUB_WORKSPACE/bin/git-fleximod update`
    - Errors will be reported in course of the test execution
        - Compare with the last time the test succeeded to see what has changed
    - Errors may also be reported at the end of the execution
- Update the `.gitmodules` file to fix any errors and push up your changes to the branch, at which time the tests will be rerun

!!! Note "Run git-fleximod tests locally"
    You can (somewhat) reproduce the git-fleximod test locally by running the following

    ```
    bin/git-fleximod update &&
    bin/git-fleximod test
    ```

**Some example failures:**

URL does not match (no forks allowed!):
```
cice url https://github.com/peverwhee/CESM_CICE not in sync with required https://github.com/ESCOMP/CESM_CICE
```

Tags out of sync (may need to commit the updated module directory):
```
hemco hemco-cesm2_0_hemco_3_9_0 7bd8358 is out of sync with .gitmodules hemco-cesm2_0_hemco3_9_0
```

Sparse checkout file missing (check the path):
```
mpas sparse checkout file .mpas_sparse_checkout not found
```
