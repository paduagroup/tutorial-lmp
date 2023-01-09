# tutorial-lmp

This tutorial gives step-by-step instructions to setup a system and run a molecular dynamics simulation with the CL&P (fixed-charge) and the CL&Pol (polarizable) force fields for ionic liquids, using the LAMMPS code. It guides through the following steps:

1. Make sure the necessary packages are installed: Python, VMD, Packmol, and LAMMPS compiled with the DRUDE package. 
2. Download the CL&P force field (non-polarizable version), the `fftool` script (to create input files and simulation box) and the `polarizer` tools (to generate the CL&Pol polarizable model).
3. Create input files and an initial simulation box with the non-polarizable force field using `fftool` and `packmol`.
4. Run an equilibration trajectory (fixed-charge model).
5. Add explicit polarization terms using the `polarizer` tool.
6. Adjust the Lennard-Jones potentials to account for the explicit polarization terms.
7. Run a polarizable simulation.
8. Calculate structural and dynamic quantities from the trajectory.

Molecular simulation using force fields requires the specification of **a lot of information** concerning the system (coordinates, topology), the force field model and the simulation conditions. 

**Go slowly. Inspect input and output files. Try to understand each step.**

---

## 1  Make sure software is installed

>If you are following this tutorial on a machine of the CBP center at ENS de Lyon, the packages needed are installed. If you login to a CBP machine from outside, then it is pertinent that you install VMD in your computer because visualization may be slow over an internet connection.

Python 3 is necessary to run the tools we develop in the group (`fftool`, etc).

VMD is a molecular visualization program.
    
        vmd

Gnuplot is good to quickly plot graphs (you can use any other plotting software).

Packmol is a program that packs molecules in simulation boxes.
    
        packmol
    
Finally, LAMMPS is the molecular dynamics code we use in this tutorial.

        export LMP=/path/to/lammps/bin
        $LMP/lmp -h
    
and check that LAMMPS was compiled with the MOLECULE, KSPACE, CORESHELL and DRUDE packages.
 

---

## 2  Download CL&P, fftool and CL&Pol

