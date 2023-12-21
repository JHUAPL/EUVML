# EUVML

We provide an 'ML-ready' dataset for the full SOHO EIT and STEREO A&B EUVI instruments from 1996 through 2023.  By 'ML-ready', this means the data has been unified so all images are sun-centered, rotated so solar north is the y-axis, scaled to a uniform plate scale of RSUN=997 (radius of the Sun in arcseconds), and binned down to 512 x 512 pixels.  Updated values in the FITS headers were computed, and unchanged FITS headers are retained.

# What EUVML is
Since SOHO began observing in 1996, full-disk images of the extreme ultraviolet (EUV) Sun in multiple wavelengths have been regularly available, with STEREO providing EUV imaging from off the Sun-Earth line starting in 2007. For the first time, we provide a machine-learning (ML) ready data set comprised of full-disk EUV images across more than two solar cycles from SOHO EIT and STEREO EUVI. To prepare these images for ML applications, we have processed full L1 data sets for each instrument and homogenized the data, accounting for differences in spatial resolution and cadence along with imaging anomalies. This data set will support many science investigations that benefit from multi-viewpoint and/or long-timescale observations, including studies of the evolution of active regions, long-range interactions (e.g., sympathetic flaring), solar irradiance across the solar cycle, and more. The combined L1 data set for both missions (no data added or lost) and the ML-ready data will be made publicly available in the cloud, along with tutorials for easy access to the data.

Creating a new homogenized machine learning (ML) ready data set(ML) from 25 years’ worth of full-disk extreme ultraviolet (EUV) images and handling the issues involved was a great lesson in improving technical processes and also bridging the gap in expertise between scientist and developer. The technical issues our team had to solve included handling large data acquisition, integrating available data processing tools and processing in a cloud environment in a way that was time, space, and cost effective. This involved leveraging previous work done on ML dataset pipelines, experimenting with different techniques of processing data in the cloud, and setting up systems to track data processing steps. Scientists were involved throughout the development cycle of processing the data. Even though it was invaluable for validating the final data and ensuring its usefulness, this collaboration led to unforeseen complications, involving the knowledge gap between scientist and developer and the differing language sets, both technical jargon and programming languages, used by each. These issues, like the technical ones, required leveraging past work and thinking of new ways to document and convey information to solve. Overall, encountering these issues led to the creation of a science conscious cloud-accessible data set.

# When it is coming
We aim to have it available Feb 2024.

# How to access the data
Initially the EUVML dataset will be available in both traditional web-download and in cloud storage.  The traditional storage is at lib.jhuapl.edu.

