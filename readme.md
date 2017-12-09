# Spack package manager at CHPC

## Basic information

- features overview - [http://spack.readthedocs.io/en/latest/features.html](http://spack.readthedocs.io/en/latest/features.html)
- Github repo - [https://github.com/spack/spack](https://github.com/spack/spack)
- documentation - [http://spack.readthedocs.io/en/latest/](http://spack.readthedocs.io/en/latest/)
- tutorial (very useful for learning both basic and more advanced functionality) - [http://spack.readthedocs.io/en/latest/tutorial.html](http://spack.readthedocs.io/en/latest/tutorial.html)

## Installation and setup

### CHPC custom configuration (to be confirmed)

```/uufs/chpc.utah.edu/sys/spack``` - root directory for package installation

```/uufs/chpc.utah.edu/sys/modulefiles/spack``` - Lmod module files

```/scratch/local``` - local drive where the build is performed (= need to make sure to build on machine that has it)


### Package installation and setup

Install from Github (may need to version in the future so may need to have own fork)

```git clone https://github.com/spack/spack.git```

Basic setup (to be put to hpcapps .tcshrc)
```
setenv SPACK_ROOT /uufs/chpc.utah.edu/sys/installdir/spack/spack
source $SPACK_ROOT/share/spack/setup-env.csh
```

### CHPC configuration

All configuration files (and package definition files) are in YAML format so pay attention to indentation, etc.
First basic configuration (paths, ...):
```
cp etc/spack/defaults/config.yaml etc/spack/
spack config --scope site edit config
```
Then based on Argonne configs:
```
spack config --scope site edit compilers
spack config --scope site edit modules
spack config --scope site edit packages
```
check with
```
spack config get config
```
Compilers defined are the latest Intel (2018.0), PGI (17.10), stock gcc (4.8.5) and gcc(5.4.0). More can be added, or perhaps easier built with Spack.

(go over config files and explain concretization preferences in ```packages.yaml```, as per [http://spack.readthedocs.io/en/latest/build_settings.html#concretization-preferences](http://spack.readthedocs.io/en/latest/build_settings.html#concretization-preferences))

Other preinstalled packages that have been included into Spack (list may grow in the future):
- Intel MPI
- Intel MKL

### CHPC Lmod integration
In our module shell init files, we also need to ```use``` the Spack Lmod module tree:

``` ml use /uufs/chpc.utah.edu/sys/modulefiles/spack/linux-centos7-x86_64/Core```

For compilers - each compiler also needs to load the compiler specific module files, e.g.:

```/uufs/chpc.utah.edu/sys/modulefiles/spack/linux-centos7-x86_64/intel/2018.0.128```

Now emulate this as:

```
ml intel/18
ml use /uufs/chpc.utah.edu/sys/modulefiles/spack/linux-centos7-x86_64/intel/2018.0.128
```

Since we are using Spack's Intel MPI package definition, the IMPI path is set up correctly when loading ```intel-mpi``` module from the Spack module tree.

For gcc, things are a bit different. gcc compiler modules equal to ```Core```, which contains intel-mpi, which sets the module path to IMPI.
(had to fix intel-mpi module as it is for some reason doing ml() and module())

### Adding already installed package (installed w/o spack)

Edit the ```packages``` config file appropriately:
```
spack config --scope site edit packages
  intel-mkl:
    paths:
      intel-mkl@2018.0.128 arch=linux-centos7-x86_64: /uufs/chpc.utah.edu/sys/installdir/intel/compilers_and_libraries_2018.0.128/linux/mkl
    buildable: False
```
- the ```buildable: False``` tells Spack to never install this package

In the actual package file, check the ```prefix``` variable so it maps to our directory structure
```
spack edit intel-mkl
```
Check what dependencies is Spack going to get:
```
spack spec hpl
spack spec hpl%intel
```

Test build the code (if the ```prefix``` is not correct, the build will crash here)
```spack install hpl```

## Basic use

i.e. build predefined packages

### Source Spack

if not done in .tcshrc
```
setenv SPACK_ROOT /uufs/chpc.utah.edu/sys/installdir/spack/spack
source $SPACK_ROOT/share/spack/setup-env.csh
```

### Basic installation workflow
```spack list <package>``` - find if the package is known to Spack

```spack compilers``` - list available compilers (pre-installed as defined in ```compilers.yaml``` file or installed with Spack)

```spack spec <package> <options>``` - see to be installed version/compiler/dependencies

```spack env <package><options>``` - display build environment. TIP: make sure to have all variants listed w/o space otherwise the `env` command will want to run the spaced out variant as another command, e.g. `spack env espresso@6.1.0%intel@2018.0.128~elpa+mpi+openmp^intel-mkl`

```spack install <package> <options>``` - install the package

```spack find -dl <package>``` - display installed packages (```-dl``` will print version details)

#### Dependencies:
```%``` - compiler, e.g. ```%intel```

```@``` - version, e.b. ```%intel@2018.0.128```

... more to be added

#### Examples
```spack install hpl%intel``` - install HPL with default version of Intel compiler and default BLAS (MKL) and MPI (Intel MPI).

```spack install --keep-stage espresso@6.1.0 %intel@2018.0.128 -elpa +mpi +openmp ^intel-mkl threads=openmp``` - install Quantum Espresso with Intel compiler, MKL, Intel MPI (default) and threads

#### Modifying the spec file
In general, be careful when editing the spec files so they don't break. In the future we will use our own branch of Spack to version control our changes and potentially create pull requests to Spack's main branch if we add new packages or do useful change to existing package spec.

```spack edit <package>``` will open vi editor with the package's package.py spec file. E.g. ```spack edit hpl```.
```env['F90']=spack_fc``` - adds environment variable F90 that point's to Spack's defined FC compiler, ```spack_fc```
```env['FCFLAGS']='-g -O3 -ip'``` - adds explicitly defined environment variable

#### Adding new version of an existing package

First get the checksum with `spack checksum <package>`. If new version is not found, see what is the hierarchy of the versions in the URL specified in the `package.py`. This can be specified by `list_url` and `list_depth`, as e.g. in `postgresql` package. This is detailed in [http://spack.readthedocs.io/en/latest/packaging_guide.html#finding-new-versions](http://spack.readthedocs.io/en/latest/packaging_guide.html#finding-new-versions)

#### Troubleshooting
```spack -d <command>```  will print out more detailed information with stack trace of the error - one then can look in the Spack python code and potentially modify it (as hpcapps) to print function arguments that may hint on the problem

```spack help -a``` - gives the most generic help

``` spack python``` - runs Python with Spack module being loaded, so, one can run Spack functions, e.g. ```>>> print(spack.config.get_config('config'))```

## Things to discuss at CHPC
- install dir and local drive for building, module files location
- Lmod integration (module use for the Spack generated modules)
- consider allowing only one compiler (spack modules set CC, CXX, ...)
- consider bash as shell for hpcapps
- policy in locating and loading Python modules
- platform specific settings/install tree - no need to specific sys branch for different architectures (x84, ppc64)

### Spack vs. easybuild
- spack has stronger dependency model (DAG), uninstall
- easier to use by user feedback
- power users can fairly easily use it for their own package building

## Advanced usage

I.e. installing a package that is not defined yet

### Creating a package definition file

- follow the package creation tutorial [http://spack.readthedocs.io/en/latest/tutorial_packaging.html](http://spack.readthedocs.io/en/latest/tutorial_packaging.html) if stuck.
- create basic YAML definition file that needs to be further filled in
 ```spack create <package url>```
- edit the file and fill in what's needed
```spack edit <package>```
- find a package that is similar to what you're trying to install to get further ideas on what to do, based on the build specifics
-- autoconf, cmake, make, ...

### Debugging spack

I.e. figuring out where build or other errors come from

1. Run Spack inside Python
- source spack
- get into Python with `spack python`
- run Spack functions, e.g. `>>> print(spack.config.get_config('config'))` 

2. Put temporary print statements into the Spack code


## Plan:
- go over the ANL config and test - DONE
- test implementation in installdir - DONE
- local customizations, such as installdir and local scratch - DONE
 - ANL has /soft/spack - we can have /uufs/chpc.utah.edu/sys/spack
- local preinstalled software - mainly - PART
- prototype installdir structure, modules (module use for the spack built tree) - DONE
- apps to try (unless there's an explicit need)
 -- hpl - DONE
 -- check difference between intel, intel-parallel-studio, intel-mpi -> use pre-installed intel, intel-mpi, no intel-parallel-studio as it puts everything in one module
 -- when building look at how dependencies are being built (e.g. mpi, fftw)
- check package files
 -- lammps, matlab, pgi, namd, nwchem, espresso (QE)
- describe workflow on github
 -- installing existing package version - DONE
 -- installing new version of an existing package (including github pull request)
- things to decide/discuss in the future 
 -- python and R - Spack has separate package definitions for individual Python and R libraries
- package files to create - WRF, Amber
- webpage documentation for enabling power users to build codes on their own
- more preinstalled packages - Matlab, IDL
- Python specifics
 - make sure numpy, etc, are built with MKL - current py-numpy spec is confusing


## Tidbits

Local package repos
 var/spack/repos/builtin

Tidbits
 - use RPATH
 - generate modulefiles
 - - and ~ do the same thing - ~ may expand to user name in which case use - instead
 - can use find through dependencies, flags, etc
 - spack find -pe -- lists the location of the installed packages
   -- can change the root location, but spack will maintain the tree structure
 - make version dependencies
   ^openmpi@1.4: - use version higher than, inclusive (python slices)
   ^mpi@2 - spec constraint (MPI spec > 2.0)
 - extensions - Python packages - 
  -- they also have module files for each python package - may work better for us
 - while installing, can pres "v" for verbose output


- GPU - depend("cuda")

spack build cache -h - what is in binary cache

 - location of the packages to install
 -- etc/spack/defaults/config.yaml - never edit - git versioned
   -> $SPACK_ROOT - path to spack install that we're using
  -> $SPACK_ROOT/etc/spack/config.yaml 
  -> install_tree
 - can edit config files, e.g. 
    spack config edit compilers
     -- that edits the user specific config 
    spack config --scope defaults edit config
     -- that edits global config
 - compilers can be specified through modules in the config file - may be better for us
 - use distro packages
    zlib:
    paths:
      zlib@1.2.8%gcc@5.4.0 arch=linux-ubuntu16.04-x86_64: /usr
    buildable: False
  -> link version/arch to /usr and tell spack to never build it (even for different compiler)
  -> probably want to set some to this
  - limit # of threads to build package (default max cores)
   - just use -j xx in the spack instal, or in config file
   config:
     build_jobs: 4
 - in the config file use local mirror if it's available


Creating package names
 -- quick creation - will create basic YAML definition file that needs to be further filled in
 ```spack create <package url>```
 



