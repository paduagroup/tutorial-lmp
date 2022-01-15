# tutorial-lmp

Running a molecular simulation with the CL&Pol polarizable force field for ionic liquids in LAMMPS.

## Steps

1. Make sure the necessary packages are installed: Python, VMD, Packmol and LAMMPS compiled with the DRUDE package. 
2. Download the CL&P force field. This is the non-polarizable version on which CL&Pol is based.
3. Download the fftool script, to create input files for LAMMPS and an initial configuration of the simulation box.
4. Download the `polarizer` tools and files needed to generate the CL&Pol force field.
5. Use `fftool` to create input files and an initial configuration using the non-polarizable force field.
6. Use the `polarizer` tool to add the elements needed for a polarizable simulation.
7. Run an equilibration trajectory.
8. Run a production trajectory.
9. Do post-treatment calculations of structural and dynamic quantities from the trajectory.

## Make sure packages are installed

If you are following this tutorial on a machine of the CBP center at ENS de Lyon, the packages needed are installed. If you are logging to the CBP machines from outside, then it is pertinent that you install VMD in your computer because visualization may be slow over an internet connection.

Python 3 is necessary to run the tools we develop.

VMD is a molecular visualization program:
    
    vmd
    
Packmol is a program that packs molecules in simulation boxes.
    
    packmol
    
Finally, LAMMPS is the molecular dynamics code we use in this tutorial. On the CBP LAMMPS is installed in `/projects/RFCT2022/software/lammps`.

    /projects/RFCT2022/software/lammps/bin/lmp -h
    
and check that the binary was compiled with the DRUDE package

