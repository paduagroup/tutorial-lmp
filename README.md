# tutorial-lmp

Running a molecular simulation with the CL&Pol polarizable force field for ionic liquids in LAMMPS.

## Steps

1. Make sure the necessary packages are installed: Python, VMD, Packmol and LAMMPS compiled with the DRUDE package. 
2. Download the CL&P force field (non-polarizable version), the `fftool` script (to create input files and simulation box) and the `polarizer` tools (to generate the CL&Pol force field).
3. Create input files and an initial simulation box with the non-polarizable force field using `fftool` and `packmol`.
4. Add explicit polarization terms using the `polarizer` tool.
5. Run an equilibration trajectory.
6. Run a production trajectory.
7. Do post-treatment calculations of structural and dynamic quantities from the trajectory.

GO SLOWLY. TRY TO UNDERSTAND EACH STEP.


## 1. Make sure software is installed

If you are following this tutorial on a machine of the CBP center at ENS de Lyon, the packages needed are installed. If you are logging to the CBP machines from outside, then it is pertinent that you install VMD in your computer because visualization may be slow over an internet connection.

Python 3 is necessary to run the tools we develop.

VMD is a molecular visualization program:
    
    vmd
    
Packmol is a program that packs molecules in simulation boxes.
    
    packmol
    
Finally, LAMMPS is the molecular dynamics code we use in this tutorial. On the CBP LAMMPS is installed in `/projects/RFCT2022/software/lammps`.

    /projects/RFCT2022/software/lammps/bin/lmp -h
    
and check that the binary was compiled with the DRUDE package (or USER-DRUDE in older versions).

## 2. Download CL&P, fftool and CL&Pol

These tools are available on [https://github.com/paduagroup] 
        mkdir sim
        cd sim
        git clone https://github.com/paduagroup/clanp
        git clone https://github.com/paduagroup/fftool
        git clone https://github.com/paduagroup/clanpol
        
You can check that `fftool` runs and learn about the command-line options
        ~/sim/fftool/fftool -h
        
## 3. Create an initial simulation box

Let's try an ionic liquids composed of the butylmethylimidazolium cation and the bistriflamide anion, [C4C1im][Ntf2]
        mkdir c4c1im_ntf2
        cd c4c1im_ntf2

Copy the CL&P data base and molecule specification files for the ions:
        cp ~/sim/clandp/clandp.ff .
        cp ~/sim/clandp/c4c1im.zmat .
        cp ~/sim/clandp/ntf2.zmat .
        
Study these files. The molecule specifications files contain atomic coordinates in `xyz` or `z-matrix` format and point to a file `.ff` which is a database of force field parameters.

Use `fftool` to create a system with one ion pair in a cubic box of 30 Å:
        ~/sim/fftool/fftool 1 c4c1im.zmat 1 ntf2.zmat -b 30

Verify the charges of the ions are correct. Inspect the `pack.inp` file which is an input for `packmol`.

Use `packmol` to generate the atomic coordinates placing the ions in the box:
        packmol < pack.inp

Have a look at the box:
        vmd simbox.xyz

Run `fftool` again to generate the LAMMPS in put files:
        ~/sim/fftool/fftool 1 c4c1im.zmat 1 ntf2.zmat -b 30
        
Study the `in.lmp` and `data.lmp` files. These are LAMMPS input files.

