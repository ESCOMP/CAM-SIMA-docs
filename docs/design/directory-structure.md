# CAM-SIMA directory structure

This page lists out the directory structure for CAM-SIMA, and what the general purpose of each directory and subdirectory is.

## Overview
Linked directories represent externals submodules (except .lib/git-fleximod, which is not a submodule but instead code copied from the external repository)
<small><pre>
CAM-SIMA
|-- .github
|-- [.lib/git-fleximod](https://github.com/ESMCI/git-fleximod)
|-- bin
|-- [ccpp_framework](https://github.com/NCAR/ccpp-framework)
|-- [ccs_config](https://github.com/ESMCI/ccs_config_cesm)
|-- [cime](https://github.com/ESMCI/cime)
|-- cime_config
|-- components
|   |-- [cdeps](https://github.com/ESCOMP/CDEPS)
|   |-- [cice](https://github.com/ESCOMP/CESM_CICE)
|   |-- [cism](https://github.com/ESCOMP/CISM-wrapper)
|   |-- [clm](https://github.com/ESCOMP/CTSM)
|   |-- [cmeps](https://github.com/ESCOMP/CMEPS)
|   |-- [mizuRoute](https://github.com/ESCOMP/mizuRoute)
|   |-- [mosart](https://github.com/ESCOMP/MOSART)
|   |-- [rtm](https://github.com/ESCOMP/RTM)
|-- docker
|-- libraries
|   |-- [FMS](https://github.com/ESCOMP/FMS_interface)
|   |-- [parallelio](https://github.com/NCAR/ParallelIO)
|-- [share](https://github.com/ESCOMP/CESM_share)
|-- src
|   |-- control
|   |-- core_utils
|   |-- cpl
|   |   |-- nuopc
|   |-- data
|   |-- dynamics
|   |   |-- mpas
|   |   |   |-- assets
|   |   |   |-- driver
|   |   |   |-- [dycore](https://github.com/MPAS-Dev/MPAS-Model)
|   |   |-- none
|   |   |-- se
|   |   |   |-- dycore
|   |   |-- tests
|   |   |   |-- initial_conditions
|   |   |-- utils
|   |-- history
|   |-- physics
|   |   |-- [ncar_ccpp](https://github.com/ESCOMP/atmospheric_physics)
|   |   |-- utils
|   |-- utils
|-- test
|   |-- system
|   |-- unit
|   |   |-- sample_files
|-- tools
</pre></small>

## Top-level external directories

These directories contain external repository source code, libraries, or tools, and usually are only present after `bin/git-fleximod update` has been run.

Please note that any modifications to the source code in these directories should generally be made outside of CAM-SIMA and instead in the associated external repository listed below.

### **bin/**

Contains the `git-fleximod` tool for downloading needed source code and software from external repositories.  More information on `git-fleximod` can be found [here](https://github.com/ESMCI/git-fleximod).

### **ccpp_framework/**

Contains the core [CCPP-framework](https://github.com/NCAR/ccpp-framework) source code and tools.

### **ccs_config/**

Contains the configuration files used by CIME to properly configure and build a CAM-SIMA (or CESM) case.  The Github repository associated with this directory can be found [here](https://github.com/ESMCI/ccs_config_cesm).

### **cime/**

Contains the [Common Infrastructure for Modeling the Earth (CIME)](https://github.com/ESMCI/cime) tools and associated libraries.  Used by CAM-SIMA and CESM for the general configuration and building of a simulation (or "case"). See more on CIME usage in the [Creating, configuring, and running a case](../usage/creating-a-case.md) section.

### **components/**

Contains the following other CESM components:

- [Community Data Models for Earth Prediction Systems (CDEPS)](https://github.com/ESCOMP/CDEPS)
- [Community Ice CodE sea ice model (CICE)](https://github.com/ESCOMP/CESM_CICE)
- [Community Ice Sheet Model (CISM)](https://github.com/ESCOMP/CISM-wrapper)
- [Community Terrestrial Systems Model (CTSM/CLM)](https://github.com/ESCOMP/CTSM)
- [Community Mediator for Earth Prediction Systems (CMEPS)](https://github.com/ESCOMP/CMEPS)
- [Reach-based river routing model (mizuRoute)](https://github.com/ESCOMP/mizuRoute)
- [MOdel for Scale Adaptive River Transport (MOSART)](https://github.com/ESCOMP/MOSART)
- [River Transport Model (RTM)](https://github.com/ESCOMP/RTM)


### **libraries/**

Contains the following external libraries:

- [Flexible Modeling System (FMS)](https://github.com/ESCOMP/FMS_interface.git)
- [Parallel IO (PIO)](https://github.com/NCAR/ParallelIO)

### **share/**

Contains source code [shared across all CESM components](https://github.com/ESCOMP/CESM_share).

## Top-level CAM-SIMA directories

These directories represent code and tools that is solely contained within the CAM-SIMA repository.  If there is a sub-directory that is also an external submodule or repository it will be marked with the "(external)" label.

### **cime_config/**

Contains the SIMA-specific python and XML configuration routines used by CIME and the CCPP-framework to properly configure and build a CAM-SIMA simulation, including the CCPP-generated caps and namelist files.

**Subdirectories**:

- testdefs - Location of the CAM-SIMA regression test list and associated files used by CIME during regression (system) testing.
    - testmods_dirs/cam - Location of CAM-SIMA case configuration files used during CIME regression testing.

### **docker/**

Contains files needed to run certain CAM-SIMA configurations in a docker container.

### **src/**

Contains all of the SIMA-specific source code needed to run the model. Additional details can be found below in the [Source directories](#source-src-directories) section.

### **test/**

Contains everything needed to perform software testing and validation of CAM-SIMA source code.

**Subdirectories**:

- system/ - Includes scripts needed to run CIME (integration) regression tests.
- unit/   - Includes scripts and source code needed to run CAM-SIMA unit tests.
    - sample_files/ - Contains files that are used to validate the unit test results.

### **tools/**

Contains non-required scripts and source code that may be useful for CAM-SIMA development.

## **Source (src) directories**

This section lists all of the directories underneath the "src" top-level directory.

### **control/**

Contains all of the source code needed for general model configuration, organization, and workflow, i.e. all of the "control" systems.

### **cpl/**

Contains all of the source code needed for SIMA to interact with a coupler

**Subdirectories**:

- nuopc/ - Contains the source code needed to interact with the [NUOPC](https://earthsystemmodeling.org/nuopc/) coupler, which is brought in via the CMEPS external.

### **data/**

Contains all of the source code needed to manage internal model data.  This includes auto-generated registry and initial conditions files code, physical constants, and atmospheric composition and thermodynamic properties.

### **dynamics/**

Contains all of the dynamical core (dycore) source code.

**Subdirectories**:

- mpas/ - Contains all of the source code needed for SIMA to properly couple to the [Model for Prediction Across Scales (MPAS) dynamical core](https://github.com/MPAS-Dev/MPAS-Model).
    - assests/ - Contains helper scripts, makefiles, and other files that are needed or useful but don't have scientific model source code.
    - driver/ - Contains the MPAS <-> SIMA interface source code.
    - dycore/ (external) - External submodule that contains the MPAS dycore code.
- none/ - Contains the "null" dycore source code, which allows physics and chemistry routines to be forced with atmospheric data coming directly from input files as opposed to dycore calculations.
- se/ - Contains all of the source code needed for SIMA to properly couple to the [Spectral Element dynamical core](https://ncar.github.io/CAM/doc/build/html/cam5_scientific_guide/dynamics.html#spectral-element-dynamical-core).
    - dycore/ - Contains the internal SE dycore source code files.
- tests/ - Contains the source code needed to configure analytic initial conditions for dycores.
    - initial_conditions/ - Contains source code needed to configure specific analytic initial conditions formulations.
- utils/ - Contains utility code used by all dycores in SIMA.

### **physics/**

Contains all of the SIMA physics and chemistry source code.

**Subdirectories**:

- ncar_ccpp/ (external) - External submodule that contains all of the code in the [atmospheric_physics](https://github.com/ESCOMP/atmospheric_physics) repo.
    - The directory structure for atmospheric_physics can be found in the [atmospheric_physics directory structure](../atmospheric_physics/directory-structure.md) section.
- utils/ - Contains SIMA-specific utility routines for working with the CCPP-framework and CCPP-ized physics routines.

### **utils/**

Contains source code for generic SIMA utility routines that can be used throughout the model.
