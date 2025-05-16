---
title: "Running MESA on Stampede2"
date: 2019-03-01T10:49:28-04:00
author: Matt A. Wood
---

# Steps to Running MESA on TACC Stampede2

**Matt A. Wood**  
[matt.wood@tamuc.edu](mailto:matt.wood@tamuc.edu)  
2019 March 1  

## Review the Stampede2 User Guide
[https://portal.tacc.utexas.edu/user-guides/stampede2](https://portal.tacc.utexas.edu/user-guides/stampede2)

Stampede2 mounts three Lustre file systems shared across all nodes: the home, work, and scratch file systems. The `$HOME` file system is limited to 10GB and will not be sufficient to run MESA. You will probably want to build MESA in your `$WORK` directory.

## Go to Your Scratch Space
```bash
cd $WORK
```

## Download MESA and the MESASDK
Download MESA with `svn` (update to the current release):  
```bash
svn co -r 10398 https://subversion.assembla.com/svn/mesa^mesa/trunk mesa
```

Download and untar the SDK (update to the current release):  
```bash
wget --user-agent="" http://www.astro.wisc.edu/~townsend/resource/download/mesasdk/mesasdk-x86_64-linux-20180822.tar.gz
tar xf mesasdk-x86_64-linux-20180822.tar.gz
```

## Edit Your `.bashrc` (Default Shell on Stampede2)
```bash
# for MESA
export MESA_DIR=$WORK/mesa
export MESASDK_ROOT=$WORK/mesasdk
source $MESASDK_ROOT/bin/mesasdk_init.sh
export OMP_NUM_THREADS=68
```

## Or Edit Your `.cshrc`
```csh
# for MESA
setenv MESA_DIR $WORK/mesa
setenv MESASDK_ROOT $WORK/mesasdk
source $MESASDK_ROOT/bin/mesasdk_init.csh
setenv OMP_NUM_THREADS 48
```

## Compile MESA
```bash
source ~/.bashrc
cd mesa
./install
```

## Make Your Work Directory
You will want to do this in `$WORK` or `$SCRATCH`.

## Run MESA
For small test runs you can get away with running on the login node, but you won't want to do this for production runs. 

For large runs you will want to make a batch file. Here is a sample that runs the 'tutorial' example, adapted from a TACC-provided example:  
[https://portal.tacc.utexas.edu/user-guides/stampede2#running-jobs-on-the-stampede2-compute-nodes](https://portal.tacc.utexas.edu/user-guides/stampede2#running-jobs-on-the-stampede2-compute-nodes)

### Sample Slurm Job Script
```bash
#!/bin/bash
#----------------------------------------------------
# Sample Slurm job script
#   for TACC Stampede2 KNL nodes
#
#   *** OpenMP Job on Normal Queue ***
# 
# Notes:
#
#   -- Launch this script by executing
#   -- Copy/edit this script as desired.  Launch by executing
#      "sbatch knl.openmp.slurm" on a Stampede2 login node.
#
#   -- OpenMP codes run on a single node (upper case N = 1).
#        OpenMP ignores the value of lower case n,
#        but slurm needs a plausible value to schedule the job.
#
#   -- Default value of OMP_NUM_THREADS is 1; be sure to change it!
#
#   -- Increase thread count gradually while looking for optimal setting.
#        If there is sufficient memory available, the optimal setting
#        is often 68 (1 thread per core) or 136 (2 threads per core).

#----------------------------------------------------

#SBATCH -J MESA            # Job name
#SBATCH -o myjob.o%j       # Name of stdout output file
#SBATCH -e myjob.e%j       # Name of stderr error file
#SBATCH -p skx-normal      # Queue (partition) name
#SBATCH -N 1               # Total # of nodes (must be 1 for OpenMP)
#SBATCH -n 1               # Total # of mpi tasks (should be 1 for OpenMP)
#SBATCH -t 00:10:00        # Run time (hh:mm:ss)
#SBATCH --mail-user=your.email@yourschool.edu
#SBATCH --mail-type=all    # Send email at begin and end of job

# Other commands must follow all #SBATCH directives...

module list
pwd
date

# Set thread count (default value is 1)...

export OMP_NUM_THREADS=48

# Launch OpenMP code...

./rn > output.txt

# --------------------------------------------------
```

Submit the job with:  
```bash
sbatch myjobscript
```

Check the status of your job with:  
```bash
squeue -u bjones
```
*(Replace `bjones` with your username.)*

---

**Author Note:** I'm quite new to this, so if there are errors, please let me know so I can update it.  

This writeup was adapted from Josiah Schwab's "Installing and Running MESA on a Cluster":  
[http://mesastar.org/marketplace/guides/cluster01](http://mesastar.org/marketplace/guides/cluster01)
