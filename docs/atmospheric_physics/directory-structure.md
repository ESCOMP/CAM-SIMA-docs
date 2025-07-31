# atmospheric_physics directory structure

This page lists out the directory structure for the [atmospheric_physics](https://github.com/ESCOMP/atmospheric_physics) repo, and what the general purpose of each directory and subdirectory is.

## Top-level directories

These directories represent code and tools that are contained within the atmospheric_physics repository.

### **doc/**

Contains files used to document the current state of the atmospheric_physics repository, such as a `ChangeLog` and the `NamesNotInDictionary.txt` file which lists all standard names that are currently not in the official [CCPP Standard Names repo](https://github.com/ESCOMP/CCPPStandardNames). More on the NamesNotInDictionary.txt file [here](development_workflow.md/#updating-namesnotindictionarytxt-file).

### **phys_utils/**

Contains portable helper functions and utilities for use in CCPP-ized physics schemes. A general requirement is to have all routines in this directory automatically unit tested, either in atmospheric_physics itself or in a host model like CAM-SIMA.

### **schemes/**

Contains subdirectories for all of the CCPP-ized physics schemes, with the subdirectories containing the core physics scheme source code,
the associated CCPP metadata files, namelist XML files, and any relevant dependency files outside of `phys_utils` and `to_be_ccppized`.

### **suites/**

Contains [CCPP Suite Definition Files (SDFs)](https://ccpp-techdoc.readthedocs.io/en/v6.0.0/ConstructingSuite.html?highlight=sdf#constructing-suites) for official scientific and production configurations of the CAM-SIMA host model.

### **test/**

Contains code and tools used to run tests on the CCPP-ized physics schemes.

**Subdirectories**:

- cmake   - Contains cmake files needed to configure and build  unit tests.
- docker  - Contains dockerfiles needed to configure, build, and run unit tests.
- include - Contains utility source code needed to build and run unit tests.
- musica  - Contains test code for the CCPP-ized [Multi-Scale Infrastructure for Chemistry Modeling (MUSICA)](https://github.com/NCAR/musica) components.
- test_schemes - Contains CCPP physics schemes that are only used for testing.
- test_suites  - Contains test SDFs for use in CAM-SIMA snapshot regression testing.
- unit-test    - Contains the source code for the automated Fortran unit testing with pFUnit
    - include - Contains "mock" source code needed to run unit tests without bringing in an entire host model
    - tests - Contains subdirectories which have the actual pFUnit unit test code and related CMake files for building.

### **to_be_ccppized/**

Contains source code files that are needed by certain CCPP physics schemes, but which are not
yet fully CCPP compliant (e.g. have code that is still host-model-specific).