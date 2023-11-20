# Noah-MP version With Mosaic and HUE

This github branch contains the new mosaic scheme and Heterogenous Urban Environments (HUE) in the older version of Noah-MP. 

File Directories are as follows:  
```
    .
    ├── drivers
    │   └── wrf
    │       └── module_sf_noahmpdrv.F
    ├── noahmp-HUE-specific-files
    │   ├── module_hrldas_netcdf_io.F
    │   ├── module_NoahMP_hrldas_driver.F
    │   └── MPTABLE.noahmp-mosaic-hue-NLCD40.TBL
    ├── parameters
    │   ├── GENPARM.TBL
    │   ├── MPTABLE.TBL
    │   └── SOILPARM.TBL
    ├── README.md
    └── src
        ├── module_sf_noahmp_glacier.F
        ├── module_sf_noahmp_groundwater.F
        └── module_sf_noahmplsm.F
```
## module_sf_noahmpdrv.F 

The driver contains most of the changes for the mosaic scheme, found starting at line **3780**. The first new sub-routine assignes the new mosaic variables based on what was already assigned in the normal noah-MP init subroutine. 

There is also a modified version of the 'bubble up' search routine, that will either find the largest probable land-cover distribution, or will find the largest probable land-cover distribution with the certain land-covers placed adajcent to one other (which is the key to how Noah-MP HUE works). There was an issue with the WRF driver and teh zsnsoxy_mosaic variable, which was not reading in restart snow files. SO to address we add an extra assignment that will always happen (even if ZSNSOXY_mosaic is provided in restart files).

The mosaic module is ran starting on line 4806. this contains the calls of the Noah-MP LSM with an extra for loop in order to loop over land-covers. This area also has the URBAN LSM call. BEP and BEP+BEM are not currently supported, though should be easy to integrate. 

The general way the mosaic module works is :
+ extract the previous time steps information from the mosaic values 
+ pass the mosaic values to the now 'dummy' variables that are 2 - dimension (for examplpe TSK(I,J))
+ Pass the 2-dimension variable directly to the single, non-dimensional value that will be ran in the column model
+ replace the variables that come out of the column model. 
+ once every mosaic land-cover is ran per grid, average based on land-cover type and save in the normal 2-d outputs. 
    + these will be taken by the other WRF modules to influence their physics. 
+ Iterate to a new grid cell (increment I, J) in the tile

Land-cover averaging is a bit nuanced, as some land-covers do not have vegetation. This means that there are some times that we have to 'inflate' some vegetation specific values. For example: Vegetation temperature (TV) only is defined if there is vegetation on the grid. If there is a UCM or if there is a barren land-cover type, no vegetation is defined. In the mosiac averageing, we remove that value from the land-cover average to ensure that we have realistic values for the grid average vegetation temperature. 


## module_sf_noahmplsm.F

This is the changed around noah-mp column model for the LSM. The big difference from normal Noah-MP and this module is the change of a copy soil moisture value, and input/output values for extraction and scaling of soil moisture and runon terms. There is also a new green roof module that is not currently in use, but will be added in the next few months. 

Extra soil moisture is to ensure that we are always evaluating the potential ET correctly (e.g. the soil stress is accounted for correctly). The new values of soil moisture and runon are always evaluated for the land-type we are currently in, and will be appropriatly added to the next land-cover value if HUE is engaged (iopt_HUE == 1). 

## MPTABLE.noahmp-mosaic-hue-NLCD40.TBL

This is the MPTABLE that contains the constants for noah-MP mosaic, HUE, which exclusively uses the NLCD40 land-cover dataset. note that this does not have the Urban climate zones integrated directly, so that will need to be addressed. 

## Drivers & IO: 
I have also given the hrldas driver and the IO driver for netcdfs. The hrldas driver requird a change to the namelists, as well as variable initilization. The current WRF model implementation that I am running also builds on this framework, but within the registry files for WRF. 

IO is also added, because in HRLDAS, you must read in a few extra variables (LANDUSEF) from a geogrid file to be able to sort through the land-covers. 
