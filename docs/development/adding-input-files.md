# Adding input files

Instructions for how to add a new file to the CESM inputdata repository can be found here.  Doing so will allow the file to be accessible by all users of CAM-SIMA and/or CESM.

**Please note that these instructions can only be followed if one has access to the NCAR file systems (i.e. access to the NCAR clusters)**

## 1. Add required attributes to data file.

All input files should have metadata describing relevant data and file information.  With NetCDF files this metadata should be added as global file attributes. The list of required and suggested metadata fields can be found online here:

[CESM required dataset metadata](https://www.cesm.ucar.edu/models/cam/metadata)

## 2. Set the correct file permissions and add datestamp to file name

Make sure the file is globally readable, but only user writeable.  This can be done on the command line like so:

`chmod 644 <file>`

Where `<file>` is the name (or path) of the file.

Also, it is generally recommended to add a datestamp to the filename indicating when the file was created.  This is done by adding a `_cYYMMDD` suffix to the filename.  For example, if a file named `input.nc` was created on July 28, 2023, the filename should be updated to `input_c230728.nc`.

## 3. Move file to inputdata directory

Once permissions and metadata have been properly set, move the file to the respective location under `/glade/campaign/cesm/cesmdata/inputdata`. In general you'll want to move the file to the same subdirectory as previous versions of that dataset.  Of course if you are unsure where the new input data file should go then please reach out to an NCAR scientist or SE and they will help determine where the file should go.

## 4. Import the file to the Geoscience Data Exchange (GDEX)

Finally, you'll want to add the file to GDEX (which ensures the file is backed-up and accesible to the wider community) by using the `rimport` command located in `/glade/campaign/cesm/cesmdata`.  To add a file to GDEX, simply run

`rimport -file <file>`

Where `<file>` is the file path/name.

Once that command is done then congrats!  You have successfully added a file to the CESM inputdata repository.