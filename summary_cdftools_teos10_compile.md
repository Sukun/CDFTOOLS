
# CDFTOOLS with TEOS-10 Support: Installation Guide

This guide details the process of enabling TEOS-10 (Thermodynamic Equation of Seawater - 2010) support in CDFTOOLS by linking the GSW-Fortran library. This is required for correctly processing modern ocean model outputs (e.g., NEMO v4+) that use Conservative Temperature and Absolute Salinity.

## Prerequisites

- **Fortran Compiler**: `gfortran` (recommended) or `ifort`.
- **NetCDF Libraries**: `libnetcdf` and `libnetcdff` (C and Fortran interfaces).
- **CMake**: Version 3.0+ (highly recommended for building GSW-Fortran).
- **Git**: For cloning repositories.

***

## Step 1: Download and Compile GSW-Fortran

CDFTOOLS relies on the external GSW-Fortran library. You must download and compile it as a static library first.

### 1. Clone the Repository

```bash
# Navigate to your tools directory
cd /path/to/your/tools/

# Clone the official GSW-Fortran repository
git clone https://github.com/TEOS-10/GSW-Fortran.git
cd GSW-Fortran
```


### 2. Build the Library

**Method A: Using CMake (Recommended)**
This method automatically handles dependencies and module generation.

```bash
# Clean previous builds if any
rm -rf build

# Create build directory
mkdir build && cd build

# Configure and compile
cmake ..
make

# Verify outputs
ls -lh libgsw.a        # The static library
find . -name "*.mod"   # The module files
```

**Method B: Manual Compilation (If CMake is unavailable)**
If you cannot use CMake, you must manually compile source files and create the static archive.

```bash
# Create directories
mkdir -p lib modules

# Compile modules and toolbox files
# Note: Ensure you include the current directory for module files
gfortran -c modules/*.f90 toolbox/*.f90 -I./modules -J./modules -O3 -fPIC

# Create the static library archive
ar rcs lib/libgsw.a *.o

# Move module files to the modules directory if not already there
mv *.mod modules/

# Clean up object files
rm *.o
```


***

## Step 2: Configure CDFTOOLS

You need to modify the CDFTOOLS macro file to point to your compiled GSW library.

### 1. Locate or Create `make.macro`

Navigate to the CDFTOOLS source directory or macro library.

```bash
cd /path/to/cdftools/
# Copy a template if you haven't already
cp Macrolib/macro.gfortran make.macro
```


### 2. Edit `make.macro`

Open `make.macro` and modify the following sections. **Crucially, you must enable the `-D key_GSW` preprocessor flag.**

```makefile
# ... (Keep existing NetCDF settings) ...

# =================================================================
# TEOS-10 / GSW Support Configuration
# =================================================================

# 1. Enable the GSW preprocessor flag (CRITICAL!)
GSW = -D key_GSW

# 2. Define paths to GSW headers (.mod files) and library (.a file)
# Replace /path/to/... with your actual GSW-Fortran directory

# If you used CMake (Method A):
GSWINC = -I/path/to/GSW-Fortran/build/modules
GSWLIB = /path/to/GSW-Fortran/build/libgsw.a

# If you used Manual Compilation (Method B):
# GSWINC = -I/path/to/GSW-Fortran/modules
# GSWLIB = /path/to/GSW-Fortran/lib/libgsw.a

# =================================================================
# Compilation Flags
# =================================================================

# 3. Add $(GSW) and $(GSWINC) to FFLAGS
# Ensure $(GSW) is present so the compiler sees the -D key_GSW flag
FFLAGS = -O $(NCDF) $(NC4) $(CMIP6) $(GSW) $(GSWINC) -fno-second-underscore -ffree-line-length-256

# 4. Add $(GSWLIB) to Linking Flags
# Using an absolute path in GSWLIB avoids -L/-l lookup issues
LDFLAGS = $(GSWLIB)
```


***

## Step 3: Compile CDFTOOLS

Now compile the tools. You can compile specific tools or the entire suite.

```bash
cd /path/to/cdftools/src/

# Clean previous builds
make clean

# Compile a specific tool to test (e.g., cdfmocsig)
make cdfmocsig

# Or compile everything
make
```


### Verification

Check if the tool compiled with TEOS-10 support:

```bash
./cdfmocsig --help
```

You should see references to `-teos10` in the help text.

***

## Step 4: Running with TEOS-10

When using the tools, you must explicitly enable TEOS-10 mode and provide the correct input variables.

```bash
./cdfmocsig -t conservative_temp.nc -s absolute_salinity.nc -teos10 ...
```

- **`-teos10`**: Flag to use GSW functions for density calculation.
- **Inputs**: Ensure inputs are **Conservative Temperature** and **Absolute Salinity**. CDFTOOLS *does not* check the physical validity of inputs; it blindly computes density based on the flag provided.

***

## Troubleshooting \& Common Issues

### 1. Linker Error: `cannot find -lgsw`

* **Cause**: The linker cannot locate `libgsw.a`.
* **Fix**: Use the absolute path to the library file in `GSWLIB` (e.g., `GSWLIB = /abs/path/to/libgsw.a`) instead of `-L/path -lgsw`.


### 2. Compiler Error: `Can't open module file 'gsw_mod_toolbox.mod'`

* **Cause**: The compiler cannot find the `.mod` files.
* **Fix**: Check that `GSWINC` points to the exact directory containing the `.mod` files. Note that CMake often places them in `build/` or `build/modules/`.


### 3. Compilation Ignores TEOS-10 Code

* **Symptom**: Compilation succeeds, but `-teos10` flag is unrecognized or results are identical to EOS80.
* **Cause**: The preprocessor flag `-D key_GSW` was missing during compilation.
* **Fix**: Ensure `GSW = -D key_GSW` is defined in `make.macro` and that `$(GSW)` is included in the `FFLAGS` variable.


### 4. `undefined reference to gsw_*`

* **Cause**: Header files were found (compilation passed), but the library was not linked (linking failed).
* **Fix**: Ensure `$(GSWLIB)` is added to `LDFLAGS` or appended to the end of the compile command.


### 5. `multiple definition of main`

* **Cause**: You mistakenly included the `test/` directory object files in `libgsw.a`.
* **Fix**: Rebuild `libgsw.a` excluding the `test/` directory. Only include `modules/` and `toolbox/`.
