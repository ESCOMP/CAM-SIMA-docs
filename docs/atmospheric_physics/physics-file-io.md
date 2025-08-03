# Physics scheme file I/O

This page describes the `ccpp_io_reader` class in `atmospheric_physics/utils`, including it's purpose, design, and usage in CCPP physics schemes.

## Why a special file reader class?

Many physics schemes have a need to read specialized data from input files, especially those in the NetCDF file format.  While one could just read all of the files on the host side before calling the CCPP, this would require a physics developer to modify the host model whenever they wanted to use the physics scheme, which arguably hurts portability.

Alternatively, one could just use the NetCDF library directly in the physics scheme, but this oftentimes adds a lot of additional lines of code to the physics scheme, and may prevent the physics scheme from taking advantage of specialized I/O libraries contained within the host model that may improve performance, error handling, etc.

To try and find an optimal solution to this problem, it was decided to create an abstract class that describes general interfaces for reading variables from a file, which allows a physics scheme developer to read data from a file without having to modify the host model or use the NetCDF library directly.  At the same time, the actual implementation of the class can be done in the host model, which then allows for the use of host-specific I/O libraries, while still maintaining the portability of the physics scheme itself.

## Design and implementation of `ccpp_io_reader` in CAM-SIMA

The file `ccpp_io_reader.F90` file itself contains the declaration of an abstract NetCDF reader type (`abstract_netcdf_reader_t`), including its various procedures, and an associated constructor function (`create_netcdf_reader_t`).  This abstract interface is assumed to be the same for all host models and physics schemes, which allows CCPP physics schemes to remain portable, at least amongst the supporting host models.

The actual implementation of this abstract class is expected to be done by the host model using whatever methodology it wants, as long as it doesn't violate the rules of the CCPP itself (e.g. it is thread safe).  The specific implementation for CAM-SIMA can be [found here](https://github.com/ESCOMP/CAM-SIMA/blob/development/src/physics/utils/pio_reader.F90).  It uses the [ParalellIO (PIO) library](https://github.com/NCAR/ParallelIO), which is used in NCAR's [CESM](https://github.com/ESCOMP/CESM) model (the parent model of CAM-SIMA) to read and write NetCDF files.

There are several ways one could make the connection between the abstract class and the actual, concrete implementation in Fortran.  For the abstract NetCDF reader defined in `ccpp_io_reader.F90`, it was decided to use the Fortran submodule approach, which was a feature introduced in the Fortran 2008 standard.  This approach allows a physics scheme to bring in the file reading module via a Fortran `use` statement, and then declare, construct, and work with reader object in an entirely local way within the scheme.  An alternative method would be to have the host model instantiate the object on the host side before any CCPP physics steps are called, but this would require the object to be permanently held in memory, passed around as a subroutine argument to all of the physics schemes that use it (which would require additional metadata), and provide a way to open multiple files within the same object.

An example of the submodule approach connecting the abstract interface with a "concrete" implementation can be found in [CAM-SIMA here](https://github.com/ESCOMP/CAM-SIMA/blob/development/src/physics/utils/pio_reader_sub.F90).

## File reader class methods and code example

The following methods are provided either for or by the NetCDF reader class (`abstract_netcdf_reader_t`).  Additional methods, or changes to the existing methods, may be added as new functionality is introduced.  In the list below, the variable `reader` represents a variable declared as `class(abstract_netcdf_reader_t), pointer` in the physics scheme.

### *reader => create_netcdf_reader_t()*

The constructor function for the `abstract_netcdf_reader_t` class.  This must be used to actually allocate/associate the object.  It has no input arguments.

### *reader%open_file(file_path, errmsg, errcode)*

Opens the NetCDF file and stores the associated (meta)data within the object.  The only input is `file_path`, which is a string containing the path to the file, while `errmsg` and `errcode` are standard CCPP outputs, with `errcode` being a non-zero integer, and `errmsg` being a non-empty string, if an error occurs.

Please note that all of the remaining methods below also have `errcode` and `errmsg`, which behave in the exact same way.

### *reader%close_file(errmsg, errcode)*

Closes the NetCDF file that was opened by an `open_file` call.  It has no inputs.

### *reader%get_var(varname, var, errmsg, errcode, start, count)*

Reads in a variable from the NetCDF file that was opened with `open_file`.  The method has two required inputs:

1.  `varname` -> A character-type variable which contains the name of the variable on the NetCDF file.
2.  `var`     -> An assumed-shape allocatable variable of type integer, real, or character with the same rank (i.e. shape) as the variable on the file.  This variable will automatically be allocated to the same dimensions as described for the variable in the NetCDF file, and will contain the actual variable data.

It also has two optional inputs:

1.  `start` - a 1-D integer array with the same number of elements as there are dimensions for the variable that indicate the starting indices when subsetting the data.  CURRENTLY NOT USED.
2.  `count` - a 1-D integer array with the same number of elements as there are dimensions for the variable that indicate the number of elements, starting from `start` that will be read in when subsetting the data.  CURRENTLY NOT USED.


Finally, an actual example of using the NetCDF reader object in a CCPP physics scheme can be [found here](https://github.com/ESCOMP/atmospheric_physics/blob/development/test/test_schemes/file_io_test.F90).
