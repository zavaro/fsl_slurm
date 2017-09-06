# fsl_slurm
A FSL version for use with Slurm currently in use at The University of Arizona. Includes a number of modified scripts which call on *sbatch*, a Slurm jub submission system. 

## Untested
These likely do not work at this time. Haven't had time to test them, nor does our lab commonly use them. They would require minimal changes in the case that they do fail.

* dual_regression
* feat_gm_prepare
* possumX
* qboot_parallel
* run_first_all

## GPU (Broken)
These do not work on our systems. Each binary program requires some fixing, which I will get to eventually.

* bedpostx_gpu
* bedpostx_postproc_gpu.sh
* bedpostx_preproc_gpu.sh
* bedpostx_single_part_gpu.sh
* merge_parts_gpu
* split_parts_gpu
* xfibres_gpu
