---
title: "FreeEOS"
date: 2014-01-16T10:49:28-04:00
author: MESA-Dev
---

# Installing FreeEOS with the MESA SDK  
(updated 01/2014)

Describes how to install the FreeEOS library using the MESA SDK.

FreeEOS is a library for computing the equation of state (EOS) for stellar interiors work. It handles many non-ideal effects, see [freeeos.sourceforge.net](http://freeeos.sourceforge.net) for documentation.

These instructions suppose that you have FreeEOS version 2.2.1 and that you have the MESA SDK installed and typical environment variables set as explained on the SDK webpage.

It also supposes that you use bash for your shell. If not, then replace the `export VAR=junk` commands with appropriate for your shell, e.g. `setenv VAR junk`.

**Note that FreeEOS requires the cmake utility.**

This document is intended to explain how to compile FreeEOS using the MESA SDK, and to store the compiled library into the SDK lib directory so that you can use it easily with MESA.

## Step-by-step:

1. Download the FreeEOS tarball from the stable releases at [freeeos.sourceforge.net](http://freeeos.sourceforge.net). Unpack and have a look at the README. The MESA SDK already includes the LAPACK and BLAS libraries you'll need.
2. Follow the instructions for compiling FreeEOS (see section 4 of the README).

```bash
cd free_eos-2.2.1
mkdir build_dir; cd build_dir
export CMAKE_LIBRARY_PATH=$MESASDK_ROOT/lib
export PREFIX=$MESASDK_ROOT
cmake -DCMAKE_INSTALL_PREFIX=$PREFIX -DCMAKE_VERBOSE_MAKEFILE=ON .. >& cmake.out  # make sure this contains no error messages
make  # again, make sure no errors
make install  # may have to add sudo on to the front if you don't have write privileges to the PREFIX directory set above
```

That's it! If you want to try the test codes, search for "Run the compiled test code" in the FreeEOS README.

Now you should have the FreeEOS library installed in `$MESASDK_ROOT/lib` and the header files in `$MESASDK_ROOT/include`.

You'll still need to add `-lfree_eos` to the list of libraries in your makefile whenever you want to use FreeEOS with MESA code, but the MESA makefiles should know where to find it now.

**Note:** It was pointed out on mesa-users that the standard build doesn't work as of January 2014. It requires that some files be edited after running cmake -- or -- someone with better knowledge of cmake than me suggests a better fix. See this mesa-users thread for a suggested fix: [http://sourceforge.net/p/mesa/mailman/message/33283067/](http://sourceforge.net/p/mesa/mailman/message/33283067/)

