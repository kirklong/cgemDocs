Author: @kirklong

This goes through the extra steps I had to do to install dependencies required to compile the CGEM code -- I had no previous solar experience so this process will likely be similar to any starting from ~scratch on a Linux based machine.

**To install FFTW:** [Download FFTW](http://www.fftw.org/download.html) and extract (`sudo tar -xzvf file.tar.gz`), I like to install packages like this in `/opt/`. Need to compile with FORTRAN and OpenMP enabled -- after extracting change into directory and run `./configure --enable-openmp --with-g77-wrappers` followed by `sudo make` and finally `sudo make install`. After installing update the Makefile in the MF directory of the CGEM code to point to wherever you've installed FFTW.

**To install fishpack:** [Download FISHPACK 4.1](https://www2.cisl.ucar.edu/resources/legacy/fishpack) and unpack (`sudo tar -xopf file.tar`). By default the install requires that you have gmake and pgf90 -- check if these are installed (ie `which gmake`) before proceeding. My Linux distribution (Linux Mint 20.1) does not include these by default (just make instead of gmake and gfortran instead of pgf90) and I'm lazy, so instead of figuring out how to install them I hackily fixed (appears to work fine as code runs) this by creating a "fake" gmake that's a symbolic link to cmake (ie `sudo ln -s /usr/bin/make /usr/bin/gmake`) and I edited part of the `make.inc` file in the main fishpack directory to read:


```bash
LIB=../lib/libfishpack.a

UNAMES := $(shell uname -s)

ifeq ($(UNAMES),Linux)

 PGI := $(shell gfortran 2>&1)

 ifeq ($(PGI),pgf90-Warning-No files to process)

   F90 := pgf90 -module ../lib -I../lib
   CPP := pgf90 -E

 else

   F90 := gfortran -DG95 -g -fdefault-real-8 -L=../lib -I../lib
   CPP := gfortran -E -DG95 -fdefault-real-8

 endif

 MAKE := gmake
 AR := /usr/bin/ar
```

Note that I'm pretty sure you could also just change the MAKE variable to make instead of gmake in that file and then you wouldn't have to make the symbolic link, or alternatively could make a second symbolic link for pgf90 to gfortran -- the reason I did it this way is because I also added flags to the gfortran compile command that are not included by default (ie most importantly the `-fdefault-real-8` bit). You could also link this to the Intel compiler (`ifort` instead of `gfortran`), but when I did this step I did not yet have an Intel compiler installed. After installing update the Makefile in the MF directory of the CGEM code to point to wherever you've installed fishpack.

**To install SDF:** [Download SDF](http://solarmuri.ssl.berkeley.edu/~fisher/public/software/SDF/) and unpack in install directory of choice as before. The Makefile is configured by default for gcc, so if you're on Linux this will almost certainly be the easiest for you and you can just type `make` in the directory with Makefile to install. For other systems or if you encounter problems, see the `INSTALL.txt` file in the `docs` folder. After installing update the Makefile in the MF directory of the CGEM code to point to wherever you've installed SDF.

**To install IDL:** This is easy once you can figure out how to download it, as it appears you can't just buy a single use license as an individual? For me I had to buy it from my university bookstore ($30) and then our IT department had to send me an activation code + link to campus server where a recent version of the code was stored that I could download from. This was annoying because I had to wait a few days before I could install, and you can't do anything without IDL. Once you have the code follow the instructions (for me I just had to run a simple shell script and everything was good) your IT dept provides, but it should be relatively painless.

**To install SolarSoft:** Fill out the installation form [here](https://www.lmsal.com/solarsoft/) to generate a c shell install script -- I left the default Mac/Linux install location and checked just the SDO boxes as that's all you need for this code (really just HMI I think), but it should be easy to add more parts later if you want. Then download the resulting shell script, mark it as executable (`chmod +x`), start a csh/tcsh session (ie type `tcsh` from the bash prompt -- note if you don't have a csh/tcsh shell you're going to need one for the rest of the code, you can get one easily from your package manager like `sudo apt-get install tcsh`) and then run the .csh install script (`sudo ./ssw_install*.csh`). This took a little while on my computer but it wasn't too bad.

After running the script follow the instructions [here](https://www.lmsal.com/solarsoft/ssw_setup.html) (have to have IDL installed first). If the final command doesn't work, it's probably because you installed IDL in a place people didn't know about 20 years ago and/or your IDL file tree is slightly different than what it used to be (I encountered this issue, and the script I ended up changing to fix this had a comment indicating it was last changed in 1997...). To fix this issue I changed into the ssw directory (`cd $SSW`), then went into the setup folder and opened the `ssw_idl` file (`sudo vim ssw_idl`), modifying this bit of the .csh script starting on line 67 to become:

```shell
##################### Verify a working IDL_DIR is found ##################
# test to see if user has defined IDL_DIR, if not, set it.
# code is bulky due to moving RSI target and multi IDL Version support (SLF)
#
# slf 22-may-1997 - loop through
if !( $?IDL_DIR ) then
   set slist=(/opt/idl/idl /usr/local/lib/idl /Applications/itt/idl /usr/local/itt/idl /usr/local/itt/idl/idl80 /usr/local/rsi/idl_6 /usr/local/rsi/idl_5 /usr/local/rsi/idl_4 /usr/local/rsi/idl /Applications/rsi/idl)
   foreach member ($slist)
      if (-d $member) then
            setenv IDL_DIR $member
            break
      endif
   end
   if !( $?IDL_DIR ) then                    # still undefined??
      if (-d /usr/local/lib/idl) then
        # set idl_dir=`find /usr/local/lib/idl -name bin -print`
         set idl_dir=`find /opt/idl/idl -name bin -print`
         setenv IDL_DIR `echo $idl_dir[1]:h`
      else
         echo "Cannot find idl directory - (IDL_DIR)"
         exit
      endif
   endif
endif
```

Since I installed my version of IDL in /opt/ I added that as an option to the first bit of the `set slist` command + the fallback `set idl_dir` command in the nested if statement. In the old days it appears it used to just go `IDL_DIR/bin/executable` but for my install it's now `IDL_DIR/idl/bin/executable` or `IDL_DIR/idl88/bin/executable`, and the script doesn't look in /opt/ by default for an executable. After doing this you should be able to type `sswidl` and get to an IDL command prompt within SolarSoft -- yay!

**To install Intel compilers:** To compile later steps of the code you will need to use ifort, not gfortran. I did not have ifort, and it was hard to figure out how to install it. Download the local shell script for your OS [here](https://software.intel.com/content/www/us/en/develop/articles/oneapi-standalone-components.html#fortran), mark it as executable, then run it. On my Linux distribution it installed itself in /opt/intel/oneapi/compiler/latest/linux/bin/intel64/ifort -- if you want to add permanently to path, find wherever yours has installed (should tell you at end of installation) and then create a symbolic link in /usr/bin to add ifort to your path (ie change into /usr/bin, then `sudo ln -s /opt/intel/oneapi/compiler/latest/linux/bin/intel64/ifort`). You also need to install the Intel MPI libary -- a link to them is further down that same [page](https://software.intel.com/content/www/us/en/develop/articles/oneapi-standalone-components.html#fortran). Download that script and run it just as you did the previous one, then link mpiifort as you did with ifort.

Alternatively (easier way if you don't think you're going to use the intel compilers all the time -- this is what I ended up doing), every time you're going to use the Intel libraries you can just source the script `setvars.sh` found in /opt/intel/oneapi after installing both packages, which will then define ifort and mpiifort for that shell session (but not permanently). If you go this route follow the extra instructions in the main README. 
