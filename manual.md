# fMRI Pipeline Manual

## Overview



## Skill Prerequisites



## Pipeline Dependencies



## Tutorial Data



The tutorial data set is one of our functional localizer data sets we have sitting around on the server. It includes only two subjects from the data set to simplify things. The directory containing the data looks like this: 

```
pipeline-tutorial
└── data
    ├── sourcedata
    │   ├── SUB1_MAR5_2014
    │   └── SUB2_MAR10_2014
```

If you want to use the tutorial data set, **make sure you copy over your own separate version** (either to your user account on the server or locally). 

If you do not have access to this data (i.e. you are not in our lab, you do not have access to the server, etc) then you can try to create your own data set with the same directory structure, using two subjects from your own data. This *will* change things (especially when configuring your BIDS conversion), but hopefully you can still follow along. 



# The pipeline



## Converting raw DICOMs to NIfTI

### BIDS format

https://github.com/bids-standard/bids-starter-kit/wiki



https://github.com/bids-standard/bids-starter-kit/wiki/The-BIDS-folder-hierarchy





### Bidsify




#### General Example 1: Single-session conversion

`$ bidsify -d dicom-directory -o bids-directory -c config.json`

`$ bidsify -d dicom-directory -o bids-directory -c config.json --ignore behaviour-folder`

#### General Example 2: Multi-session conversion

`$ bidsify -d dicom-directory -o bids-directory -c config.json -s ses-01 -m submap.json`

`$ bidsify -d dicom-directory -o bids-directory -c config.json -s ses-02 -m submap.json`



### Tutorial Example: 

First, let us begin with the initial step to determine what properties are needed to map the raw DICOM images to a formated BIDS directory. As mentioned earlier, this step cannot be automated.

Make sure you are in the `pipeline-tutorial` directory. Then, you'll need to run `dcm2bids_helper` on one folder to get out the initial conversion conversion by `dcm2niix`. This will produce the full NIfTI images (`.nii.gz` files), and the meta-data files (`.json` files) that `dcm2bids` uses to guide BIDS formatting.   

`$ dcm2bids_helper -d data/sourcedata/SUB1_MAR5_2014 -o ./`

This will create a temporary directory that contains the converted example in the `helper` directory:

```
pipeline-tutorial
└── data
    ├── sourcedata
    │   ├── SUB1_MAR5_2014
    │   └── SUB2_MAR10_2014
    └── tmp_dcm2bids
        └── helper
```

Inside `helper`, you'll see your NIfTIs and your meta data files. For example: 

```
002_SUB1_MAR5_2014_CNS_SAG_MPRAGE_ISO_20140305125615.json (anatomical metadata)
002_SUB1_MAR5_2014_CNS_SAG_MPRAGE_ISO_20140305125615.nii.gz (anatomical T1w image)
003_SUB1_MAR5_2014_ep2d_bold_TR2000_350Vols_20140305125615.json (functional metadata 1)
003_SUB1_MAR5_2014_ep2d_bold_TR2000_350Vols_20140305125615.nii.gz (functional image 1)
004_SUB1_MAR5_2014_ep2d_bold_TR2000_350Vols_MoCo_20140305125615.json (functional metadata 2)
004_SUB1_MAR5_2014_ep2d_bold_TR2000_350Vols_MoCo_20140305125615.nii.gz (functional image 2)
```

`dcm2niix` automatically determines the file names using the protocol defined at the time of the scanning. Each .json contains key-value pairs for each property (e.g., acquisition time and date, sequence, etc). These .json files are critical for using `dcm2bids`, and by extension `bidsify` because they determine which image is when converting your data to BIDS. 

If you open up the `.json` file for both functional images, you can see that these images are actually the same: they were acquired at the same time. The difference is that`004` is the Siemens motion-corrected data, which is typically indicated in the `SeriesDescription` and/or ` ProtocolName`. For `dcm2bids`, and thus `bidsify`, **you'll want to identify the meta-data properties that distinguish functional runs from different tasks, and from the anatomical image(s) **. 

Going through the dataset, I can create a BIDS configuration file for `dcm2bids`/`bidsify` using properties from the meta-data files:

```json
{
  "descriptions": [
    {
      "dataType": "func",
      "modalityLabel": "bold",
      "customLabels": "task-aud",
      "criteria": {
          "ProtocolName": "*ep2d_bold_TR2000_250Vols_MoCo*"
      }
    },
    {
      "dataType": "func",
      "modalityLabel": "bold",
      "customLabels": "task-motor",
      "criteria": {
          "ProtocolName": "*ep2d_bold_TR2000_350Vols_MoCo*"
      }
    },
    {
      "dataType": "anat",
      "modalityLabel": "T1w",
      "criteria": {
        "SeriesDescription": "CNS_SAG_MPRAGE_ISO"
      }
    }
  ]
}
```

There are two types of functional images I want to use in the data sets: The auditory localizer images (`task-aud`), and the motor localizer images (`task-motor`). The `dataType` property is set as `func` and `modalityLabel` is set as `bold`. These tell `dcm2bids` that these are indeed functional images. `customLabels` will label the filename according to the task, which is a requirement of BIDS (make sure you prefix with `task-`)'. Then, under `criteria`, you will see that I used the `ProtocolName` to identify the motion corrected files for each task (`*` indicates wildcard characters for pattern matching). 

The anatomical image in this example has different values for `dataType` and `modalityLabel`, along with using its `SeriesDescription` for the `criteria`. If you had a T2-weighted anatomical image, you would change `modalityLabel` to `T2w`.  

