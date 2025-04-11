# Compiling SQuIRE with Modern GCC (11.2.0+)

<img align="right" width="260" height="260" src="https://raw.githubusercontent.com/wyang17/SQuIRE/master/images/squire.png">

**SQuIRE** (Software for Quantifying Interspersed Repeat Expression) is a valuable tool for transposable element analysis, but it encounters compilation issues with modern compilers like GCC 11.2.0+. This guide explains how to overcome these compatibility challenges.

**Original repository:** [wyang17/SQuIRE](https://github.com/wyang17/SQuIRE)

## The Problem

SQuIRE was developed when C++ standards were less strict. Modern compilers enforce rules that cause compilation failures when building SQuIRE's dependencies (particularly BEDTools 2.25.0).

When compiling with GCC 11.2.0 or newer, you'll encounter errors like:
```
static assertion failed: value_type must be the same as the underlying container
```

Direct edits to source files don't work because the SQuIRE build process extracts fresh source code each time you run the build command, overwriting any changes.

## Solution: Continuous Monitoring Script

Our approach uses a script that continuously monitors the build process and automatically applies fixes whenever source files are extracted or regenerated.

### Notes ###

_*SQuIRE is compatible with the following specific versions of software:_*
* `STAR 2.5.3a`
* `bedtools 2.25.0`
* `samtools 1.1`
* `stringtie 1.3.3`
* `DESeq2 1.16.1`
* `R 3.4.1`
* `Python 2.7.4`
  
Maybe you should download these softwares first
Attention!This software only run on python 2.7.4 environment!!!

You need install python 2.7.4 first
```
conda create -n squire python=2.7.4
```

### Step 0: Download squire software by pip (no dependency software version)
```
pip install squire
```
You may get many pip dependency package error, try to fixed it by yourself😊.

### Step 1: Create the Helper Script

Create a file named `fix_compiler.sh`:
> I have create this script in sourcecode file

```bash
#!/bin/bash

echo "Starting SQuIRE compatibility monitoring for GCC 11.2.0+"
echo "Keep this script running while you build SQuIRE in another terminal"

# Create directory for original files (optional but useful for debugging)
mkdir -p ./squire_build/original_files

# Continuous monitoring loop
echo "Beginning continuous monitoring of source files..."
while true; do
  # Fix StrandQueue.h template parameter mismatch
  if [ -f ./squire_build/bedtools-2.25.0/bedtools2/src/utils/FileRecordTools/Records/StrandQueue.h ]; then
    if grep -q "vector<const Record \*" ./squire_build/bedtools-2.25.0/bedtools2/src/utils/FileRecordTools/Records/StrandQueue.h; then
      echo "$(date): Detected original StrandQueue.h - applying fix..."
      sed -i 's/vector<const Record \*/vector<Record \*/g' ./squire_build/bedtools-2.25.0/bedtools2/src/utils/FileRecordTools/Records/StrandQueue.h
      echo "$(date): Fix applied to StrandQueue.h"
    fi
  fi
  
  # Fix Makefiles to use permissive compilation flags
  find ./squire_build -name "Makefile" 2>/dev/null | while read makefile; do
    if ! grep -q -- "-fpermissive" "$makefile"; then
      echo "$(date): Found unmodified Makefile at $makefile - applying fix..."
      sed -i 's/CXXFLAGS =/CXXFLAGS = -fpermissive -Wno-register -Wno-deprecated -std=c++11 /g' "$makefile"
      echo "$(date): Fixed Makefile at $makefile"
    fi
  done
  
  # Brief pause to avoid excessive CPU usage
  sleep 1
done
```

### Step 2: Make the Script Executable

```bash
chmod +x fix_compiler.sh
```

## Compilation Process

The compilation requires two terminal windows running simultaneously:

### Terminal 1: Run the Monitoring Script

```bash
./fix_compiler.sh
```

This script will continuously run, monitoring for source files and automatically applying fixes as needed. You'll see messages whenever it detects and fixes files.

### Terminal 2: Build SQuIRE

```bash
# Activate your conda environment first
conda activate squire

# Run the build command
squire Build -s all
```

## How It Works

The script performs several important functions:

1. **Continuous Monitoring**: It constantly checks for newly extracted source files
2. **Template Fixes**: It corrects C++ template parameter mismatches in StrandQueue.h
3. **Compiler Flag Adjustments**: It adds these essential flags to all Makefiles:
   - `-fpermissive`: Makes the compiler more lenient about type mismatches
   - `-Wno-register`: Suppresses warnings about the deprecated register keyword
   - `-Wno-deprecated`: Ignores other deprecation warnings
   - `-std=c++11`: Explicitly sets the C++ standard

During compilation, you'll see:

- **Terminal 1**: Messages about files being detected and fixed
- **Terminal 2**: The normal build output, which should now proceed without errors

If the build process extracts new files or regenerates previously modified ones, the script will detect and fix them again automatically.

## Troubleshooting

If you encounter additional errors:

- Check Terminal 1 for messages about detected and fixed files
- Look for new error messages in Terminal 2
- You may need to modify the script to include fixes for additional files

> **Note**: If SQuIRE extracts source code to a different location than expected, you'll need to update the paths in the script accordingly.

## Why This Works

This approach allows SQuIRE to be compiled successfully on systems with modern GCC compilers by:

- Addressing C++11 template parameter compatibility issues
- Suppressing warnings about deprecated features
- Using more permissive compiler settings
- Continuously applying fixes even when the build process overwrites modified files

The result is a successful build of SQuIRE and its dependencies without having to modify the original build process or downgrade your compiler.
