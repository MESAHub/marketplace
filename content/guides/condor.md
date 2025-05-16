---
title: "Running MESA on Condor"
date: 2017-02-16T10:49:28-04:00
author: Earl Bellinger
---

Active condor user here. I wrote a script called `maybe_sub.sh` that submits jobs to condor to run in parallel if condor is available (hence 'maybe'). You can see it here:  
[https://github.com/earlbellinger/asteroseismology/blob/master/scripts/maybe_sub.sh](https://github.com/earlbellinger/asteroseismology/blob/master/scripts/maybe_sub.sh)

You can do something like:

```bash
maybe_sub.sh -p 8 echo hi
```

which will execute `echo hi` using 8 processors on the cluster. In addition to running the program on the cluster as normal, it will create a directory called `maybe_sub_logs` which inside details the outputs of the calculation ("hi"). You can run `maybe_sub.sh -h` to see all the options.

*I personally drop this file in a `~/scripts` directory, make it executable (`chmod +x maybe_sub.sh`), and add `~/scripts` to my `$PATH` variable in my `.bashrc` file so that I can call it from anywhere.*

My workflow is to write a simple bash script for setting up my MESA job and then use `maybe_sub.sh` to execute that bash script optionally in parallel using the `-p` argument to specify the number of processors. The trick with MESA+condor is to send along the `$OMP_NUM_THREADS` variable to condor.

For example, in our recent paper [1], I made a quasirandom grid of main-sequence solar-like oscillators. For each combination of mass, metallicity, etc., I called a bash script called `dispatch.sh` which can be seen here:  
[https://github.com/earlbellinger/asteroseismology/blob/master/forward/dispatch.sh](https://github.com/earlbellinger/asteroseismology/blob/master/forward/dispatch.sh)

On the compute nodes, it copies over my "mesa template," changes the inlist settings to the desired properties, and runs the track in parallel. This would be called, for example, with:

```bash
maybe_sub.sh -p 4 dispatch.sh -M 1.2 -Z 0.001
```

to run a track with a mass of 1.2 and a metallicity of 0.001 using 4 processors.

To make a grid, I have a Python program called `sobol_dispatcher.py`:  
[https://github.com/earlbellinger/asteroseismology/blob/master/forward/sobol_dispatcher.py](https://github.com/earlbellinger/asteroseismology/blob/master/forward/sobol_dispatcher.py)

which calls `maybe_sub.sh` on `dispatch.sh` with varied arguments, with something like:

```python
bash_cmd = "maybe_sub.sh -p 2 ./dispatch.sh -M 0.9"
subprocess.Popen(bash_cmd.split(), shell=False)
```

You can write your own script (which I personally find good to do for each individual project at hand) or link up `maybe_sub.sh` to one of the existing MESA scripts.

Hope this is helpful.

[1] Earl P. Bellinger & George C. Angelou et al., "Fundamental Parameters of Main-Sequence Stars in an Instant with Machine Learning," 2016 ApJ 830 31,  
[http://adsabs.harvard.edu/abs/2016ApJ...830...31B](http://adsabs.harvard.edu/abs/2016ApJ...830...31B)

Best regards,  
Earl Bellinger
