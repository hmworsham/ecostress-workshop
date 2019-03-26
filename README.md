# Working with ECOSTRESS Evapotranspiration Data
---
# Objective:

### This tutorial demonstrates how to work with the ECOSTRESS Evapotranspiration PT-JPL Daily L3 Global 70m Version 1 ([ECO3ETPTJPL.001](https://doi.org/10.5067/ECOSTRESS/ECO3ETPTJPL.001)) data product in Python.
The Land Processes Distributed Active Archive Center (LP DAAC) distributes the Ecosystem Spaceborne Thermal Radiometer Experiment on Space Station ([ECOSTRESS](https://ecostress.jpl.nasa.gov/)) data products. The ECOSTRESS mission is tasked with measuring the temperature of plants to  better understand how much water plants need and how they respond to stress.  ECOSTRESS products are archived and distributed in the HDF5 file format as swath-based products.

In this tutorial, you will use Python to perform a swath to grid conversion to project the swath data on to a grid with a defined coordinate reference system (CRS), compare ECOSTRESS data with ground-based AmeriFlux flux tower observations, and export science dataset (SDS) layers as GeoTIFF files that can be loaded into a GIS and/or Remote Sensing software program.

***
### Example: Converting a swath ECO3ETPTJPL.001 HDF5 file into a GeoTIFF with a defined CRS and comparing ECOSTRESS Evapotranspiration (ET) with ground-based ET observations from an AmeriFlux flux tower location in California.           
#### Data Used in the Example:  
- Data Product: ECOSTRESS Evapotranspiration PT-JPL Daily L3 Global 70m Version 1 ([ECO3ETPTJPL.001](https://doi.org/10.5067/ECOSTRESS/ECO3ETPTJPL.001))  
     - Science Dataset (SDS) layers:  
        - ETinst     
        - ETinstUncertainty    
- Data Product: ECOSTRESS Geolocation Daily L1B Global 70m Version 1 ([ECO1BGEO.001](https://doi.org/10.5067/ECOSTRESS/ECO1BGEO.001))  
     - Science Dataset (SDS) layers:  
        - Latitude     
        - Longitude    
- Data Product: AmeriFlux Ground Observations for [Flux Tower US-CZ3](https://ameriflux.lbl.gov/sites/siteinfo/US-CZ3): Sierra Critical Zone, Sierra Transect, Sierran Mixed Conifer, P301
     - Variables:  
        - Latent Heat (W/m<sup>2</sup>)     
***  
# Topics Covered:
1. **Getting Started**    
    1a. Import Packages    
    1b. Set Up the Working Environment      
    1c. Retrieve Files    
2. **Importing and Interpreting Data**    
    2a. Open an ECOSTRESS HDF5 File and Read File Metadata     
    2b. Subset SDS Layers   
3. **Performing Swath2grid Conversion**    
    3a. Import Geolocation File  
    3b. Define Projection and Output Grid  
    3c. Read SDS Metadata  
    3d. Perform K-D Tree Resampling  
    3e. Basic Image Processing  
4. **Exporting Results**    
    4a. Set Up a Dictionary  
    4b. Define CRS and Export as GeoTIFFs  
5. **Combining ECOSTRESS and AmeriFlux Tower Data**    
    5a. Loading Tables with Pandas  
    5b. Locate ECOSTRESS Pixel from Lat/Lon Coordinates  
6. **Visualizing Data**      
    6a. Create Colormap   
    6b. Plot ET Data  
    6c. Exporting an Image  
7. **Comparing Observations**      
    7a. Calculate Distribution of ECOSTRESS Data    
    7b. Visualize Ground Observations  
    7c. Combine ECOSTRESS and Ground Observations  

***
# Before Starting this Tutorial:
### Dependencies:
- This tutorial was tested using Python 3.6.7.  
- A [NASA Earthdata Login](https://urs.earthdata.nasa.gov/) account is required to download the data used in this tutorial. You can create an account at the link provided.  
- Libraries: h5py, numpy, pandas, matplotlib, datetime, pyproj, pyresample, gdal
- The [Geospatial Data Abstraction Library](http://www.gdal.org/) (GDAL) is required.

#### For a complete list of packages and a yml file that can be used to create your own environment, check out [windows-py.yml](https://git.earthdata.nasa.gov/projects/LPDUR/repos/tutorial-ecostress/browse/windows-py.yml) or [mac-py.yml](https://git.earthdata.nasa.gov/projects/LPDUR/repos/tutorial-ecostress/browse/mac-py.yml).  
   - Need help setting up an environment? check out the Python Environment Setup section in the README.
***
### Example Data:
This tutorial uses an ECO3ETPTJPL.001 (and accompanying ECO1BGEO.001) observation from August 05, 2018. Until the data are released publicly, you can download the files directly via [NASA Earthdata Search](https://search.earthdata.nasa.gov/search/project?p=!C1534730469-LPDAAC_ECS!C1534584923-LPDAAC_ECS&pg[2][x]=1571874880!LPDAAC_ECS&m=35.77587890625!-122.1240234375!6!1!0!0%2C2&qt=2018-08-05T00%3A00%3A00.000Z%2C2018-08-05T23%3A00%3A00.000Z&q=ecostress&ok=ecostress&sp=-119.1951%2C37.0674) using the query in the link provided. Note that NASA Earthdata Search will only work for Early Adopters. You can alternatively download the granules below from the following location: https://ecostress.jpl.nasa.gov/mar21workshop (follow the Workshop Training Material link).   
- ECOSTRESS_L1B_GEO_00468_007_20180805T220314_0502_02.h5
- ECOSTRESS_L3_ET_PT-JPL_00468_007_20180805T220314_0502_02.h5   

The AmeriFlux Latent Heat data used in Section 4 can be downloaded via a [csv file](https://git.earthdata.nasa.gov/projects/LPDUR/repos/tutorial-ecostress/raw/tower_data.csv?at=refs%2Fheads%2Fmaster).

### Source Code used to Generate this Tutorial:
- [Jupyter Notebook](https://git.earthdata.nasa.gov/projects/LPDUR/repos/tutorial-ecostress/browse/ECOSTRESS_Tutorial.ipynb)  

#### If you are simply looking to batch process/perform the swath2grid conversion for ECOSTRESS files, be sure to check out the [ECOSTRESS Swath to Grid Conversion Script](https://git.earthdata.nasa.gov/projects/LPDUR/repos/ecostress_swath2grid/browse).

NOTE: This tutorial was developed specifically for the ECOSTRESS Evapotranspiration PT-JPL Level 3, Version 1 HDF5 files and will need to be adapted to work with other ECOSTRESS products.
---
# Procedures:
## Getting Started:
> #### 1.	Download ECOSTRESS_L3_ET_PT-JPL_00468_007_20180805T220314_0502_02.h5 and ECOSTRESS_L1B_GEO_00468_007_20180805T220314_0502_02.h5 from the [LP DAAC Data Pool](https://e4ftl01.cr.usgs.gov/) or [Earthdata Search Client](http://search.earthdata.nasa.gov) (if both fail, try: https://ecostress.jpl.nasa.gov/mar21workshop) to a local directory.
> #### 2.	Copy/clone/download  [ECOSTRESS_Tutorial.ipynb](https://git.earthdata.nasa.gov/projects/LPDUR/repos/tutorial-ecostress/browse/ECOSTRESS_Tutorial.ipynb) from LP DAAC Data User Resources Repository   
> #### 3.	Download  [tower_data.csv](https://git.earthdata.nasa.gov/projects/LPDUR/repos/tutorial-ecostress/raw/tower_data.csv?at=refs%2Fheads%2Fmaster) from LP DAAC Data User Resources Repository (make sure all files are located in the same directory).   
## Python Environment Setup
> #### 1. It is recommended to use [Conda](https://conda.io/docs/), an environment manager to set up a compatible Python environment. Download Conda for your OS here: https://www.anaconda.com/download/. Once you have Conda installed, Follow the instructions below to successfully setup a Python environment on MacOS or Windows.  
> 1a. If you already have Conda installed, please run `conda update conda` before getting started below.

> #### 2. Windows Setup
> 1.  Download the [windows-py.yml](https://git.earthdata.nasa.gov/projects/LPDUR/repos/tutorial-ecostress/browse/windows-py.yml) file from the repository.  
> 2. Navigate to the yml file location in your Command Prompt, and type `conda env create -f windows-py.yml`
> 3. Navigate to the directory where you downloaded the `ECOSTRESS_Tutorial.ipynb` Jupyter Notebook.
> 4. Activate ECOSTRESS Python environment (created in step 3) in the Command Prompt  
    > 5a. Type `activate ecostress`   
> #### 3. MacOS Setup
> 1.  Download the [mac-py.yml](https://git.earthdata.nasa.gov/projects/LPDUR/repos/tutorial-ecostress/browse/mac-py.yml) file from the repository.  
> 2. Navigate to the directory containing the `mac-py.yml` file in your Command Prompt, and type `conda env create -f mac-py.yml`
> 3. Navigate to the directory where you downloaded the  `ECOSTRESS_Tutorial.ipynb` Jupyter Notebook.  
> 4. Activate ECOSTRESS Python environment (created in step 3) in the Command Prompt   
    > 5a. Type `source activate ecostress`  

[Additional information](https://conda.io/docs/user-guide/tasks/manage-environments.html) on setting up and managing Conda environments.
## Script Execution
> #### 1.	Once you have set up your MacOS/Windows environment and it has been activated, run the Jupyter Notebook with the following in your Command Prompt/terminal window:
  > 1.  `Jupyter Notebook`  
    > 1a. This should open a new Jupyter dashboard in your web browser.
  > 2. Open `ECOSTRESS_Tutorial.ipynb`
  > 3. Run through the tutorial.   
---
# Contact Information:
#### Author: Cole Krehbiel¹  and Gregory Halverson<sup>2</sup>
**Contact:** LPDAAC@usgs.gov  
**Voice:** +1-866-573-3222  
**Organization:** Land Processes Distributed Active Archive Center (LP DAAC)  
**Website:** https://lpdaac.usgs.gov/  
**Date last modified:** 03-13-2019  

¹Innovate!, Inc., contractor to the U.S. Geological Survey, Earth Resources Observation and Science (EROS) Center,  
 Sioux Falls, South Dakota, USA. Work performed under USGS contract G15PD00467 for LP DAAC<sup>3</sup>.  
<sup>3</sup>LP DAAC Work performed under NASA contract NNG14HH33I.  
<sup>2</sup>Jet Propulsion Laboratory, California Institute of Technology, Pasadena, CA, USA
