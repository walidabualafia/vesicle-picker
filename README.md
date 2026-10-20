![Vesicle Picker banner.png](docs/vesicle_picker_banner_withtext.png)

## Installation ##

1. Ensure [Git](https://github.com/git-guides/install-git) and [Anaconda](https://www.anaconda.com/download) (or [Miniconda](https://docs.anaconda.com/miniconda/)) are installed on your computer.  
2. If you wish to run the Segment Anything model on GPU, ensure an [NVIDIA driver](https://www.nvidia.com/en-us/drivers/) supporting CUDA 12.8 (driver version 570 or newer) is installed on your machine. You do not need to install the CUDA toolkit separately, since PyTorch ships with its own CUDA runtime. A GPU is not necessary to run Segment Anything, but the program will run much slower on CPU.
3. Clone the Vesicle Picker repository:  
	```
	git clone https://github.com/r-karimi/vesicle-picker.git
	```
4. Enter this repository:
	```
	cd vesicle-picker
	```
5. Create a clean conda virtual environment.
	```
	conda create -n vesicle-picker python=3.11
 	conda activate vesicle-picker
 	conda install pip
 	```
6. Check that the PyTorch install command in [`pyproject.toml`](pyproject.toml) suits your computer. By default, `install-pytorch` installs PyTorch built for CUDA 12.8, which supports both NVIDIA A100 (Ampere) and RTX PRO 6000 (Blackwell) GPUs, as well as most other recent NVIDIA GPUs:
	```
	install-pytorch = "pip install 'torch>=2.7' 'torchvision>=0.22' --index-url https://download.pytorch.org/whl/cu128"
	```

	### GPU Installation ###
	- Check your NVIDIA driver version and GPU by running:
		```
		nvidia-smi
		```
	- If the driver version is 570 or newer, keep the default command above. This is required for Blackwell GPUs such as the RTX PRO 6000, which are only supported by PyTorch 2.7 or newer built for CUDA 12.8. Older builds (e.g. `cu118`) will fail on these GPUs with `no kernel image is available for execution on the device`.
	- If the driver is older than 570 and cannot be updated, the CUDA 12.8 build will not run. On older GPUs such as the A100, you can instead use a build for an older CUDA version supported by your driver (see the [PyTorch installation page](https://pytorch.org/get-started/locally/)), for example:
		```
		# Example: CUDA 12.6 (driver 560 or newer). Does not support Blackwell GPUs.
		install-pytorch = "pip install 'torch>=2.7' 'torchvision>=0.22' --index-url https://download.pytorch.org/whl/cu126"
		```
	- If you share one environment across machines with different GPUs (e.g. A100 and RTX PRO 6000 nodes on a cluster), use the default CUDA 12.8 build and make sure all machines have driver 570 or newer.

	### CPU Installation ###
	- Modify `install-pytorch` in `pyproject.toml` to install the CPU build of PyTorch:
		```
 		install-pytorch = "pip install 'torch>=2.7' 'torchvision>=0.22' --index-url https://download.pytorch.org/whl/cpu"
 		```
  
8. Install `vesicle-picker` and dependencies:
	```
	pip install .
	poe install-pytorch
 	```
	To confirm that PyTorch can use your GPU, run:
	```
	python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0)); print(torch.ones(1, device='cuda') + 1)"
	```
	This should print `True`, your GPU name, and `tensor([2.], device='cuda:0')` without errors or warnings about an unsupported CUDA capability.
9. Download the Segment Anything [model weights](https://github.com/facebookresearch/segment-anything#model-checkpoints) from the Segment Anything GitHub page and place them in the `vesicle-picker` repository. We recommend trying with the ViT-L model weights first.
10. Modify csparc_login.ini to match the active cryoSPARC instance from which micrographs will be imported into Vesicle Picker and into which particle locations will be exported. Vesicle Picker requires cryoSPARC v5, and the installed `cryosparc-tools` version must match your cryoSPARC version (e.g. `pip install cryosparc-tools~=5.0.0` for cryoSPARC v5.0.x). A license ID is no longer required. Instead of storing your password in `csparc_login.ini`, you may leave `email` and `password` blank and log in once with `python -m cryosparc.tools login`.

## Usage ##

Vesicle Picker is broadly subdivided into three sub-programs, each of which can be run from a seperate script. These sub-programs are `find_vesicles.py`, `filter_vesicles.py`, and `generate_picks.py`. The scripts can be run from the command line and take one argument, which is the path of the respective parameter file for a script. This parameter file is where one can set all the necessary parameters to execute a sub-program. For instance, the parameters for `filter_vesicles` sub-program consist of general parameters such as pixel size and cryoSPARC instance information, as well as specific parameters such as the minimum area or roundness for a predicted vesicle area to be selected for subsequent analysis.

Before processing your own dataset, we recommend working through the introductory Jupyter notebook [`find_vesicles.ipynb`](tests/find_vesicles.ipynb). This notebook describes how the program imports data residing in cryoSPARC and describes each step of the processing pipeline. It also allows for empirical fine-tuning of the various parameters mentioned above on a test image from a dataset. We note in our paper that a combination of roundness and area postprocessing filters are sufficient to obtain high precision and recall in the task of finding synaptic vesicles. If these are sufficient for your dataset as well, then the parameters that need to be set by the user are as follows.
   
- $\sigma_{space}$, $\sigma_{colour}$, and $d$ for the bilateral filter. Increasing these parameters blurs the image such that Vesicle Picker can operate at higher recall at the expense of precision.
- $Roundness_{min}$ for the roundness filter. The roundness ranges from 0 (i.e. the object is a line) to 1 (i.e. the object is a perfect circle) for every predicted vesicle.
- $Area_{min}$ and $Area_{max}$ for the area filter. This parameter is useful if a user knows the rough size of the vesicles in advance.
- $r_{dilation}$ or $r_{erosion}$ for particle picks offset from the membrane edge. This parameter can be useful if the user has prior knowledge about the position of proteins of interest relative to the lipid bilayer.
- Box size to control the density of the picks. A higher density of picks can lead to an increased likelihood of identifing a protein of interest because one of the picks is more likely to be well-centered on the protein. However, this can come at the cost of increased computational burden as the number of particle images subjected to 2D classification is increased.

There are a variety of other postprocessing filters that can be applied to your data as well. These filters are commented out in [`parameters/filter_vesicles.ini`](parameters/filter_vesicles.ini) by default, but can be uncommented and applied depending on the specific use case of the user. More information about the various postprocessing methods implemented in this library can be found in [`vesicle_picker/postprocessing.py`](vesicle_picker/postprocessing.py).

To process your own full dataset with the command line scripts, follow the steps below:

### cryoSPARC (Part 1) ###

1. [Import](https://guide.cryosparc.com/processing-data/all-job-types-in-cryosparc/import/job-import-movies) your movies, then perform [patch motion correction](https://guide.cryosparc.com/processing-data/all-job-types-in-cryosparc/motion-correction/job-patch-motion-correction) and [patch CTF estimation](https://guide.cryosparc.com/processing-data/all-job-types-in-cryosparc/ctf-estimation/job-patch-ctf-estimation). Vesicle Picker operates on motion corrected micrographs.
   
2. [Curate](https://guide.cryosparc.com/processing-data/all-job-types-in-cryosparc/exposure-curation/interactive-job-manually-curate-exposures) your motion corrected micrographs to discard any micrographs that, for instance, contain no vesicles. Vesicle Picker is designed to operate on the output of a Curate Exposures job.
   
3. Note the project ID, workspace ID, and job ID of your Curate Exposures job. You will input this information into the parameter files to run the sub-programs referenced below.

### Python ###

4. Find vesicles by modifying the [`find_vesicles.ini`](parameters/find_vesicles.ini) parameter file with your desired parameters, ensuring you fill in the correct cryoSPARC information. Also make sure to fill in your cryoSPARC login information, using [`csparc_login.ini`](csparc_login.ini) as a template. Finally, indicate an appropriate output directory for the detected vesicles. The detected vesicles will be stored as Python `.pkl` files in the output directory. We have pre-filled this parameter file with a reasonable set of starting parameters.

	When you're ready, run the [`find_vesicles.py`](find_vesicles.py) script. The script takes a file path to the parameters file as its only argument:
	
 	```
	python find_vesicles.py parameters/find_vesicles.ini
 	```
5. Filter the found vesicles by modifying the [`filter_vesicles.ini`](parameters/filter_vesicles.ini) parameter file, uncommenting the selection filters that you want to use and setting their minimum and maximum values. **Ensure you set the input directory for this script as the output directory of `find_vesicles.py`.** Again, we have pre-filled this parameter file with a reasonable set of starting parameters.
   
   	When you're ready, run [`filter_vesicles.py`](filter_vesicles.py):
   
	```
	python filter_vesicles.py parameters/filter_vesicles.ini
	```
 
6. Generate particle picks by modifying the [`generate_picks.ini`](parameters/generate_picks.ini) parameter file. Set the workspace into which the vesicle picks will be exported. Set the dilation or erosion radius if desired, and set box size parameter to control the density of picks. We recommend picking with a reasonably high density (i.e. a box size less than half of the expected diameter of your protein of interest) and removing duplicate particles later in cryoSPARC. **Ensure you set the input directory for this script as the output directory of `filter_vesicles.py`.**

   	Run [`generate_picks.py`](generate_picks.py):
   
	```
	python generate_picks.py parameters/generate_picks.ini
 	```

	Once this script has finished, you should see a collection of `.pkl` files in the output directory of this script, as well as a new, completed job in cryoSPARC called **Vesicle Picks**. This job will be used for downstream processing in cryoSPARC.

### CryoSPARC (Part 2) ###

7. [Extract](https://guide.cryosparc.com/processing-data/all-job-types-in-cryosparc/extraction/job-extract-from-micrographs) particles from the micrographs that were used as input to `find_vesicles.py`. We recommend extracting with a box size 2x to 3x larger than the box size used to generate picks.

8. Proceed with downstream analysis in cryoSPARC, such as [2D classification](https://guide.cryosparc.com/processing-data/all-job-types-in-cryosparc/particle-curation/job-2d-classification) and [*Ab initio* reconstruction](https://guide.cryosparc.com/processing-data/all-job-types-in-cryosparc/3d-reconstruction/job-ab-initio-reconstruction).

## Tips ##

- We recommend experimenting with different model architectures and downsampling factors to find a good trade-off between accuracy and speed when processing a full dataset. We have found that perfect recall when finding vesicles is usually unnecessary for obtaining a structure. A small set of high-quality vesicles are usually more informative than vesicles mixed with non-vesicle objects, so do not be afraid of stringently filtering your vesicles.

- If you are able to generate good 2D classes of a membrane protein complex with Vesicle Picker, these particles can be used for template matching and training a Topaz model to obtain a larger and better centered particle stack for subsequent 3D reconstruction and refinement.

- When performing 2D classification, particularly when searching for small membrane proteins and protein complexes, we found that it is important to perform at least 40 iterations of expectation-maximization. We also typically increase the batchsize per class to 150 or 200. Finally, we almost always see better results when we disable the `Recenter 2D classes` parameter.

- We typically iterate 2D classification and selection of promising 2D classes several times. In early iterations, we enable the `Force Max over poses/shifts` parameter to efficiently classify large numbers of particles. In later iterations, where images of proteins in membranes are enriched, we typical disable the `Force Max over poses/shifts` parameter to better resolve low SNR particles within membranes.

- Sometimes, a user will already have a high quality particle stack and is interested in filtering an existing stack by selecting particles on vesicles found by vesicle picker. If this is the case, we recommend exporting a set of dense (i.e. picked with a small box size) vesicle picks to cryoSPARC and using the `Remove Duplicates` job, to intersect the two existing particle stacks. See this job's [cryoSPARC documentation](https://guide.cryosparc.com/processing-data/all-job-types-in-cryosparc/utilities/job-remove-duplicate-particles) for more details.

## Troubleshooting ##

| Failure Mode      | Suggestion      |
| ------------- | ------------- |
| Model is not detecting vesicles. | Increase the filter diameter and $\sigma_{c}$ and $\sigma_{s}$ parameters. |
| Model detects vesicles and annotates patches in the background of the image. | Decrease the filter diameter and $\sigma_{c}$ and $\sigma_{s}$ parameters. |
| Model cannot discriminate between vesicles and contaminants. | Experiment with the available filters. If vesicles are larger than contaminants on average, increase the minimum area filter. If contaminants are irregularly shaped, increase the minimum roundness filter. |

## Reference ##

Karimi, R., Coupland, C. E. & Rubinstein, J. L. Vesicle Picker: A tool for efficient identification of membrane protein complexes in vesicles. *bioRxiv* 2024.07.15.603622 (2024) doi:10.1101/2024.07.15.603622.

