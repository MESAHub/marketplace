---
title: "Structuring inlists for running many MESA models"
date: 2017-01-30T10:49:28-04:00
author: Rob Farmer
draft: false
---

## This is a guide for how to set up inlists when you wish to run many (100's, maybe more) MESA models.

## Introduction

This guide is designed for people wanting to run many MESA models on a cluster, perhaps a parameter study. If you are interested in single runs of MESA on a cluster see Josiah's guide. This guide is aimed at simplifying the management of the inlists needed for running hundreds/thousands of MESA models. For this I'll use an example of running 100 models where we only vary the `initial_mass`.

## Basic setup

Firstly install MESA as normal on your cluster, somewhere where all the compute nodes can see, so I have MESA in the directory `$MESA_DIR`.

Next create a new work directory, which I'll call `MESA_BASE`. You won't actually run anything from this directory, however it's used as a central repository for the actual MESA runs to read information from.

## Inlists

Next alter the inlists inside `$MESA_BASE`, these should have ONLY the parameters that DO NOT change between your models, so for this example don't set the `initial_mass` but perhaps you want to set the `initial_z` or the opacity tables etc. or maybe you want some pgplots saved. Also set the `history_columns` and `profile_columns` for your required output.

Next alter the main inlist file (`$MESA_BASE/inlist`). You want to set the `history_columns_file`, `profile_columns_file` and the `*_inlist1` terms to be the ABSOLUTE path to those files. For me I have the files in:

```fortran
&star_job
 history_columns_file ='/home/rjfarmer/base/history_columns.list'
 profile_columns_file ='/home/rjfarmer/base/profile_columns.list'

 read_extra_star_job_inlist1 =.true.
 extra_star_job_inlist1_name ='/home/rjfarmer/base/inlist_star' 

 read_extra_star_job_inlist2 = .true.
 extra_star_job_inlist2_name ='inlist_cluster' 

/ ! end of star_job namelist

&controls
 read_extra_controls_inlist1 =.true.
 extra_controls_inlist1_name ='/home/rjfarmer/base/inlist_control'

 read_extra_controls_inlist2 =.true.
 extra_controls_inlist2_name ='inlist_cluster'     

/ ! end of controls namelist

&pgstar
 read_extra_pgstar_inlist1 =.true.
 extra_pgstar_inlist1_name ='/home/rjfarmer/base/inlist_pgstar'

/ ! end of pgstar namelist
```

Note the second star and control inlists, these will contain the parameters that vary between runs, in this case the `initial_mass`. This will be explained later.

Finally compile MESA in `$MESA_BASE`.

## Cluster inlists

Next create the output folders, in bash this could be done with:

```bash
for j in $(seq 1 100);do mkdir $j;done
```

in the location you want to run MESA in, which I'll call `$MESA_RUN`.

Then in each folder (`$MESA_RUN/1` for instance) add a new file called `inlist_cluster`, with the information that varies with each run. Exercise is left for the reader to work out how to script this step, however a sample can be found here: [https://github.com/rjfarmer/multimesa](https://github.com/rjfarmer/multimesa). So for this example the file might look like this (`$MESA_RUN/1/inlist_cluster`):

```fortran
&star_job  

/ ! end of star_job namelist

&controls
 initial_mass= 1.0     

/ ! end of controls namelist
```

You don't need to worry about making any directories like the `LOGS` or photos etc., as of MESA version 6794, they will be made for you. So in the folder `$MESA_RUN/1` there should only be 1 file, `inlist_cluster`.

```bash
ls $MESA_RUN/1
inlist_cluster
```

And that is all you need in each of the `$MESA_RUN` folders.

## Submission script

Now here is the submission script (`cluster.sh`), note this may vary depending on your cluster software:

```bash
#!/bin/bash
#PBS -N mesaMass
#PBS -M you@email.com
#PBS -m abe
#PBS -V
#PBS -l nodes=1:ppn=1
#PBS -l walltime=168:00:00
#PBS -j oe
#PBS -t 1-100
#Set variables

export MESA_DIR="/home/rjfarmer/cluster"
export OMP_NUM_THREADS=1
export MESA_BASE="/home/rjfarmer/base"
export MESA_INLIST="$MESA_BASE/inlist"
export MESA_RUN="/home/rjfarmer/runs"

#CD to folder
cd $MESA_RUN/$PBS_ARRAYID

#Run MESA
$MESA_CLUSTER_BASE/star
```

So most of the first lines with `#PBS` are set up parameters for the cluster. The important one for this work is the:

```bash
#PBS -t 1-100
```

This sets up a job array (Google for "job array" if you have a different software setup). What happens is when I submit this script:

```bash
qsub cluster.sh
```

I will submit one job but the cluster will create 100 jobs numbered between 1 and 100 and set that number in `$PBS_ARRAYID`. This variable is then used to determine which folder to change directory to and then run MESA in. So I end up with 100 jobs running in different folders, where each folder has a different `initial_mass` set, but they all run with the same base configuration which is set in `$MESA_BASE`.

Note `$MESA_INLIST` variable only exists in MESA as of version 6955. This is used to redirect the main inlist MESA reads, from trying to read the file `inlist` in the current directory to the one in `$MESA_BASE`.

If you are using an earlier version of MESA, then a workaround is to create a soft link to the inlist file into the run folder.

```bash
for j in $(seq 1 100);do ln -s $MESA_BASE/inlist $MESA_RUN/$j/inlist;done
```

## Why?

Why do it this way? It seems like we're making lots of extra work with this `$MESA_BASE` folder. Well, imagine you copied the whole inlist into each work directory. But now you decided you need to change a parameter or you need extra output in the history file, you've now got 100 files to alter. Or you do it this way you only have 1 file to alter.

Also imagine you decide to put something in the `run_stars_extra.f`, previously you'd have to make sure all folders recompiled properly. This way you only recompile MESA in 1 folder in `$MESA_BASE` and all the runs will end up using the new version, without you having to worry that some runs might run with an old version.

Why set `OMP_NUM_THREADS=1`? When MESA runs faster with a higher number? Because clusters are usually limited in the number of cores they have (or you can use at once). So by setting this to 1 we can run more jobs at once (though each job is slower). But it means we effectively get to parallelize the serial parts of MESA. Thus overall to run all 100 jobs with one thread each, should finish quicker than running a few jobs at a time but where each has a higher `OMP_NUM_THREADS`.

## Other hints and advice

- Be careful with the amount of output, if you've ever filled your hard drive with one MESA run imagine what happens when you run 100 jobs.
- The terminal output from MESA will usually be redirected to a file (see your cluster documentation for what this file is called) so we don't need to worry about redirecting it ourselves. In fact, I just turn it off with:

```fortran
terminal_interval = 1000000
```

As all we need should be in the `LOGS` folder.
