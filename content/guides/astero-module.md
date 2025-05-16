---
title: "Overview of the `astero` module"
date: 2016-09-05T10:49:28-04:00
author: Warrick Ball
draft: false
---

# The `astero` module

The `astero` module serves two basic functions in MESA.

1. It links to two stellar oscillation packages to compute stellar oscillation frequencies from inside MESA.

2. It implements algorithms to search for model parameters that best fit some observations (seismic or not).

By design, the `astero` module doesn't do much of the first task for you automatically. Instead, it provides hooks that you can use in `run_star_extras`. So if you want to compute a grid of models with oscillation frequencies, you can call these routines according to your needs.

The `astero/work` directory presumes that you wish to perform the second task and most of this tutorial is about how to use MESA and `astero_search_controls` to fit models to data. It's strongly recommended that you try running the example in work and make sense of what the inputs and outputs are. A brief description of the example is at the end of this document.

## Calculating stellar oscillations
MESA provides interfaces to two stellar oscillation codes: ADIPLS and GYRE. The subroutines for calling these codes is found in `star/astero/src`. So, for example, `adipls_support.f` has various subroutines for calling ADIPLS to compute frequencies. For examples of how this is done, have a look in `astero_support.f`. To use these routines in `run_star_extras`, use the relevant module in your own subroutine.

### ADIPLS
ADIPLS is well-established code, written and maintained by Jørgen Christensen-Dalsgaard. It computes linear, adiabatic oscillation frequencies using one of a second-order shooting method, fourth-order shooting method or relaxation method, with or without Richardson extrapolation. Detailed documentation is bundled with MESA in two files, both found in `$MESA_DIR/adipls/adipack.c/notes`:

* `adiab.prg.c.pdf` describes the details of the oscillation calculation, including the control file, `adipls.c.in`.

* `operation.ps` describes how to use several utility scripts that convert between binary and ASCII formats and remesh the model. If you're only using ADIPLS from inside MESA, you generally don't need to worry about these.