These tools are available on [github.com/paduagroup](https://github.com/paduagroup)

        mkdir sim
        cd sim
        git clone https://github.com/paduagroup/clandp
        git clone https://github.com/paduagroup/fftool
        git clone https://github.com/paduagroup/clandpol

If cloning doesn't work because of permissions, try to download a zip archive directly from the GitHub pages.

You can check that `fftool` runs and **learn about its command-line options**:

        ~/sim/fftool/fftool -h

---

## 3  Create a simulation box

Let's try one of the most studied ionic liquids, composed of the butylmethylimidazolium cation and the bis(trifluoromethanesulfonyl)amide anion (a.k.a. TFSI), [C4C1im][Ntf2]:

        mkdir c4c1im_ntf2
        cd c4c1im_ntf2

copy the CL&P parameter database and molecule specification files for these ions:

        cp ~/sim/clandp/il.ff .
        cp ~/sim/clandp/c4c1im.zmat .
        cp ~/sim/clandp/ntf2.zmat .
        
Study these files. The molecule specification files contain atomic coordinates in `xyz` or `z-matrix` format and point to a `.ff` file which is the database of force-field parameters.


### 3.1 Create a test system with one molecule or ion

Use `fftool` to create a system with **one molecule** in a cubic box of 30 Å side:

        ~/sim/fftool/fftool 1 c4c1im.zmat -b 30

**Verify the electrostatic charge** of the molecule or ion (if it is not an integer then the attribution of atom types from the force field is not correct).

Inspect the `pack.inp` file, which contains instructions for `packmol` to pack the molecules in the simulation box.

Use `packmol` to generate the atomic coordinates packing the ions in the box:

        packmol < pack.inp

This generates a `simbox.xyz` file with atomic coordinates. View the molecules in the box:

        vmd simbox.xyz

Run `fftool` again repeating the previous command line with `-l` to generate the LAMMPS input files including the force field and the initial configuration:

        ~/sim/fftool/fftool 1 c4c1im.zmat -b 30 -l
        

### 3.2 Look at the `data.lmp` file

The `data.lmp` file describes the system to be simulated and is usually a large file, not meant to be edited by hand. It includes box dimensions, atomic coordinates, electrostatic charges and molecular topology (every atom, bond, angle, torsion in the system). LAMMPS is a very atomistic code, working at the level of the atoms, with molecules thinly defined by just an index.

**Verify the number of bonds**: if this is not the number expected for the molecule, then the topology is wrong (maybe the initial geometry is not correct, with some bonds significantly different from the equilibrium distances in the force field file `il.ff`).

> **Question:** What is the general rule to determine the number of bonds in a molecule given the number of atoms and cycles? (It's easy.) And the number of angles (between 3 atoms connected 1-2-3)? And torsions or dihedrals (between 4 atoms connected 1-2-3-4)?

Look at the `Atoms` section which lists: atom index, molecule index, atom type, charge, x, y, z.

        Atoms

              1       1    1   0.150000  3.973990e+00  8.929037e+00  3.934037e+00  # NA c4c1im+
              2       1    2  -0.110000  3.751917e+00  7.657070e+00  3.685020e+00  # CR c4c1im+
              3       1    1   0.150000  4.051689e+00  6.936962e+00  4.743701e+00  # NA c4c1im+
              ...


### 3.3  Study the `in.lmp` file

The `in.lmp` file contains LAMMPS commands to perform a simulation, so it is meant to be read and modified by the user.

In our examples it also contains certain informations about the force field, namely:

- the Lennard-Jones parameters of the different atom types (may be placed in an included file `pair.lmp`),
- the cutoff distance for the non-bonded interactions,
- the long-range part of the electrostatic interactions (`kspace`).

LAMMPS has a **very large set of commands**, which you can get acquainted with at [docs.lammps.org/Commands.html](https://docs.lammps.org/Commands.html). LAMMPS is a generalist code that can model molecules and materials, and perform many different calculations during the run itself, so it provides a wide array of commands to evaluate quantities or to modify the system.

Two important classes of LAMMPS commands are `fix` and `compute`:

- a `fix` is executed in the main time-step iteration loop of molecular dynamics, at specified intervals; `fix`es are the main workhorses to perform calculations or operate on the system in LAMMPS; a `fix` is referred to in other commands by prepending `f_`;
- a `compute` declares a calculation but does not execute it: it has to be called from a `fix` by prepending `c_`.
    
The `variable` command declares variables containing either numerical values, formulas or other types that can be used in other commands by prepending `v_`.

The `thermo` command prints information on thermodynamic quantities during the simulation (energies, temperature, pressure, density, etc.) and can also print values of computes, fixes and variables (it acts like a `fix`).

The `dump` command writes configurations to a trajectory file, which `vmd` can display.

Integrators of the equations of motion, a central piece in molecular dynamics, are implemented as `fix`es. LAMMPS has a choice of integrators including `fix nve` (isolated system), `fix nvt` to couple the system with a thermostat, or `fix npt` add pressure regulation by changing the box dimensions.

Every command used in a simulation should be fully understood. This is a long learning process...


### 3.4 Repeat for each component

Repeat 3.1 for each component, verifying the charge and the number of bonds.


### 3.5 Create a box for a larger system

Let us build a box with 200 ion pairs with a confortable box size so that molecules do not overlap and are not too far either. Since we don't know beforehand the size of the box, let's give a density of 2.5 mol/L:

        ~/sim/fftool/fftool 200 c4c1im.zmat 200 ntf2.zmat -r2.5
        packmol < pack.inp
        
If `packmol` takes more than about 30 seconds, the density is probably too high. Try a smaller value.

Visualize with `vmd`.

Run `fftool` again with `-a -l` to generate the input files: 

        ~/sim/fftool/fftool 200 c4c1im.zmat 200 ntf2.zmat -r2.5 -a -l

**Good! You just set up a system!**

---

## 4  Run an equilibration trajectory

Run a few steps to make sure the system does not blow up.

Edit the `in.lmp` to run a few steps in NPT:

        ...
        thermo 200
        ...
        dump TRAJ all custom 200 dump.lammpstrj ...
        ...
        run 20000

Run on 16 cores:

        mpirun -np 16 $LMP/lmp -in in.lmp > out.lmp &

Follow the progress with (`Ctrl-C` to exit)

        tail -f out.lmp

Once LAMMPS terminates, note the performance in ns/day. See in which tasks it spends most of the computation time (pair interactions, k-space electrostatics, ...).

Visualize the trajectory:

        vmd dump.lammpstrj
        vmd> pbc box -center origin
        vmd> pbc wrap -center origin -compound fragment -all

Plot density (column 12), temperature (column 9), energies, etc:

        gnuplot
        gnuplot> plot 'log.lammps' u 1:12 w lp

**Great! You just ran a LAMMPS simulation!**

At this point you could perform sufficiently long equilibration (2 ns) and production runs (5 ns) with the CL&P force field, which may be quite useful work to study ionic liquid systems.

However, fixed-charge force fields have some issues for ionic fluids: they tend to give too slow dynamics, leading to high viscosities when compared to experiment. Structural quantities, such as radial distribution functions (related to the structure factor measured by X-rays or neutrons), are usually well predicted.

---

## 5 Add explicit polarization terms

Adding explicit polarization terms involves, in this case, adding Drude particles to atoms (except H) so that induced dipoles appear as a response to the local electrostatic field. These terms are added to a non-polarizable system prepared following the procedure outlined in section 3.

The Drude particles (DP) represent how the canter of the electron clouds is displaced away from the nuclei. They are bonded to their atoms (the Drude cores, DC) by a harmonic potential with equilibrium distance 0, so besides the new particles, new bonds have to be created in the systems.

The Drude particles carry a small mass so they can be handled by the integrator. The motion of the Drude particles with respect to their cores is handled by a separate, specialized thermostat, keeping the Drude degrees of freedom cold, which results in a trajectory that follows closely the one that would have the relaxed Drudes (which would require a slow iterative procedure).

There are additional details, such as special exclusion lists for the Coulomb interactions and screening functions (Thole damping) to improve the description of electrostatics at short range.

The tools needed to prepare a polarizable system are in the folder `clandpol`.

The main physical parameter required for the Drude induced dipoles is the atomic polarizability, which determines the charges in the induced dipoles. These are collected in the file `alpha.ff`.

        cp ~/sim/clandpol/alpha.ff .

The `polarizer` script adds Drude dipoles to a LAMMPS `data.lmp` file:

        ~/sim/clandpol/polarizer -f alpha.ff data.lmp data-p.lmp

The new DP and their bonds to the DC are added into `data-p.lmp`, and the electrostatic charges are modified accordingly. **Study this file.**

>It is also possible to start from a system equilibrated with the non-polarizable force field, but for that it is necessary to add the atom labels as comments in the `Masses` section (LAMMPS doesn't write those).

LAMMPS commands to handle the Drude induced dipoles are written to `in-drude.lmp`. These are to be merged with `in.lmp` to produce a new input stack, `in-p.lmp`, for the polarizable simulation.

> **Task:** Try to merge the `in.lmp` and the `in-drude.lmp` files into a new `in-p.lmp`.
>- Pay attention to: `pair_style hybrid/overlay`
>- Don't create initial velocities for DP: `velocity ATOMS create ${TK} 12345`
:
## 6 Adapt the LJ potentials to account for explicit polarization

There is one last step: to scale down the Lennard-Jones potentials to account for the inclusion of explicit polarization terms. This is a fragment-based procedure. The fragments composing our ions are `c2c1im` for the imidazolium ring, `C4H10` for the side chain, and `ntf2` for the anion.

        cp ~/sim/clandpol/fragment.ff .
        cp ~/sim/clandpol/fragment_topologies/c2c1im.zmat .
        cp ~/sim/clandpol/fragment_topologies/C4H10.zmat .
        cp ~/sim/clandpol/fragment_topologies/ntf2.zmat .

then prepare a simple file attributing the atom indices to the fragments, named `fragment.inp`:

        # name  indices
        c2c1im  1:9
        C4H10  10:12
        ntf2   13:17

Then run the `scaleLJ` script:

        ~/sim/clandpol/scaleLJ -i fragment.inp -s -ip pair.lmp -op pair-sc.lmp

this generates a `pair-sc.lmp` file with scaled LJ parameters (signaled by `~` and `*`).

In `in-p.lmp` change `include pair-p.lmp` to `include pair-sc.lmp`.

**Excellent! You are ready to launch a polarizable simulation!**

---

## 7. Run a polarizable simulation

Run a test trajectory of 10000 steps printing every 100:

        mpirun -np 16 $LMP/lmp -in in-p.lmp > out-p.lmp &

It is highly likely you'll need to add an option to the `read_data` command in `in-p.lmp`:

        read_data data-p.lmp extra/special/per/atom 6

LAMMPS will exit with a message telling you this.

Follow the progress of the run:

        tail -f out-p.lmp

paying attention to the temperatures of the center of mass of molecules, of the intramolecular degrees of freedom, and of the Drude degrees of freedom, printed by the Drude integrator, `fix tgnh/npt`:

        f_TSTAT[1] f_TSTAT[2] f_TSTAT[3]

The first two should be close to the temperature of the thermostat and the third around 1 K.

**Fantastic! You are running MD with a polarizable force field!**

---

## 8.  Calculate structural and dynamic quantities

Sometimes, one includes calculations of diffusion coefficients or radial distribution functions in the `in-p.lmp`. Other times, one uses previously generated trajectories and calculates quantities from there, using the LAMMPS `rerun` command, which is fast. For that, prepare a `in-rerun.lmp`:

        units real
        boundary p p p

        atom_style full
        bond_style harmonic
        angle_style harmonic
        dihedral_style opls

        read_data data.eq.lmp

        # mean-squared displacements
        group cat molecule 1:200
        group ani molecule 201:400
        compute MSDcat cat msd
        compute MSDani ani msd

        # radial distribution functions NA-NA N-N NA-O CT-CT
        comm_modify cutoff 22.0
        neigh_modify one 3000
        compute RDF all rdf 200 1 1 16 16 1 17 12 12 cutoff 20.0
        fix RDF all ave/time 2000 1000 2000000 c_RDF[*] file rdf.lmp mode vector

        thermo 2000
        thermo_style custom step c_MSDcat[4] c_MSDani[4]

        rerun dump.lammpstrj dump x y z box yes

These commands specify calculations of diffusion coefficients (`compute MSD`) and radial distribution functions (`compute rdf`) and then the configurations are read from the `dump` file via the `rerun` command.

Backup your previous `log.lammps` and run the calculations:

        mv log.lammps log-run.lammps
        mpirun -np 16 $LMP/lmp -in in-rerun.lmp > out-rerun.lmp &

Plot the cation and anion diffusion coefficients:

        gnuplot
        gnuplot> plot 'log.lammps' u 1:2 w l
        gnuplot> replot 'log.lammps' u 1:3 w l

Plot radial distribution functions between NA and CT atoms from the cations, and N and O atoms from the anions:

        gnuplot
        gnuplot> plot 'rdf.lmp' u 2:3 w l t 'NA-NA'
        gnuplot> replot 'rdf.lmp' u 2:5 w l t 'NA-N'
        gnuplot> replot 'rdf.lmp' u 2:7 w l t 'NA-O'
        gnuplot> replot 'rdf.lmp' u 2:9 w l t 'CT-CT'

**Congratulations for completing the tutorial!**

You've learned the typical workflow involved in polarizable molecular dynamics simulations of ionic liquids.


---

Feedback to improve this tutorial is appreciated.
