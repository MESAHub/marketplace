---
title: "Installing and Running MESA on a Cluster"
author: Josiah Schwabe
date: 2019-03-01T10:40:28-04:00
---

## Table of Contents
1. [Download](#download)  
  1.1 [Go to your scratch space](#go-to-your-scratch-space)  
  1.2 [Download MESA](#download-mesa)  
  1.3 [Download & untar the SDK](#download--untar-the-sdk)  
2. [Edit your .bashrc](#edit-your-bashrc)  
3. [Compile MESA](#compile-mesa)  
  3.1 [Compile the main program](#compile-the-main-program)  
  3.2 [Make your work directory](#make-your-work-directory)  
4. [Run MESA](#run-mesa)  
  4.1 [Make a batch file](#make-a-batch-file)  
  4.2 [Submit](#submit)  

## 1. Download

### 1.1 Go to your scratch space
```bash
cd /clusterfs/henyey/<username>
```

### 1.2 Download MESA
```bash
svn co -r 4631 http://mesa.svn.sourceforge.net/svnroot/mesa/trunk mesa
```

### 1.3 Download & untar the SDK
```bash
wget --user-agent="" http://www.astro.wisc.edu/~townsend/resource/download/mesasdk/mesasdk-x86_64-linux-20121106.tar.gz
tar xf mesasdk-x86_64-linux-20121106.tar.gz
```

## 2. Edit your .bashrc
```bash
export SCRATCH="/clusterfs/henyey/jwschwab/"

# for MESA
export MESA_DIR=$SCRATCH/mesa
export MESASDK_ROOT=$SCRATCH/mesasdk
source $MESASDK_ROOT/bin/mesasdk_init.sh
```

## 3. Compile MESA

### 3.1 Compile the main program
Make sure to either source your .bashrc or log out and back in. Now running the `./install` command should work.

### 3.2 Make your work directory
As usual, make a copy of the work directory. Since you may need a decent amount of space for the logs, this should probably live on your scratch space. Once you have that set up the way you normally would (run `./mk`, set up your inlist, etc).

## 4. Run MESA

### 4.1 Make a batch file
On one of these clusters, the way you run a job is by submitting one to the batch queue system. There's specific info about this on the Henyey bSpace page. In general, you need a batch file that describes the job you want to run. My MESA ones look like this:

```bash
#!/bin/bash
#PBS -N MESA
#PBS -M <email_address>
#PBS -m abe
#PBS -l nodes=1:ppn=8,walltime=10:00:00
#PBS -q henyey_batch
#PBS -V
#
cd $PBS_O_WORKDIR

#
# this should be equal to the value of ppn
# there are 8 cores on each henyey node
#

export OMP_NUM_THREADS=8

./rn > rn.out
```

Things that you might want to adjust are the job name (`-N`) and the job time limit (`walltime`). The `-m` & `-M` flags have the system send you email when your jobs start/stop. And then of course, you may want to do something different with your `./rn` command, etc.

### 4.2 Submit
Supposing you named your batch file `runmesa.sh` then:
```bash
qsub runmesa.sh
```

Then you can see where you are in the queue by typing:
```bash
qstat
```