### GYRE
GYRE is a new, open-source code, written and maintained by Rich Townsend. GYRE computes linear adiabatic or non-adiabatic oscillation frequencies using a parallelized multiple Magnusson shooting scheme. Like ADIPLS, tThere are multiple options for the formulation of the equations, boundary conditions, order of the integration method, and more. The documentation is online at the [GYRE website](https://bitbucket.org/rhdtownsend/gyre/wiki/Home).

## Stellar modelling as an optimization problem
To understand the second task, let's briefly consider a very general problem. Suppose that I have a magical black box that produces some numbers *y* given some input numbers *x*. In other words, I have some function *y(x)* but not its inverse: I can't run the box backwards. Now, suppose that some other kindly soul has gone out and measured some values of *y*. How do I decide which values of *x* reproduce *y*, using my black box? As you can imagine, this is a common problem in science and engineering, known variously as *optimization*, *parameter estimation* or occasionally *inversion* (though the latter usually refers to the case where *x* is a continuous variable and *y(x)* a function, rather than a discrete set of paremeters and observations).

Here, our magical black box is MESA (although it's unlocked, so you can look inside yourself!) and the observations *y* it produces are anything about a star that can be observed: effective temperature, surface gravity, metallicity, radius, mass, mode frequencies, whatever. The parameters *x* are the numbers we need to make a stellar model, like mass, age, mixing-length, and composition. The astero module implements some standard algorithms that optimize the stellar model, given observational constraints. The user can choose which parameters to vary and which to fix. You can also constrain the helium abundance and metallicity to vary together according to some enrichment law.

Although we're discussing the `astero` module, this process doesn't have to involve seismology. We can try to fit a stellar model even if we don't have any frequency information because we may also have luminosities, interferometric radii or even dynamical masses. In fact, this is precisely what MESA does for the Sun. This distinguishes MESA's current solar calibration from the standard procedure. Usually, one fixes the mass and age of the Sun and fits the initial metallicity, initial helium and mixing length to the observed luminosity, radius and metallicity. MESA uses other information, including, for example, a direct fit to the rms variation of the sound speed relative to a reference profile.

### Age
MESA capitalizes on the special status of the stellar age. It's special because to have a model at a particular age, you must have the model at the previous age, and the age before that, etc., going all the way back to your initial model. The evaluation of one more model along the current evolutionary track is inexpensive, so it makes sense to optimize the age for every set of trial parameters. Thus, you never need to specify that you want to optimize the age. Rather, you have to specify if you only want to evaluate models at a particular age, as is the case in the solar calibration.

### Optimization algorithms
MESA has several optimization algorithms available. They're described here very briefly, so it's recommended that you look up these methods to know exactly what MESA is doing.

* **Downhill simplex**. Constructs a simplex and reflects, expands and contracts it towards a minimum. This is a standard method for functions where reliable derivatives are unavailable. If you don't know where to start, you could do worse than a downhill simplex. Also known as the Nelder--Mead method, and implemented as `amoeba` in IDL and `fminsearch` in MATLAB.

* **NEWUOA and BOBYQA**. Roughly speaking, these two methods approximate the shape of the objective function using quadratic approximation and then use the minimum of the approximation as the next guess for the best-fitting model. These methods are slightly more prone to getting stuck in local minima, so are better-suited to improving a good fit. BOBYQA is a bounded method, so it respects the maximum and minimum values you provide.

### User-defined searches
MESA can also search for the best-fitting model based on user-supplied data.

* **Grid search.** Evaluates all models at regular intervals in a user-specified range of parameter space.

* **Input file.** Evaluates models with parameters given in a file. Useful for re-evaulating models from other runs to get extra output.

* **No search/optimization.** Just runs the user-specified initial parameters. Important during testing, so that you don't lose time to an improperly formulated run. Should also be used if no optimization is desired, but remember to disable the model-fitting specifics like stop conditions and timestep adaptation.

### User-defined observables
MESA allows you to define up to three of your own variables. These are denoted `my_var1`, `my_var2` and `my_var3`, and included just like other observables. Their values must be defined in `run_star_extras.f`. An example of your own variables might be the acoustic depth of the base of the convection zone measured from the acoustic glitch.

## Model inputs
Although astero_search_controls provides detailed parameters for the input data, initial guess and progress of the optimization, all the details about the stellar model (e.g. formulation of mixing-length theory, opacity tables, etc.) are still contained in the usual `inlist`. Some of these (e.g. the model mass) will be overwritten by values in astero_search_controls, but most won't. So don't forget to check the usual `inlist` to make sure you're fitting a model with the physics you want!

### Basic overview of `astero_search_controls` structure
In a nutshell, `astero_search_controls` contains everything we need to set up the optimization problem. The components are described in more detail elsewhere, but the overall structure is given below. Note that there are some other controls that either don't fit into these categories or aren't quite in the order given here.

* **Observational data.** These are the numbers we're trying to reproduce. There are several standard options, as well as room to add up to three of your own variables, defined in `run_star_extras.f.` The constraints are separated into seismic and non-seismic and there is an additional control for the relative weights of the two.

* **Fitting algorithm.** These controls select which optimization algorithm is used and set parameters specific to those methods.

* **Initial guess and variable parameters.** All optimization algorithms require an initial guess. In addition, one must specify which parameters are to be varied and, if using a bounded method, the bounds of the parameter ranges.

* **Partial evaluation controls.** MESA performs some adaptive tricks to avoid calculating everything that's necessary until it's already close to a good fit. For example, computing mode frequencies is more expensive than non-seismic inputs, and non-radial mode frequencies are more expensive that radial mode frequencies.

* **Stopping conditions.** A particular run can be terminated when the fit is becoming much worse or when the stellar model moves outside the bounds of the observations.

* **Controls for the oscillation codes.** The oscillation codes have some options which can be set in MESA. There are always many more that are set in the separate input file, which is specified in this section.

* **Output.** MESA provides options for what output to store.

* **Other inlists.** Finally, like the normal MESA inlist, there is the option to include other inlists. This is very useful if you're performing fits for many stars and want some parameters to remain fixed.

## `star/astero/work` walkthrough
To concretize this material, let's have a look at the example contained in the `star/astero/work` folder. These are observable details for the CoRoT target HD49385. Not all the controls are represented here, so be sure to look at the `star/astero/defaults/astero_search_controls.defaults` too.

### Observational data
```fortran
chi2_seismo_fraction = 0.667d0
```

This first option specifies that the total reduced chi-squared is the some of one-third the non-seismic component and two-thirds the seismic component.

```fortran
include_Teff_in_chi2_spectro = .true.
Teff_target = 6095d0
Teff_sigma = 65d0

include_logg_in_chi2_spectro = .false.
logg_target = 3.97d0
logg_sigma = 0.02d0

include_logL_in_chi2_spectro = .true.
logL_target = 0.67d0
logL_sigma = 0.05d0

include_FeH_in_chi2_spectro = .true. ! [Fe/H]
FeH_target = 0.09
FeH_sigma = 0.05

    Z_div_X_solar = 0.02293d0

include_logR_in_chi2_spectro = .false.
logR_target = 0
logR_sigma = 1d-4
```

The list of observational starts with "classical" values. In this case, we have constraints Teff=6095+-65K, logg=3.97+-0.02, logL=0.67+-0.05 and [Fe/H]=0.09+-0.05, though the logg constraint is not being used. We must define the solar metallicity to make sense of [Fe/H]. This should be the same as the composition of your stellar model! The option for a radius (e.g. from interferometry) is available but not used here. Keep in mind that, because of the surface boundary condition L=4\pi R^2\sigma Teff^4, constraining R, L and Teff is degenerate.

```fortran
include_surface_Z_div_X_in_chi2_spectro = .false.
surface_Z_div_X_target = 2.292d-2 ! GS98 value
!surface_Z_div_X_target = 1.81d-2 ! Asplund 09 value
surface_Z_div_X_sigma = 1d-3

include_surface_He_in_chi2_spectro = .false.
surface_He_target = 0.2485d0 ! Bahcall, Serenelli, Basu, 2005
surface_He_sigma = 0.0034

include_age_in_chi2_spectro = .false.
age_target = 4.5695d9 ! (see Bahcall, Serenelli, and Basu, 2006)
age_sigma = 0.0065d9

   num_smaller_steps_before_age_target = 50 ! only used if > 0
   dt_for_smaller_steps_before_age_target = 0.0065d8 ! 1/10 age_sigma
      ! this should be << age_sigma

include_Rcz_in_chi2_spectro = .false. ! radius of base of convective zone
Rcz_target = 0.713d0 ! Bahcall, Serenelli, Basu, 2005
Rcz_sigma = 1d-3

include_csound_rms_in_chi2_spectro = .false. ! check sound profile
csound_rms_target = 0
csound_rms_sigma = 2d-6
report_csound_rms = .false.
```

We then have a series of non-seismic constraints that are principally used in the solar calibration, including a raw value of Z/X, the surface helium abundance, the age target (and controls for meeting it), the depth of the convection zone and the rms difference against a reference sound speed profile.

```
include_my_var1_in_chi2_spectro = .false.
my_var1_target = 0
my_var1_sigma = 0
my_var1_name = 'my_var1'

include_my_var2_in_chi2_spectro = .false.
my_var2_target = 0
my_var2_sigma = 0
my_var2_name = 'my_var2'

include_my_var3_in_chi2_spectro = .false.
my_var3_target = 0
my_var3_sigma = 0
my_var3_name = 'my_var3'
```

These controls include user-defined variables. They are computed in `run_star_extras.f`.

```
chi2_seismo_delta_nu_fraction = 0d0  
chi2_seismo_nu_max_fraction = 0d0    
chi2_seismo_r_010_fraction = 0d0
chi2_seismo_r_02_fraction = 0d0
```

We now specify how much of the seismic reduced chi-squared comes from the large separation ($\Delta\nu$), the frequency of maximum oscillation power ($\nu_\text{max}$) and the frequency ratios. The remainder of the seismic reduced chi-squared comes from the individual mode frequencies.

```
trace_chi2_seismo_delta_nu_info = .false. ! if true, output info to terminal
trace_chi2_seismo_nu_max_info = .false. ! if true, output info to terminal
trace_chi2_seismo_ratios_info = .false. ! if true, output info to terminal
trace_chi2_seismo_frequencies_info = .false. ! if true, output info to terminal
trace_chi2_spectro_info = .false. ! if true, output info to terminal
```

These controls provide terminal output about the reduced chi-squared calculation.

```
nu_max = 1010
nu_max_sigma = -1

delta_nu = 56.28
delta_nu_sigma = 1.0
```

We now set the overall seismic properties. Note that `delta_nu` and `delta_nu_sigma` must always be set, even when `chi2_seismo_delta_nu_fraction` is 0. This is because the asymptotic value of delta_nu can be evaluated without computing the mode frequencies.

If `delta_nu` in the inlist is positive, then the code uses values for both `delta_nu` and `delta_nu_sigma` from the inlist. If `delta_nu` is negative in the inlist, the code estimates it by linear fit to the observed radial frequencies and orders, `l0_obs` and `l0_n_obs`. If `delta_nu_sigma` from the inlist is also negative, then the code also sets it by using the radial data.

```fortran
   nl0 = 9
   l0_obs(1) = 799.70d0
   l0_obs(2) = 855.30d0
   l0_obs(3) = 909.92d0
   l0_obs(4) = 965.16d0
   l0_obs(5) = 1021.81d0
   l0_obs(6) = 1078.97d0
   l0_obs(7) = 1135.32d0
   l0_obs(8) = 1192.12d0
   l0_obs(9) = 1250.12d0
   l0_obs_sigma(1) = 0.27d0
   l0_obs_sigma(2) = 0.73d0
   l0_obs_sigma(3) = 0.26d0
   l0_obs_sigma(4) = 0.36d0
   l0_obs_sigma(5) = 0.28d0
   l0_obs_sigma(6) = 0.33d0
   l0_obs_sigma(7) = 0.34d0
   l0_obs_sigma(8) = 0.45d0
   l0_obs_sigma(9) = 0.89d0

   nl1 = 10
   l1_obs(1) = 748.60d0
   l1_obs(2) = 777.91d0
   l1_obs(3) = 828.21d0
   l1_obs(4) = 881.29d0
   l1_obs(5) = 935.90d0
   l1_obs(6) = 991.09d0
   l1_obs(7) = 1047.79d0
   l1_obs(8) = 1104.68d0
   l1_obs(9) = 1161.27d0
   l1_obs(10) = 1216.95d0
   l1_obs_sigma(1) = 0.23d0
   l1_obs_sigma(2) = 0.24d0
   l1_obs_sigma(3) = 0.42d0
   l1_obs_sigma(4) = 0.29d0
   l1_obs_sigma(5) = 0.23d0
   l1_obs_sigma(6) = 0.22d0
   l1_obs_sigma(7) = 0.24d0
   l1_obs_sigma(8) = 0.22d0
   l1_obs_sigma(9) = 0.33d0
   l1_obs_sigma(10) = 0.53d0

   nl2 = 8
   l2_obs(1) = 794.55d0
   l2_obs(2) = 905.31d0
   l2_obs(3) = 961.47d0
   l2_obs(4) = 1017.56d0
   l2_obs(5) = 1075.01d0
   l2_obs(6) = 1130.79d0
   l2_obs(7) = 1187.55d0
   l2_obs(8) = 1246.78d0
   l2_obs_sigma(1) = 0.52d0
   l2_obs_sigma(2) = 0.35d0
   l2_obs_sigma(3) = 0.49d0
   l2_obs_sigma(4) = 0.27d0
   l2_obs_sigma(5) = 0.27d0
   l2_obs_sigma(6) = 0.61d0
   l2_obs_sigma(7) = 0.32d0
   l2_obs_sigma(8) = 0.84d0

   nl3 = 0 ! number of observed l=3 modes
   l3_obs(:) = 0 ! frequencies. set l3_obs(1), l3_obs(2) .... l3_obs(nl3)   
   l3_obs_sigma(:) = 0 ! l3_obs_sigma(i) is uncertainty for l3_obs(i), for i=1,nl3
```

Finally, we provide the observed frequency data. The number of observed modes of each degree must be provided, but the radial orders are not needed. MESA matches the radial orders on its own. Even so, the order of the radial orders can be set but they are used for other purposes.

### Search controls

```fortran
eval_chi2_at_target_age_only = .false.
```

True if you only want to evaluate models at a particular target age.

```fortran
min_age_for_chi2 = -1 ! (years) only use if > 0
max_age_for_chi2 = -1 ! (years) only use if > 0
```

This confines the search to a range of ages. Chi-squared is not evaluated if the age is not within these age bounds.

```fortran
search_type = 'use_first_values'
```

This run just uses the first values, since we don't want an example that takes hours upon hours in its default mode. The other options are commented out. Each generally has some tolerance control, a choice of whether or not to resume from a file and a handful of other specific controls.

### Choice of parameters

```fortran
   Y_depends_on_Z = .false. 
   Y0 = 0.248d0
   dYdZ = 1.4d0
```

You may specify that Y varies with Z according do an enrichment law of the form Y = Y0 + dYdZ*Z. If you do so, set `vary_Y = .false.` below.

```fortran
   vary_FeH = .true. ! FeH = [Fe/H] = log10((Z/X)/Z_div_X_solar)
   vary_Y = .true. ! Y the initial uniform value
   vary_mass = .true. ! initial mass
   vary_alpha = .true.
   vary_f_ov = .false.
```

These controls specify which parameters are free to vary. They are the initial metallicity, initial helium abundance, mass, mixing-length parameters alpha and overshooting parameter `f_ov`. Composition parameters correspond to initial uniform values. The overshooting parameters sets *all* the overshooting coefficients (i.e. above and below all zones, burning or not) to the next trial value.

```fortran
   first_FeH = -0.0118938
   first_Y = 0.23765625
   first_mass = 1.31093750
   first_alpha = 1.5875
   first_f_ov = 0.015

   min_FeH = -0.012012738
   min_Y = 0.2352796875
   min_mass = 1.297828125
   min_alpha = 1.571625

   max_FeH = -0.011774862
   max_Y = 0.2400328125
   max_mass = 1.324046875
   max_alpha = 1.603375
```

The `first`, `min` and `max` parameters for each parameter control the initial guess, used for the initialization of the optimization, and the bounds, for those methods that are bounded. In addition, the `simplex` method uses the bounds to determine the size of the initial simplex.

```fortran
   delta_FeH = 0.03
   delta_Y = 0.01
   delta_mass = 0.01
   delta_alpha = 0.1
   delta_f_ov = 0
```

These controls specify the spacing in the grid when using the `grid_search` method.

### Partial evaluation controls

```fortran
! calculating mode frequencies is a relatively costly process,
! so we don't want to do it for models that are not good candidates.
! i.e., we want to filter out the bad candidates using the following
! less expensive tests whenever possible.

   ! NOTE: if none of the models in a run pass these tests,
   ! then you will not get any total chi2 result for that run.
   ! in some situations that might not matter,
   ! but if you are eliminating too many candidates in this way,
   ! the search routines might not be getting enough valid results to work properly.
   ! So watch what you are doing!  If your search or scan is getting lots of
   ! runs that fail to give chi^2 results, you'll need to adjust the limits.

      min_age_limit = 1d6
      Lnuc_div_L_limit = 0.95 ! this rules out pre-zams models
```

These controls prevent the code from considering pre-main-sequence models.

```fortran
      chi2_spectroscopic_limit = 20
      chi2_delta_nu_limit = 20
``` 
Models are only considered once the non-seismic reduced chi-squared is less than `chi2_spectroscopic_limit` and the chi-squared contribution of the asymptotic large separation is within `chi2_delta_nu_limit`. Note that this means `delta_nu` and `delta_nu_sigma` must be set, even if `delta_nu` is not included in chi-squared.

If these conditions are met

```fortran
   ! we calculate radial modes only if pass the previous checks

   ! calculating nonradial modes is much more expensive than radial ones.
   ! so we skip the nonradial calculation if the radial results are poor.

   ! don't consider models with chi2_radial above this limit 
      chi2_radial_limit = 30
```

If the previous conditions are met, the radial mode frequencies are computed. If the chi-squared contribution of the radial modes is less than `chi2_radial_limit`, then all the mode frequencies are computed and chi-squared is evaluated in full.

### Chi-squared based timestep control

```fortran
max_yrs_dt_when_cold = 1d7 ! when fail Lnuc/L, chi2_spectro, or ch2_delta_nu
max_yrs_dt_when_warm = 1d6 ! when pass previous but fail chi2_radial; < max_yrs_dt_when_cold
max_yrs_dt_when_hot = 1d5 ! when pass chi2_radial; < max_yrs_dt_when_warm

max_yrs_dt_chi2_small_limit = 5d4 ! < max_yrs_dt_when_hot
chi2_limit_for_small_timesteps = 30

max_yrs_dt_chi2_smaller_limit = 5d4 ! < max_yrs_dt_chi2_small_limit
chi2_limit_for_smaller_timesteps = -1d99 ! < chi2_limit_for_small_timesteps

max_yrs_dt_chi2_smallest_limit = 3d4 ! < max_yrs_dt_chi2_smaller_limit
chi2_limit_for_smallest_timesteps = -1d99 ! < chi2_limit_for_smaller_timesteps
```

These controls set the maximum timestep based on the current value of chi-squared. That is, as chi-squared becomes smaller, the timestep can be reduced. This is a balancing act between making sure that the age is resolved finely enough to find a good chi-squared but not so finely that a lot of time is spent calculating everything even though it isn't improving the fit much.

### Stopping conditions

```fortran
sigmas_coeff_for_logg_limit = -5 ! disable by setting to 0
sigmas_coeff_for_logL_limit = 5 ! disable by setting to 0
sigmas_coeff_for_Teff_limit = -5 ! disable by setting to 0
sigmas_coeff_for_logR_limit = 0 ! disable by setting to 0
sigmas_coeff_for_surface_Z_div_X_limit = 0 ! disable by setting to 0
sigmas_coeff_for_surface_He_limit = 0 ! disable by setting to 0
sigmas_coeff_for_Rcz_limit = 0 ! disable by setting to 0
sigmas_coeff_for_csound_rms_limit = 0 ! disable by setting to 0
sigmas_coeff_for_delta_nu_limit = 0 ! -5 ! disable by setting to 0
sigmas_coeff_for_csound_rms_limit = 0 ! disable by setting to 0
sigmas_coeff_for_my_var1_limit = 0 ! disable by setting to 0
sigmas_coeff_for_my_var2_limit = 0 ! disable by setting to 0
sigmas_coeff_for_my_var3_limit = 0 ! disable by setting to 0
```

These controls stop the current run when the given observable differs from its observed value by `sigmas_coeff_for_observable_limit`. They are only used if non-zero. If negative, the limit is a lower limit; if positive, as an upper limit.

```fortran
chi2_relative_increase_limit = 2.0
limit_num_chi2_too_big = 3

   ! and here is an absolute limit
   chi2_search_limit1 = 3.0
   chi2_search_limit2 = 4.0
   ! if best chi2 for the run is <= chi2_search_limit1,
   ! then stop the run if chi2 >= chi2_search_limit2.
```

The run is halted if `limit_num_chi2_too_big` consecutive chi-squareds are more than `chi2_relative_increase_limit` times the best chi2 for the run.

```fortran
! if you are doing a search or scanning a grid, you can use previous results
!    as a guide for when to stop a run

   min_num_samples_for_avg = 2 ! want at least this many samples to form averages
   max_num_samples_for_avg = 10 ! use this many of the best chi^2 samples for averages
   avg_age_sigma_limit = 5 ! stop if age > avg age + this limit times sigma of avg age
   avg_model_number_sigma_limit = 5 ! ditto for model number
```

These controls allow you to halt the run if the age is getting much larger than that of some average of the best-fitting models so far. Specifically, the means and standard deviations of the age and model number of at least min_num_samples_for_avg and at most `max_num_samples_for_avg` are computed. If the age or model number are greater by more than `avg_age_sigma_limit` or `avg_model_number_sigma_limit`, the run is halted. The idea here is that if the 10 best-fitting models have an average age of 1+-0.1 Gyr, it doesn't make sense to follow a model past, say, 1.5 Gyr.

### Surface corrections
```fortran
! surface corrections (see K08 -- ApJ 683, pg 175)
correction_factor = 1
! use this fraction of the correction; set to 0 to skip doing corrections.
l0_n_obs(:) = -1 ! the observed radial orders (ignored if < 0)
! the observed radial orders are used in calculating surface corrections
! if <= 0, use default calculation for radial orders
correction_b = 4.25d0
```

### Output controls

```fortran
write_best_model_data_for_each_sample = .true.
num_digits = 4 ! number of digits in sample number (with leading 0's)
sample_results_prefix = 'outputs/sample_' 
   ! note that you can include a directory in the prefix if desired
sample_results_postfix = '.data'

model_num_digits = 4 ! number of digits in model number (with leading 0's)

write_fgong_for_each_model = .false.
fgong_prefix = 'fgong_' 
   ! note that you can include a directory in the prefix if desired
fgong_postfix = '.data'

write_fgong_for_best_model = .false.
best_model_fgong_filename = ''

write_gyre_for_each_model = .false.
gyre_prefix = 'gyre_' 
   ! note that you can include a directory in the prefix if desired
gyre_postfix = '.data'
max_num_gyre_points = -1 ! only used if > 1

write_gyre_for_best_model = .false.
best_model_gyre_filename = ''

write_profile_for_best_model = .false.
best_model_profile_filename = ''

save_model_for_best_model = .false.
best_model_save_model_filename = ''

save_info_for_last_model = .false. ! if true, treat final model as "best"
last_model_save_info_filename = '' ! and save info about final model to this file.
```

The controls for recording models and profiles are self-explanatory. Options include writing model files, profiles, and inputs for ADIPLS and GYRE (if you wish to run those codes manually later). The `best_model_data` is a short summary of the parameters of the best model in that run, similar to output given in the terminal.

The `best` model is always the best model of *the most recent run*. Thus, at the end of the optimization, these models might not correspond to the optimal model (though they should be close). You can save the best models from each run using the scripting option below.

```fortran
shell_script_for_each_sample = '' ! executed after at end of sample run
shell_script_num_string_char = '#' ! replace by num string for sample
```

The shell script specified by `shell_script_for_each_sample` will be evaluated after a run, with `shell_script_num_string_char` substituted by the current sample number. This script can be a single shell script or a sequence of commands, as long as it works properly as a one-liner. e.g. `cp best.mod outputs/sample#_best.mod; cp LOGS/history.data outputs/sample#_history.data`

### Miscellaneous

```fortran
! save all control settings to file
   save_controls = .false. ! dumps &astero_search_controls controls to file
   save_controls_filename = '' ! if empty, uses a default name
```

Saves the control settings (i.e. `&astero_search_controls`) to the specified file.

```fortran
   Y_frac_he3 = 1d-4 ! = xhe3/(xhe3 + xhe4); Y = xhe3 + xhe4
```

Specifies the fractional abundance of helium-3 relative to helium-4.

### Controls for oscillation codes.

```fortran
   save_mode_model_number = 0
   save_mode_filename = ''
   el_to_save = 0
   order_to_save = 0
```

You can use these controls to save an eigenfunction. If you want a more output, you can compute eigenfunction yourself using the model output and oscillation codes.

```fortran
   add_atmosphere = .false.
```

If true, then the atmosphere is added to the stellar model before it is passed to the oscillation code. The atmosphere model is determined by the control `which_atm_option` in `&controls`. It should be one of the T(tau) integration options or Paczynski_grey, which takes into account radiation dilution when tau < 2/3,

```fortran
   keep_surface_point = .false.
```

If true, the surface point of the stellar model is included when the model is passed to the oscillation code.

```fortran
   add_center_point = .true.
```

As above, but for the centre point.

```fortran
   oscillation_code = 'adipls' ! or 'gyre'   <<< lowercase
   trace_time_in_oscillation_code = .false.
```

Selects the oscillation code and specifies whether to report how much time is spent computing the frequencies.

### GYRE controls

```fortran
   gyre_l0_input_file = 'gyre.l0.in'
   gyre_l1_input_file = 'gyre.l1.in'
   gyre_l2_input_file = 'gyre.l2.in'
   gyre_l3_input_file = 'gyre.l3.in'
```

These specify the input files for GYRE, for each degree.

```fortran
   ! comments from Rich on setting gyre controls

      ! I suggest setting freq_min to 0.9*MINVAL(l0_obs),
      ! and freq_max to 1.1*MAXVAL(l0_obs) 
      ! (similarly for the other l values).

      ! freq_units should be 'UHZ',
      ! and set grid_type to 'LINEAR'.

      ! For n_freq, I suggest either setting it to 10*(freq_max - freq_min)/dfreq,
      ! where dfreq is the estimated frequency spacing; or, set it to 10*nl0.
      ! The factor 10 is arbitrary, but seems to be a good safety factor.
```

### ADIPLS controls

```fortran
   do_redistribute_mesh = .false.
```

If true, the stellar model will be remeshed according to the settings in `redistrb.c.pruned.in`.

```fortran
   iscan_factor_l0 = 15
   iscan_factor_l1 = 15
   iscan_factor_l2 = 15
   iscan_factor_l3 = 15

   nu_lower_factor = 0.8
   nu_upper_factor = 1.2
```

ADIPLS looks for frequencies in a given range and with a given density of coverage. For example, for l=0, the frequency search range is `nu_lower_factor*l0_obs(1)` to `nu_upper_factor*l0_obs(nl0)` and ADIPLS uses `iscan = iscan_factor_l0*nl0`, where `iscan` is the number of scan points in the frequency range. The same applies for the higher degree modes.

```fortran
   adipls_irotkr = 0
   adipls_nprtkr = 0
   adipls_igm1kr = 0
   adipls_npgmkr = 0
```

These are a handful of extra controls that are passed to ADIPLS. Look them up in the ADIPLS documentation.

### Other inlists

```fortran
   read_extra_astero_search_inlist1 = .false.
   extra_astero_search_inlist1_name = 'undefined'

   read_extra_astero_search_inlist2 = .false.
   extra_astero_search_inlist2_name = 'undefined'

   read_extra_astero_search_inlist3 = .false.
   extra_astero_search_inlist3_name = 'undefined'

   read_extra_astero_search_inlist4 = .false.
   extra_astero_search_inlist4_name = 'undefined'

   read_extra_astero_search_inlist5 = .false.
   extra_astero_search_inlist5_name = 'undefined'
```

Finally, like all MESA inlists, there are controls to read other `astero_search_control` inlists. This is useful if you're trying to fitting multiple stars with similar optimization parameters.