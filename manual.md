# fMRI Pipeline Manual

## Overview



### Skill prerequisites

- Familiarity with bash commands and the command-line
- Familiarity with `.json` files
- Familiarity with `pip` for installing the Python packages in the pipeline 

### Pipeline dependencies



### Tutorial data

The tutorial data set is a subset of data from one of our functional localizer data sets we have sitting around on the server. The localizer includes one T1-weighted anatomical image, two auditory functional localizer images, and 4 motor functional localizer images. All of the DICOM files from the two subjects are provided. The directory containing the data looks like this: 

```
pipeline-tutorial
└── data
    ├── sourcedata
    │   ├── SUB1_MAR5_2014
    │   └── SUB2_MAR10_2014
```

If you want to use the tutorial data set, **make sure you copy over your own separate version on your user account**. On the server, run:

`$ cp -r Raid6/raw/pipeline-tutorial your-desired-destination` 

If you do not have access to this data (i.e. you are not in our lab, you do not have access to the server, etc) then you can try to create your own data set with the same directory structure, using two subjects from your own data. This *will* change things, especially when configuring your BIDS conversion, but hopefully you can still follow along. 

## The pipeline: Step-by-step

### 1. Converting raw DICOMs to NIfTI

#### BIDS format

https://github.com/bids-standard/bids-starter-kit/wiki



https://github.com/bids-standard/bids-starter-kit/wiki/The-BIDS-folder-hierarchy





#### Bidsconv