Be sure to read the [`dcm2bids` documentation](https://cbedetti.github.io/Dcm2Bids/config/) to get a full understanding of the configuration file. Save the file as `bids_config.json` in the top-level directory. Then, run `bidsify`:

`$ bidsify -d data/sourcedata/ -o data/ -c bids_config.json`

After running, your project directory should look like this:

```
pipeline-tutorial
├── bids_config.json
└── data
    ├── CHANGES
    ├── dataset_description.json
    ├── participants.tsv
    ├── README
    ├── sourcedata
    ├── sub-01
    ├── sub-02
    └── tmp_dcm2bids
```

Apart from the `tmp_dcm2bids` directory (which you can delete once you're satisfied with the conversion), this is a minimal BIDS directory! Navigate to one of the `sub` directories and you can see the data structure for the subject:

```
sub-01
├── anat
│   ├── sub-01_T1w.json
│   └── sub-01_T1w.nii.gz
└── func
    ├── sub-01_task-aud_run-01_bold.json
    ├── sub-01_task-aud_run-01_bold.nii.gz
    ├── sub-01_task-aud_run-02_bold.json
    ├── sub-01_task-aud_run-02_bold.nii.gz
    ├── sub-01_task-motor_run-01_bold.json
    ├── sub-01_task-motor_run-01_bold.nii.gz
    ├── sub-01_task-motor_run-02_bold.json
    ├── sub-01_task-motor_run-02_bold.nii.gz
    ├── sub-01_task-motor_run-03_bold.json
    ├── sub-01_task-motor_run-03_bold.nii.gz
    ├── sub-01_task-motor_run-04_bold.json
    └── sub-01_task-motor_run-04_bold.nii.gz
```

Congratulations, you now have your first BIDS dataset! **Be sure to check everything to ensure that it is correct.**  You will notice that you now have the appropriate number of functional images in the `func` directory, and that the `anat` directory contains the one anatomical image. 

## Quality Control 









 

## Minimal Preprocessing


### fMRIprep


## Data Extraction and Post-Processing

The final step in the pipeline is to obtain ready-to-analyze data from your preprocessed functional data. For pretty much all analyses other than conventional GLM-based analyses, these data are the timeseries for each region of interest in your project. Multivariate pattern analyses (MVPA; or any voxel-level analysis) requires the timeseries for each individual voxel in a region. Meanwhile, typical functional connectivity or BOLD timecourse analyses will average across voxels to generate a mean timeseries for a region. Ultimately, the goal is to have your data in a tabular format that can be loaded and analyzed by code of any programming language.  

In addition to extracting out the data, you will also need to apply some additional post-processing steps to ensure that you properly denoise the data. This can include temporal filtering, detrending, nuisance regression, spatial smoothing, and discarding initial unstable volumes of each scan.  Applying post-processing is especially important for functional connectivity analyses where noisy data can induce spurious correlations and false-positives in your data.   

### Niimasker

Data can be extracted and post-processed using [niimasker](https://github.com/danjgale/nii-masker), which takes in functional images and returns tabular data files of extracted and processed regions. niimasker is essentially a command-line interface for some key functions in [nilearn](https://nilearn.github.io/index.html) (the package that niimasker's name pays homage to) with some extra bells and whistles thrown in. In addition to extracting data, it generates reports for each data file it created, which fully documents the extraction and allows you to quickly inspect your data quality. 

niimasker has a flexible interface that lets you specify everything directly as a command-line argument or put everything in a `.json` file. I prefer the latter, as it is easier to set. Once installed (refer to its  documentation), simply call niimasker to view the help documentation:

`$ niimasker -h`

niimasker requires command line argument, the output folder, and the rest can be set either on the command line or in the `.json` file (whatever is set in the  `.json` file takes priority if both ways are used). I highly recommend using niimasker as so:

`$ niimasker your-output-directory/ -c config.json`

And then in `config.json` set everything there (defaults are shown below just for example's sake):

 ```json
{
  "input_files": [],
  "mask_img": "",
  "labels": [],
  "regressor_files": null,
  "regressor_names": [],
  "as_voxels": false,
  "standardize": false,
  "t_r": null,
  "detrend": false,
  "high_pass": null,
  "low_pass": null,
  "smoothing_fwhm": null,
  "discard_scans": null,
  "n_jobs": 1
}
 ```

Please refer to niimasker's documentation to learn more and familiarize yourself with how to use to use it. Next, I will go over some use-cases for typical analyses done in the lab. 

### Conventional GLM

Conventional GLM do not require data extraction and post-processing the same way that MVPA or functional connectivity analyses do because you feed the functional images directly into your GLM software of choice (SPM, FSL, nistats, etc.). However, you may want to perform an ROI analysis using the betaweights of your GLMs.  For this, you will first need to concatenate all of your betaweight images generated by these programs into a single image for each subject, so that the time domain of each image corresponds to the number of experimental conditions that you are interested in. You will also need a binary mask for an ROI or atlas file to define your regions. 

In a two subject example, a configuration would look similar to this:

```json
{
  "input_files": ["sub-01_betas.nii.gz", "sub-02_betas.nii.gz"],
  "mask_img": "roi_mask.nii.gz",
  "as_voxels": true
}
```

This is a really simple use-case that pulls out the "timeseries" for each voxel in the mask. The "timeseries" in this case is just the betaweight for each experimental condition because you concatenated the betaweight images for each condition. Because you are looking at betaweights, you do not need to apply any post-processing on the data. 

If you had an atlas file and were interested in multiple regions, the configuration file would be something like so:

```json
{
  "input_files": ["sub-01_betas.nii.gz", "sub-02_betas.nii.gz"],
  "mask_img": "your_atlas.nii.gz",
  "labels": atlas_labels.tsv
}
```

This would pull out the average betaweight of every experimental condition for each region defined by the atlas. 

### Multivariate pattern analyses

### Functional Connectivity



## 

## Glossary

