# Creating, configuring, and running a case
Because CAM-SIMA uses the [CIME](https://esmci.github.io/cime/versions/master/html/index.html) build system, creating and configuration a case is mostly the same as it was in CAM7 (and in CESM). But! We're going to assume you know *nothing*.

## 1. Run `create_newcase`
Before you create a case:

1. Choose the correct machine to run your case on (if this is a high-resolution or long run, you will likely want to choose derecho!)
1. Consult the [git basics](../development/git-basics.md) if you need a new clone
1. Make sure you have run `bin/git-fleximod update` to update and/or populate your externals
1. Navigate to `$CAM-SIMA/cime/scripts`

Assuming you have completed the above steps and have a nice checkout of a development branch of CAM-SIMA, let's go!

Here is the (basic) structure of the command for create_newcase:

```
./create_newcase --case <CASEDIR> --compset <COMPSET_NAME> --res <RESOLUTION> --compiler <COMPILER> --project <PROJECT KEY> --run-unsupported
```

To break that down piece by piece:

- `--case <CASEDIR>` (optional): The `case` argument specifies the full path to the directory you would like to be created, in which your case will reside.
    - pro tip: if you choose to put your case directory in your scratch directory, your `run` and `bld` directories will be automatically placed within the case directory
        - on derecho, this means: `--case /glade/derecho/scratch/<username>/<case_name>`
        - on izumi, this means:   `--case /scratch/cluster/<username>/<case_name>`
    - if you do not specify a case directory, it will default to your `$CAM-SIMA/cime/scripts` directory (and, eventually, your `run` and `bld` directories will default to a new directory in your scratch space)
- `--compset <COMPSET_NAME>` (required): the compset (or "component set") tells CIME which base-level configurations you want to run with (including model versions and input data sets)
    - The compset supplied to the `create_newcase` command can be either:
        - The long name of a compset OR
        - The alias (if it exists) for the compset
    - Compset options and aliases can be found in [here](https://github.com/ESCOMP/CAM-SIMA/blob/development/cime_config/config_compsets.xml) (`$CAM-SIMA/cime_config/config_compsets.xml`)
    - The structure of a compset long name is `<initialization time>_<atmosphere model>_<land model>_<sea-ice model>_<ocean model>_<river runoff model>_<land ice model>_<wave model>`
        - Additional context: for each of the seven model components of the compset configuration, there can be three different implementations:
            - **active** (prognostic, full): solves a complex set of equations to describe model's behavior (must solve all equations on a numerical grid)
            - **data**: version that sends/receives same variables to/from other models, but read from files rather than computed from equations (reduces feedback within a system)
            - **stub**: occupies the required place in the driver but does not send or receive data
- `--res <RESOLUTION>` (required): the resolution specifies both the dycore being run and the grid.
    - The structure of the resolution argument is: `<atmosphere>_<ocean>_<mask>`
    - The resolution determines the dycore:
        - A grid resolution that looks like "ne*" indicates that this is spectral element (SE) cube sphere grid (SE dycore)
        - A grid resolution that looks like "C*" indicates that this is the FV3 (finite volume cubed sphere) dycore [not yet implemented in CAM-SIMA]
        - A grid resolution that looks like "mpasa*" indicates that this is an MPAS (Model for Prediction Across Scales) grid (MPAS dycore) [work-in-progress in CAM-SIMA]
    - You can find the options for these model grid aliases in `$CAM-SIMA/ccs_config/modelgrid_aliases_nuopc.xml`
    - Most often, for testing CAM-SIMA, we use a coarse SE grid like `ne3pg3_ne3pg3_mg37` or `ne5_ne5_mg37`
- `--compiler <COMPILER>` (optional): the compiler you wish to use
    - Options:
        - on derecho:
            - intel (default)
            - gnu
            - nvhpc
        - on izumi:
            - intel (default) - NOTE: intel is not to be trusted on izumi (it's a known-buggy version)
            - gnu
            - nag - NOTE: until the nag version is updated beyond 7.0, CAM-SIMA won't work with nag
- `--project <PROJECT KEY>`: a project key to charge to
    - Only needed on derecho
    - You can see the project keys you can use [here](https://sam.ucar.edu) (click on your primary group)
        - You can also see how much of the allocation has been used/is left by navigating to Reports -> Project Search and searching for the key
- `--run-unsupported`: this flag is mostly in place to indicate to scientists that the configuration they're choosing to run is not scientifically supported.

Given all that, let's say you run the following command on izumi (assuming you're this mysterious person "courtneyp"):
```
./create_newcase --case /scratch/cluster/courtneyp/kessler-ne5-gnu-0722 --compset FKESSLER --res ne5_ne5_mg37 --compiler gnu --run-unsupported
```
What will happen is that a new case directory will be created here: `/scratch/cluster/courtneyp/kessler-ne5-gnu-0722` that will be configured to run the FKESSLER compset (long name: `2000_CAM%KESSLER_SLND_SICE_SOCN_SROF_SGLC_SWAV`) with the SE dycore and the ne5 grid.


## 2. Configure the case
Navigate into your shiny new case directory and run `./case.setup`

From here, there are A LOT of configuration options. We'll highlight a few here that you'll use often.

### XML configurations
Many configurable settings can be found within the env_*.xml files in your case directory. In order to protect again typos, it's not advised to edit those directly. Instead, you will run `./xmlchange` commands to alter those settings (and `./xmlquery` can give you the current setting). You can find additional information on the configurable xml variables [here](https://ncar.github.io/CESM-Tutorial/notebooks/modifications/xml.html). A few to highlight:

- `CAM_CONFIG_OPTS`: This is where we tell CAM-SIMA what physics scheme(s) we wish to run, as well as if we wish to run the `null` dycore.
    - FKESSLER will default to `--physics-suites kessler --analytic_ic`, which means we're running the `suite_kessler.xml` SDF with the SE dycore (with analytic initial conditions - no need to supply an ncdata initial conditions file)
    - If you instead want to run a test with the null dycore, you'll want to change this with `./xmlchange CAM_CONFIG_OPTS = "--physics-suites kessler --dyn none"`
    - A list of all possible configuration options can be found at the bottom of this page, or [here](#current-cam_config_opts-options-in-cam-sima)
- `CAM_LINKED_LIBS`: Hopefully, you won't have to change this often; however, if you are getting linking errors during your build, try turning off the linked libraries (`./xmlchange CAM_LINKED_LIBS=''`)
- `DOUT_S`: This is the flag to archive log files of successful runs. During development, we advise turning this off so your log files don't disappear on you (`./xmlchange DOUT_S=False`)
- `STOP_OPTION` & `STOP_N`: How long you are going to run the model. If `STOP_N` is 8 and `STOP_OPTION` is "ndays", you are setting the model up to run for 8 days.
    - the options for `STOP_OPTION` are: 'nsteps', 'nseconds', 'nminutes', 'nhours', 'ndays', 'nmonths', 'nyears'
    - `STOP_N` is an integer
    - NOTE: if you are running the ncdata_check with snapshot files, keep the number of timesteps you are running at or below the number of timesteps on the snapshot files
    - NOTE #2: These configurations can be updated without rebuilding the case
- `DEBUG`: A flag to turn on debug mode for the run (runs without compiler optimizations) - this is very useful during development! It defaults to "False", so you can turn it on with `./xmlchange DEBUG=True`
- `RUNDIR`: this is the path to the `run` directory for your case. The `bld` directory will exist one level up.

If you run `./xmlchange`, the output will tell you if you need to re-set up and/or do a clean build of your case.

### Namelist configurations
There are countless namelist configuration options. These can be modified by updating the `user_nl_cam` file in your case directory.
Namelist options can be updated without rebuilding the case.

- [How to configure the namelist for history/output](history.md)
- [Information on namelist reader generation](../design/cam-build-process.md/#cam-sima-source-and-namelist-generation-buildnml-workflow)
- You can see the full CAM6 list of namelist options [here](https://docs.cesm.ucar.edu/models/cesm2/settings/current/cam_nml.html), keeping in mind that most of those options have not yet been ported to CAM-SIMA

## 3. Build the case
Run `./case.build`

The console output will tell you the progress of the build. If you get to `MODEL BUILD HAS FINISHED SUCCESSFULLY`, hooray!

[Debugging tips](../development/debugging.md#build-errors)

## 4. Run the case
Run `./case.submit`

The job will be submitted to the system's queue. You can see the status of your job with the `qstat` command. Once it is finished, the log files will be in the `run` directory (unless it ran successfully AND your archiver is on)

[Debugging tips](../development/debugging.md#run-time-errors)

## Current CAM_CONFIG_OPTS options in CAM-SIMA

The following is a list of all the allowed options that can be included in `CAM_CONFIG_OPTS`, what each option does, whether or not they are required (and if not what the default value is), and what config variable it maps to in the `ConfigCAM` code.

- **physics-suites** (required)
    - usage example: `--physics-suites cam7`
    - purpose:  Specifies what CCPP physics suite(s) will be built and made available for the given case.  More than one suite can be specified using a semi-colon, e.g. `cam4;cam7`.
    - ConfigCAM variable:  `physics_suites`
- **dyn** (required)
    - usage example: `--dyn mpas`
    - purpose:  Specifies what dynamical core will be built and made available for the given case.
    - ConfigCAM variable: `dyn`
- **analytic-ic** (optional)
    - useage example: `--analytic-ic`
    - purpose: Specifies whether Analytic Initial Conditions (IC) will be used for the given case.
    - default value:  `False` (file-provided ICs will be used instead)
    - ConfigCAM variable: `analytic_ic`
- **dyn-kind** (optional)
    - useage example: `--dyn-kind REAL64`
    - purpose: Specifies the precision used by the dycore for real-type variables.  Currently the only two options are `REAL64` for double precision and `REAL32` for single precision.
    - default value:  `REAL64`
    - ConfigCAM variable: `dyn_kind`
- **phys-kind**
    - useage example: `--phys-kind REAL64`
    - purpose: Specifies the precision used by the physics for real-type variables.  Currently the only two options are `REAL64` for double precision and `REAL32` for single precision.
    - default value: `REAL64`
    - ConfigCAM variable: `phys_kind`