[`bidsconv`](https://github.com/danjgale/bidsconv) is a custom solution I created to convert the lab data into BIDS. There are [several different tools to convert any data into BIDS format](https://bids.neuroimaging.io/#benefits). `bidsconv` uses one of these tools, [`dcm2bids`]() to convert single-subject data into BIDS specification. `bidsconv` essentially iterates through each subject and runs `dcm2bids`, and automatically creates some additional BIDS files (e.g., `README`, `participants.tsv`, etc) at the end of  conversion. 

Under the hood, `dcm2bids` runs a popular DICOM-to-NIfTI conversion tool, [`dcm2niix`]() (hence the name). With `dcm2bids`, you have to run `dcm2niix` first on one subject so you can pull out the images' meta-data and then use that meta-data to determine how you want to map NIfTI files to BIDS. This mapping is set in a configuration file that can be used for all your subjects. `dcm2bids` has a tool called `dcm2bids_helper`, which essentially just runs `dcm2niix` to extract out this information so you can create your own configuration file. This is all explained in the  [`dcm2bids` documentation](https://cbedetti.github.io/Dcm2Bids/config/). 

Because `bidsconv` just runs `dcm2niix` iteratively, you have to run `dcm2bids_helper` first to identify meta-data for your BIDS conversion. Once that is obtained, you can just run `bidsconv` in the terminal. The workflow is  summarized below: 

`figure here`

##### Installing bidsconv

Installation instructions can also be found in [`bidsconv`'s README.](https://github.com/danjgale/bidsconv#installation)

You must have `dcm2bids` installed first. To install `dcm2bids`:

`pip install dcm2bids`

Then, install `bidsconv` in a directory of your choice:

```
git clone git@github.com:danjgale/bidsconv.git
cd bidsconv/
pip install -e .
```

This should install `bidsconv` to your Python environment. Restart the terminal and run `bidsconv -h` to test out the installation, which will print out the tool's help if installed correctly.

##### General Example 1: Single-session conversion

`$ bidsconv -d dicom-directory -o bids-directory -c config.json`

`$ bidsconv -d dicom-directory -o bids-directory -c config.json --ignore behaviour-folder`

##### General Example 2: Multi-session conversion

`$ bidsconv -d dicom-directory -o bids-directory -c config.json -s ses-01 -m submap.json`

`$ bidsconv -d dicom-directory -o bids-directory -c config.json -s ses-02 -m submap.json`

#### Tutorial data: 

First, let us begin with the initial step to determine what properties are needed to map the raw DICOM images to a formated BIDS directory. 

Make sure you are in the `pipeline-tutorial` directory. Then, run `dcm2bids_helper` on one folder to get out the initial conversion conversion by `dcm2niix`. This will produce the full NIfTI images (`.nii.gz` files), and the meta-data files (`.json` files) that `dcm2bids` uses to guide BIDS formatting.   

`$ dcm2bids_helper -d data/sourcedata/SUB1_MAR5_2014 -o data/`

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

`dcm2niix` automatically determines the file names using the protocol defined at the time of the scanning. Each .json contains key-value pairs for each property (e.g., acquisition time and date, sequence, etc). These .json files are critical for using `dcm2bids`, and by extension `bidsconv` because they determine which image is when converting your data to BIDS. 

If you open up the `.json` file for both functional images, you can see that these images are actually the same: they were acquired at the same time. The difference is that`004` is the Siemens motion-corrected data (not to be confused with motion correction performed during preprocessing), which is typically indicated in the `SeriesDescription` and/or ` ProtocolName`.  In our BIDS data, we will only want one version of each functional image, not both. We typically use the motion-corrected data. 

When looking at the meta-data,  **you'll want to identify the properties that distinguish functional runs from different tasks, and from the anatomical image(s) ** when setting up your `dcm2bids`/`bidsconv` configuration file. 

Going through the dataset, I created a BIDS configuration file for `dcm2bids`/`bidsconv` using properties from the meta-data files:

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

**Be sure to read the [`dcm2bids` documentation](https://cbedetti.github.io/Dcm2Bids/config/) to get a full understanding of the configuration file.** Afterwards, copy and paste the example above into a new file called `bids_config.json` in the top-level directory. 

Then, run `bidsconv`:

`$ bidsconv -d data/sourcedata/ -o data/ -c bids_config.json`

After running, your project directory should look like this:

```
pipeline-tutorial
├── bids_config.json
└── data
    ├── CHANGES
    ├── dataset_description.json
    ├── derivatives
    ├── participants.tsv
    ├── README
    ├── sourcedata
    ├── sub-01
    ├── sub-02
    └── tmp_dcm2bids
```

Aside from the `tmp_dcm2bids` directory (which you can delete once you're satisfied with the conversion), this is a minimal BIDS directory! Navigate to one of the `sub` directories and you can see the data structure for the subject:

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

You will notice that you now have the appropriate number of functional images in the `func` directory (2 auditory, 4 motor), and that the `anat` directory contains the one anatomical image. 

**But you are not done yet**. Although a blank `dataset_description.json` was created, it needs to be filled with the following (update the BIDS version accordingly):

```
{
  "BIDSVersion": "1.2.0",
  "Name": "Pipeline Tutorial"
}
```

As well, task description files for the auditory and motor tasks must be manually created. First, create `task-aud_bold.json` in the BIDS directory, and fill with: 

```
{
  "RepetitionTime": 2.0,
  "TaskName": "Auditory Localizer Task"
}
```

Next, create `task-motor_bold.json` and fill with:

```
{
  "RepetitionTime": 2.0,
  "TaskName": "Motor Localizer Task"
}
```

For running tools such as `fmriprep` and `mriqc`, the `CHANGES` and `README` files cannot be blank. In each file just put their respective name in the first line (i.e. "Changes" in `CHANGES`) so that they are not empty. Lastly, add `tmp_dcm2bids` to the first line in `.bidsignore` so that it is ignored by BIDS validation tools. So, now your directory structure should look like this:

```pipeline-tutorial
├── bids_config.json
└── data
    ├── CHANGES
    ├── dataset_description.json
    ├── derivatives
    ├── participants.tsv
    ├── README
    ├── sourcedata
    ├── sub-01
    ├── sub-02
    ├── task-aud_bold.json
    ├── task-motor_bold.json
    └── tmp_dcm2bids
```

Congratulations, you now have your first BIDS dataset! **Be sure to check everything to ensure that it is correct.** 

### 2. Quality control  

### 3. Minimal Preprocessing

Minimal preprocessing refers to the preprocessing applied to data that needs to be done regardless of the project itself. These steps are pretty much ubiquitous for all fMRI projects and pretty much all fMRI packages have functions that perform various preprocessing steps. If you are unfamiliar with how to preprocess fMRI data, I would recommend reading the [Handbook of Functional MRI Analysis](http://www.fmri-data-analysis.org/) or reading through the preprocessing section of [Chen & Glover (2015)](https://mriquestions.com/uploads/3/4/5/7/34572113/chen_glover_fmri_review_2015art3a10.10072fs11065-015-9294-9.pdf). 

#### fMRIprep

In the pipeline, all minimal preprocessing is done with [`fmriprep`](https://fmriprep.readthedocs.io/en/stable/index.html) ([also see the paper here](https://www.nature.com/articles/s41592-018-0235-4)). `fmriprep` is a comprehensive automated preprocessing pipeline for fMRI data. It is built using Python's  [`nipype`](https://nipype.readthedocs.io/en/latest/), which links together existing MRI packages (e.g., `FSL`, `AFNI`, `Freesurfer`) using a powerful and flexible workflow engine. `fmriprep` aims to use the best available tool for each preprocessing step and its workflows are fully transparent to the user. `fmriprep` is a fully open-source effort that is driven by members of the [Poldrack Lab](https://poldracklab.stanford.edu/) at the [Standford Centre for Reproducible Neuroscience](http://reproducibility.stanford.edu/) (with contributions from community members all over the globe), and is rapidly growing in features and users.

A full breakdown of how `fmriprep` preprocesses data can found [here](https://fmriprep.readthedocs.io/en/stable/workflows.html). For functional data, preprocessing includes motion correction, slice time correction, distortion correction (if made possible), surface projection, and spatial normalization. Anatomical data is intensity normalized, skull stripped, spatially normalized, and run through the full `Freesurfer` surface pipeline. On top of the preprocessing, `fmriprep` also extracts a huge number of confounds (e.g., motion parameters, signals from white matter and cerebral spinal fluid) for nuisance regression, and produces comprehensive visual reports for quality inspection and documentation.  

#### Preprocessing the tutorial data

`fmriprep` can be run by copying and pasting the code in `fmriprep_cmd.txt` into the terminal. Make sure to fix the paths so that they correctly reflect where `pipeline-tutorial` and your [Freesurfer license](https://surfer.nmr.mgh.harvard.edu/fswiki/License) file are:

```
fmriprep-docker /absolute/path/to/pipeline-tutorial/data \
/absolute/path/to/pipeline-tutorial/data/derivatives participant \
--fs-license-file /absolute/path/to/freesurfer/license.txt \
-w /absolute/path/to/pipeline-tutorial/data/derivatives/fmriprep-wd \
--output-spaces MNI152NLin6Asym:res-2 MNI152NLin2009cAsym:res-2
```

This code will generate the `fmriprep` output in the `derivatives/` folder of your BIDS dataset, along with a `freesurfer` directory containing all `Freesurfer` files. As well, I explicitly set the `-w` parameter, which is the working directory. This stores all of `fmriprep`'s temporary files and cache. This *will* a very large amount of disk space and I recommend you delete this once everything has run successfully. 

In the `--output-spaces` parameter, I also specify two versions of MNI space. `MNI152NLin6Asym` refers to the conventional MNI template packaged with `FSL`, and `MNI152NLin2009cAsym` is a different version that typically yields better normalization results. I specified both because the most atlases are provided in the former, but the latter is the default for `fmriprep`. `res-2` refers to using a spatial resolution of 2mm isovoxel. Additionally, data will also be generated in `fsaverage5` space by default.  As noted in the [`fmriprep` documentation]([https://fmriprep.readthedocs.io/en/stable/usage.html#Workflow%20configuration](https://fmriprep.readthedocs.io/en/stable/usage.html#Workflow configuration)), you can also specify your output spaces to participants' native space, anatomical space, or other standard templates. 

The current settings above will take up a substantial amount of disk space. If you are concerned about disk space, I would recommend removing the `--output-spaces` line, which will default to producing data in `MNI152NLin2009cAsym` space at native resolution. 

Running `fmriprep` will take roughly a day to run. The directory structure should look like this:

```
pipeline-tutorial
├── bids_config.json
├── data
│   ├── CHANGES
│   ├── dataset_description.json
│   ├── derivatives
│   │   ├── fmriprep
│   │   ├── fmriprep-wd
│   │   └── freesurfer
│   ├── participants.tsv
│   ├── README
│   ├── sourcedata
│   │   ├── SUB1_MAR5_2014
│   │   └── SUB2_MAR10_2014
│   ├── sub-01
│   │   ├── anat
│   │   └── func
│   ├── sub-02
│   │   ├── anat
│   │   └── func
│   ├── task-aud_bold.json
│   ├── task-motor_bold.json
│   └── tmp_dcm2bids
│       ├── helper
│       ├── log
│       ├── sub-01
│       └── sub-02
└── fmriprep_cmd.txt
```

And within `data/derivatives/fmriprep`:

```
fmriprep
├── dataset_description.json
├── desc-aparcaseg_dseg.tsv
├── desc-aseg_dseg.tsv
├── logs
│   ├── CITATION.bib
│   ├── CITATION.html
│   ├── CITATION.md
│   └── CITATION.tex
├── sub-01
│   ├── anat
│   ├── figures
│   └── func
├── sub-01.html
├── sub-02
│   ├── anat
│   ├── figures
│   └── func
└── sub-02.html
```

If you navigate to `sub-01`, you will see that 182 files were generated in this example!  To highlight some key files:

- `anat/sub-01_space-MNI152NLin2009cAsym_desc-preproc_T1w.nii.gz` and `anat/sub-01_space-MNI152NLin6Asym_desc-preproc_T1w.nii.gz` are the normalized anatomical images in their respective spaces. The unormalized preprocessed anatomical image is `anat/sub-01_desc-preproc_T1w.nii.gz`.
- In the `func/` directory, files ending in `desc-preproc_bold.nii.gz` are the fully preprocessed functional images. These are the images you will use for volume-based analysis (refer to the `fsaverage5_hemi-L/R.func.gii` files for surface-based analyses) 
- In the `func/` directory, files ending in `confounds_regressors.tsv` are the associated nuisance regressors extracted by `fmriprep`. These are the regressors you can use for denoising your data (e.g., motion parameters from motion correction, white matter and cerebral spinal fluid signals, etc). 

Definitely check out the [`fmriprep` documentation](https://fmriprep.readthedocs.io/en/stable/outputs.html#preprocessed-data-fmriprep-derivatives) to see a full list of which files are which. In the main `fmriprep` directory you will also see visual report, `sub-01.html` . Descriptions of the plots shown in the visual reports are found [here](https://fmriprep.readthedocs.io/en/stable/outputs.html#confounds-and-carpet-plot-on-the-visual-reports).

### 4. Data Extraction and Post-Processing

Minimal preprocessing implies that additional analysis-specific preprocessing must be done afterwards.  The final step in the pipeline is to obtain ready-to-analyze data from your preprocessed functional data. For pretty much all analyses other than conventional GLM-based analyses, ready-to-use data are the timeseries for each region of interest in your project. Multivariate pattern analyses (MVPA; or any voxel-level analysis) requires the timeseries for each individual voxel in a region. Meanwhile, typical functional connectivity or BOLD timecourse analyses will average across voxels to generate a mean timeseries for a region. Ultimately, the goal is to have your data in a tabular format that can be loaded and analyzed by code of any programming language.  

In addition to extracting out the data, you will also need to apply some additional post-processing steps to ensure that you properly denoise the data. This can include temporal filtering and detrending, nuisance regression, spatial smoothing, and discarding initial unstable volumes of each scan.  Applying post-processing is especially important for functional connectivity analyses where noisy data can induce spurious correlations and false-positives in your data.   

This step can be performed in a number of ways, including rolling out your own post-processing pipeline via popular fMRI packages (e.g., `AFNI`, `FSL`, `SPM`) and extracting out the data in your programming language of choice (e.g., `nilearn` or `nibabel` in Python, `SPM` in MATLAB, `oro.nifti` in R), or using tools such as [`XCP Engine`](https://xcpengine.readthedocs.io/index.html). For the purpose of the lab pipeline, I created a tool called [niimasker](https://github.com/danjgale/nii-masker). 

#### Niimasker

``niimasker` is essentially a command-line interface for some key functions in [`nilearn`](https://nilearn.github.io/index.html) (the package that `niimasker`'s name pays homage to) with some extra bells and whistles thrown in. `niimasker` takes in functional images and returns tabular data files of extracted and processed regions.  In addition to extracting/processing data, it generates reports for each data file it created, which fully documents the extraction and allows you to quickly inspect your data quality. 

##### Installing niimasker

Installing `niimasker` is the same as you would do for `bidsconv`. Navigate to a directory of your choice and enter the following: 

```
$ git clone git@github.com:danjgale/niimasker.git
$ cd niimasker/
$ pip install -e .
```

Run `$ niimasker -h` to output the help information and verify that everything is installed correctly. 

##### Configuring niimasker

niimasker has a flexible interface that lets you specify everything directly as a command-line argument or put everything in a `.json` file. I prefer the latter, as it is easier to set. 

niimasker requires only one command line argument, the output folder, and the rest can be set either on the command line or in the `.json` file (whatever is set in the  `.json` file takes priority if both ways are used). I prefer setting all of the parameters in the configuration file, as this is easier to keep track of and adjust. So, I highly recommend using niimasker in the following manner:

`$ niimasker your-output-directory/ -c config.json`

And then in `config.json` you can set everything there (defaults are shown below just for example's sake):

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

You can refer to [`niimasker`'s documentation'](https://github.com/danjgale/nii-masker#using-niimasker) to learn more about the various parameters shown above. 

##### Atlases vs. ROI masks

`niimasker` can either take a integer-labeled atlas image or a single ROI binary mask. For atlases, only averaged timeseries for each ROI can be extracted; voxel-level data cannot be generated using atlases (this is a result of the underlying `nilearn` function). Meanwhile, providing a binary ROI mask lets you extract either averaged timeseries data or full voxel data. If you wish to extract out voxel-level data for multiple ROIs, you can run `niimasker` for each ROI mask you have. 

#### Niimasker output

Just to show you a typical output from `niimasker`, running `niimasker` with an atlas on the tutorial dataset produces the following (for more detail, refer to the tutorial section):

```

```

In this example, it produces a `timeseries.tsv` file per image, which contains time-by-ROI data. Additionally, two directories are also created: `niimasker_data`/, and `reports`/. `niimasker_data/` contains the `mask_img` that was used, along with `parameters.json` that documents all of the parameters used to generate the timeseries. `reports/` contain an `.html` report for each image, which shows 1) all of the parameters and software used to extract and process the data, 2) the overlay of the mask image onto the mean functional image, 3) the extracted timeseries, and 4) correlations among timeseries and regressors (if used).  The reports provide a fully transparent account of your extraction to encourage sanity checks and proper reporting.

The structure would be the same if the above example had extracted out voxel timeseries for an ROI, the only difference being that each `timeseries.tsv` file contains time-by-voxel data. As well, the `.html` reports just show you the voxel timeseries as a carpet plot and no correlation plots (as these are not usually of interest at the level of voxel data).  

Next, we will cover two examples of how to use `niimasker` with the tutorial data.

##### Example 1: Voxel data for MVPA

```
{
  "input_files": "data/derivatives/fmriprep/sub*/func/*motor*MNI152NLin6Asym*preproc-bold.nii.gz",
  "mask_img": "left_motor_cortex.nii.gz",
  "as_voxels": true,
  "standardize": true,
  "t_r": 2,
  "high_pass": 0.01,
  "discard_scans": 4,
  "n_jobs": 8
}
```

Run the following from the top-level directory where the configuration file is located:

`$ niimasker data/derivatives/motor-cortex-data -c niimasker_config_roi.json`

This will produce the following output: 

```

```

In this example, it produces a `timeseries.tsv` file per image, which contains time-by-voxel data. Additionally, two directories are also created: `niimasker_data`/, and `reports`/. `niimasker_data/` contains the `mask_img` that was used, along with `parameters.json` that documents all of the parameters used to generate the timeseries. `reports/` contain an `.html` report for each image, which shows 1) all of the parameters and software used to extract and process the data, 2) the mask overlay on the mean functional image, and 3) the voxel timeseries displayed as a carpet plot. The reports provide a fully transparent account of your extraction to encourage sanity checks and proper reporting.

One thing to note is that `niimasker` does not require your data to be in BIDS format, and the output of `niimasker` is also not in BIDS format. The reason why BIDS is not enforced is to keep `niimasker` as a fairly flexible tool that is not married to BIDS or some derivative (e.g., `fmriprep`). If the input data *is* BIDS, as it is in the lab pipeline, it is fairly easy to take `niimasker` output and transform it into BIDS style data using the subject and session labels in the file names with simple script in your programming language of choice.

##### Example 2: ROI data for functional connectivity

```
{
  "input_files": "data/derivatives/fmriprep/sub*/func/*motor*MNI152NLin6Asym*preproc-bold.nii.gz",
  "mask_img": "nilearn:schaefer:100-7-2",
  "regressor_files": "data/derivatives/fmriprep/sub*/func/*motor*confounds_regressors.tsv",
  "regressor_names": [
    "trans_x",
    "trans_y",
    "trans_z",
    "rot_x",
    "rot_y",
    "rot_z",
    "csf",
    "white_matter"
  ],
  "as_voxels": false,
  "standardize": true,
  "t_r": 2,
  "detrend": true,
  "high_pass": 0.01,
  "discard_scans": 4,
  "n_jobs": 8
}
```

Run the following from the top-level directory where the configuration file is located:

`$ niimasker data/derivatives/schaefer-100-data -c niimasker_config_atlas.json`

Like the previous example, we get a `timeseries.tsv` file per image, but this time each one is a time-by-ROI data set. Furthermore, the visual reports for atlas data are expanded to also include correlations among timeseries and regressors (if used).  


### Glossary

