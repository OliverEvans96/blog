## Intro to LAMMPS

This post is meant as a total beginner's intro to LAMMPS, the moledular dynamics simulation software. It's been a few years since I've worked on this, but I remember how overwhelming it felt trying to learn about LAMMPS and piece together all of the relevant information to actually be able to run a simulation. 

## Resources

I've found the [official LAMMPS docs](https://lammps.sandia.gov/doc/Manual.html) to be quite good. It's a bit overwhelming at first and you might have to read through it multiple times, but I think you'll find it to be well written. 

Also, the examples ([docs](https://lammps.sandia.gov/doc/Examples.html), [code](https://github.com/lammps/lammps/tree/master/examples)) are very helpful to gain a practical example of how to run a simulation.

If you look at the [peptide example](https://github.com/lammps/lammps/tree/master/examples/peptide), you'll find two important files: the **input script** ([`in.peptide`](https://github.com/lammps/lammps/blob/master/examples/peptide/in.peptide)) and the **data file** ([`data.peptide`](https://github.com/lammps/lammps/blob/master/examples/peptide/data.peptide)) [^1].

## Overview

The input script contains your instructions for LAMMPS describing all the details of your simulation. On [Line 13](https://github.com/lammps/lammps/blob/master/examples/peptide/in.peptide#L13), you'll see `read_data data.peptide`, which loads the data file, which contains the initial positions, bonds, etc. of your atoms. So the input script describes **how** to run the simulation, and the data file describes **what** you're simulating.

## Input Script

Other commands in the peptide example input script:
- [L3-4](https://github.com/lammps/lammps/blob/master/examples/peptide/in.peptide#L3-L4) describes the physical units for all values and some details about the atom model.
- [L6-10](https://github.com/lammps/lammps/blob/master/examples/peptide/in.peptide#L6-L10) describes what functions are used to calculate the interaction between atoms (see e.g. [pair_style lj/charmm/coul/charmm](https://lammps.sandia.gov/doc/pair_charmm.html), which describes a specific version of the **important [Lennard-Jones potential](https://en.wikipedia.org/wiki/Lennard-Jones_potential)**)
- [L15-16](https://github.com/lammps/lammps/blob/master/examples/peptide/in.peptide#L15-L16) describes the [neighbor list](https://en.wikipedia.org/wiki/Verlet_list), which allows the simulation to run much faster by ignoring interactions between atoms that are too far apart, and only updating the list periodically.
- [L18](https://github.com/lammps/lammps/blob/master/examples/peptide/in.peptide#L18) specifies the time precision of the simulation. Smaller time steps make the simulation more accurate, but takes longer to finish.
- [L41](https://github.com/lammps/lammps/blob/master/examples/peptide/in.peptide#L41) specifies the number of time steps to run, basically the time duration of your simulation
- [L20-24](https://github.com/lammps/lammps/blob/master/examples/peptide/in.peptide#L20-L24) are perhaps the most complicated but important. These determine the thermodynamics of your simulation, which describe the macroscopic conditions your simulation exists within. I was lucky that my professor set this for me and I didn't have to worry about it. But if you have to set these parameters yourself, you'll have to study [statistical mechanics](en.wikipedia.org/wiki/Statistical_mechanics) to determine what kind of ensemble you need.
- [L28-36](https://github.com/lammps/lammps/blob/master/examples/peptide/in.peptide#L28-L36) determine what kind of files LAMMPS should produce as it runs. The important one is [L28](https://github.com/lammps/lammps/blob/master/examples/peptide/in.peptide#L28), which specifies that the positions and velocities of all atoms should be written to a text file (`dump.peptide`) every 10 timesteps. The **dump file** (a.k.a `.lammpstrj`) is the final output of the simulation that you will later analyze. 

You might also find [`write_restart`](https://lammps.sandia.gov/doc/write_restart.html), which is like a checkpoint in case you need to stop and resume your simulation. This is important since they can take days or weeks to complete. A restart file generally contains more information than the dump file, and is stored as a binary file rather than a text file, so it's not suitable for analyzing your simulation as you would analyze the dump file to gain insight from your simulation.

You can find the full input file reference [here](https://lammps.sandia.gov/doc/Commands_input.html).

## Data file

The data file describes the atoms, their interactions, and their initial positions. The most common interactions are bonds (2 atoms apply forces on each other as a function of distance) and angles (forces applied on three atoms connected by two bonds as a function of the angle between the bonds). You can see both depicted here:
<img width="300" src="https://haygot.s3.amazonaws.com/questions/733811_668877_ans_eb092412db5141c3ac60aa7b99434c0c.png" />

There are also other interactions, including dihedrals and impropers, which act on groups of more than 3 atoms.

Note that these interactions (angles, bonds, dihedrals, impropers) are **permanent and unbreakable**. If a bond exists between two specific atoms, it will continue to exist for the whole simulation. Also keep in mind that a bond does not *constrain* two atoms to always be an exact distance apart. Rather, it *strongly encourages* them to maintain that distance by applying forces on them.

However, there are also **transient** interactions between atoms, primarily the pairwise electrostatic interaction described the Lennard Jones potential specified in the `pair_style` command in the input script. This force is only applied between two atoms if they are close enough to be considered neighbors on the neighbor list. If they move away from each other during the simulation, they will no longer interact electrostatically.

Anyway, back to the data file.

### Header

- The first section of the file ([L1-13](https://github.com/lammps/lammps/blob/master/examples/peptide/data.peptide#L1-L13) in `data.peptide`) describes the contents of the file. These should match the number of entries in each section below.
- [L15-17](https://github.com/lammps/lammps/blob/master/examples/peptide/data.peptide#L15-L17) describes the bounding box of the simulation. The universe is huge, and the bounding box describes the tiny part of it that you're simulating. The boundary conditions can be set in the input file via the [`boundary`](https://lammps.sandia.gov/doc/boundary.html) command. You might use a periodic box to simulate a large quantity of something, or a fixed boundary (hard wall) in other situations.

### Atom sections

Every atom has a `type`, which generally corresponds to an element. So all of your oxygen atoms might be type 1, and your hydrogen atoms might be type 2. However, if you have oxygen atoms in both your droplet and your substrate, you might choose to give them different types to capture the fundamentally different way they interact with other atoms.

In the `Masses` section ([L19-34](https://github.com/lammps/lammps/blob/master/examples/peptide/data.peptide#L19-L34)), there are two columns: the first one is the type, and the second one is the mass. So e.g. every atom of type 1 has a mass of `12.0110` (in whatever units are specified by the [`units`](https://lammps.sandia.gov/doc/units.html) command in the input script).

In the `Atoms` section ([L137-2142](https://github.com/lammps/lammps/blob/master/examples/peptide/data.peptide#L137-L2142)), each individual atom is described.

### Coef sections


### Interaction sections

You can read the full data file reference [here](https://lammps.sandia.gov/doc/read_data.html).

## Visualization

You can see that in the peptide input script, they are also dumping `.jpg` and `.mpg` files. I don't recommend this. LAMMPS is great for simulation, but it produces very ugly images/movies. For visualization, use [VMD](http://www.ks.uiuc.edu/Research/vmd/), which you can use interactively or automatically to create beautiful images like this:

<img width="500" src="https://www.ks.uiuc.edu/Research/vmd/minitutorials/tachyonao/vmd-hiv1capsid.jpg" />

VMD can load your LAMMS dump file directly using the [`readlammpsdata` command](https://www.ks.uiuc.edu/Research/vmd/plugins/topotools/#TOC-readlammpsdata-file-name-atom-style).

[^1]: For some strange reason, LAMMPS files tend to be named with the file extension *first* as opposed to the normal convention of putting the extension at the end. It's as if you had named a photo `jpg.myface` instead of `myface.jpg`. This is totally optional, the file extension is only a hint to tell the user what the file contains. You can name your input and data files however you want, and as long as you use the same name in your code, it will work fine.