The AWS Cloud storage (in their 'S3' disks) will initially be in a staging disk, then move to the free NASA TOPS-provided storage.  The NASA TOPS storage allows for free access both within cloud and as 'egress' (downloading it from the cloud to another machine).  In both cases, to access the data in Python, you can install the 'cloudcatalog' PyPi package.
```
%pip install cloudcatalog
import cloudcatalog
cr = cloudcatalog.CatalogRegistry()
endpoint = cr.get_endpoint('GSFC HelioCloud Public Temp')
fr = cloudcatalog.CloudCatalog(endpoint, cache=False)
dataset_id1 = 'euvml'
start = '2020-02-01T00:00:00Z'  # sample date start, change to your values
stop  = '2020-02-02T00:00:00Z'  # sample date end, change to your values
file_registry1 = fr.request_cloud_catalog(dataset_id1, start_date=start, stop_date=stop, overwrite=False)
````

# Processing and the FITS Headers

Below are the processing steps used to convert secchi_prep/eit_prep-ed L1 data into the L1_ML dataset provided.

   (a) Change due to recentering and rescaling:
    CRPIXi: pixel coordinates of sun center, from calibration, float
        e.g. 1021.81, 926.83 -> 128.375, 128.375
        Handled by = SunPy Map.wcs.crpix[2]
    CRPIXiA: same as CRPIXi but from pre-flight calibration
        e.g. 1021.81, 926.823 -> ?
        Handled by = copy of above
    
    (b) Change due to rescaling:
    CDELTi: size of pixel in data units CUNITi
        e.g. 1.587, 1.587 ->
        Handled by = rescale_factor * orig
        e.g. if original data RSUN < unified RSUN (sun is smaller in image)
        zoom in means that new CDELT* = data_RSUN/uni_RSUN i.e. is also smaller
    RSUN: radius of sun in arcseconds, gets jammed to uni_RSUN by zooming

    CDELTiA: same except in degrees
        e.g. -0.00044, 0.00044 -> 0.00132, 0.00132
        Handled by = SunPy Map.wcs.cdelt[2]
        Warning: note SunPy puts Map.wcs.cdelt as 'degrees' but the
        FITS header is actually CDELTiA in degrees, CDELTi in data units
    
    (c) Change due to rebinning:
    NAXISi: length of each axis in pixels
        e.g. 2048, 2048 -> 512, 512
        Hadled by SunPy Map.wcs.naxis[2] or manually setting
    
    (d) Change due to (a) recentering and (b) rescaling and (c) rebinning
    XCEN, YCEN: FOV center of CCD relative to sun center in CDELT1(2) units,
        derived, X(Y)XCEN = CRVAL1(2) + CDELT1(2) * [PC1(2)_1*i + PC1(2)_2*j)
        where i = (NAXIS+1)/2 - CRPIX1, j = (NAXIS2+1)/2 - CRPIX2
        e.g. -10.88, 154.75 ->
        Handled by = run calculation using new header values

    (e) SunPy converts INT 16 to FLOAT -64 (BITPIX), need to convert
    back before saving

    (f) Change documenting that we reprocessed:
    ORIGIN = APL (now)
    DATE: update as time of last file modification, use CCSDS format
    (optionally)
    COMMENT: can add comments
    HISTORY: can add history
    FILENAME: (given we depart from the spec, update to reflect or keep orig?)

    (f) New FITS header element for our reprocessing info
    MLSCALEi: rescale factor used to make uniform sun size in all ML frames
    MLPROC: level of ML reprocessing, 0 = none, 1 = standards, (others tbd)

    (z) Do not change:
    CUNITj, CUNITjA: units for coordinates along detector axis, arcsec
    CRVALi, CRVALiA: coord system reference coords of CRPIX1(2), arcsec
        e.g. RA & Dec
    DSTART(STOP)i: first/last row/col for image area on data array
        in range 1-51, 64-2098. See P1COL etc for why we can ignore
    DSUN_OBS: distance of satellite to sun center
    NAXIS: number of axes aka dimensions, is always 2
    P(R)iROW(COL): CCD rows/cols for start/end of CCD readout, typically
        51-2098, 1-2048, 1292176, 79-2176 e.g. STEREO-A.  Since actual
        data is 2048 these seem to be historical and not needed for data use.
    PCi_j; replaces CROTA

Note that a calibration factor was not applied, and units remain in DN.  We provide a CALFAC in the FITS header for users who wish to convert to photons. The STEREO equation is 15*3.65*wavelength/(13.6*911) (STEREO/SECCHI/EUVI Calibration and Measurement Algorithms Document
https://stereo-ssc.nascom.nasa.gov/publications/CMAD/secchi/STEREO_SECCHI_EUVI_C, MAD_20211206.pdf, page 4, equation 3, get_calfac.pro: line 141-142: gain = 15.0, calfac = gain*(3.65*hdr.WAVELNTH)/(13.6*911); photon/DN)

In general FITS keywords follow the STEREO reference document.  Differences with SOHO include that SOHO has no CDELT1A, CDELTA2A.  SOHO's "SOLAR_R" is retained but the unified "RSUN" is added and RSUN should be used when analyzing the data.


