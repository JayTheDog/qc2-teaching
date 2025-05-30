SCF program
===========

The main objective of this course is to write a working restricted Hartree--Fock
program.

.. contents::


Restricted Hartree--Fock
------------------------

The first task in this course is to code a working restricted Hartree--Fock
program.

Downloading the Material
~~~~~~~~~~~~~~~~~~~~~~~~

Start by extracting the ``scf.zip``, which can found `here <https://github.com/grimme-lab/qc2-teaching/releases/latest>`_.
You should find a ``scf/`` directory containing the starting code for your
program (see ``scf/app/main.f90``).

You can just continue to use fpm to work on your program now.

Alternatively, to make building your program easier with just the Fortran 
compiler and give you more time to focus on the actual programming we put 
some build automation there as well.
To use it just invoke ``make`` and find everything else taken care for.

For more details on building software and how we automated this process
you can find a guide at the
`fortran-lang learning page <https://fortran-lang.org/learn/building_programs>`_.

Getting Input
~~~~~~~~~~~~~

First, we have to gather which information is important to perform a
Hartree--Fock calculation.
We need to know about the molecular geometry. There are several ways
to represent the geometry, the most simple is to use Cartesian coordinates
as a vector of *x*, *y* and *z* coordinates.
What maybe is not that obvious, we have to decide on a unit system for
the coordinates, usually we would follow IUPAC recommendations and use
SI units like meters, but if you think about a molecule on the length
scale in meters it becomes quite inconvenient.
To get into the right ballpark we could choose Ångström which has an
obvious relation to the SI unit, but here we will use Bohr which is
the length unit of the atomic units system.
Atomic units have the advantage that a lot of constants drop out of the
equations we want to code up, and we can convert them easily back,
in fact, we provide a conversion routine back to SI units for you.

These are the considerations we have to put in the geometry, now we have
to identify the atoms/nuclei in our molecule. Chemical identity like the
element is given by the nuclear charge and the number of electrons.
It is quite popular to specify both with the element symbol which
map to the atomic number (which corresponds to the nuclear charge and the
number of electrons of the neutral atom) since it is the intuitive choice.
We will not follow this approach since it would require you to work with
character type variables, instead we will separate the nuclear charge information
and the number of electrons. For each element, we will read the nuclear charge
in multiples of the electron charge (which is the atomic unit of charge)
and specify the number of electrons separately.

Having put some thoughts in the geometric representation of the system,
we now have to tackle the electronic representation, that is, we need a basis 
set to expand our wavefunction.
There are many possible choices, like atom centered basis functions
(Slater-type, Gaussian-type, ...) plain waves, wavelets, ...
This is one of the most important choices for every quantum chemistry code.
Usually, a single kind of functions is supported which is limiting the chemical
systems that can be calculated with the program in question.
The most common distinction is made between codes that support periodic boundary
conditions or not, while periodic boundary conditions are naturally included
in plain wave and wavelet based programs, extra effort has to be put into
codes using Gaussian-type basis functions to support this kind of calculation.
Also, most wavefunction centered programs use atom centered orbitals since the
resulting integrals are easier to solve.
Here, the exception to the rule is quite common in our field of research
and usually offers a unique competitive edge.

For writing your Hartree-Fock program, you do not have to bother with this
choice, because we already made it for you by providing the implementation
for integrals over Gaussian-type basis functions.
We will limit you here to spherical basis functions only, so you can concentrate
on coding the self-consistent field procedure and do not have to worry about
mapping shells to orbitals to basis functions to primitives.

However, you require a mapping from basis functions to atoms, i.e., which basis
function belongs to which atom. This mapping is required later for the 
calculation of the integrals, but the best place to create this mapping is
when reading the input file.



We will start with the input for the dihydrogen molecule in a minimal basis set.
With the input format we provide, the geometrical structure of the system
and the basis set are tied together.

.. code-block:: bash
   :caption: h2.in
   :linenos:

   2 2 2
   0.0  0.0 -0.7  1.0  1
   1.20
   0.0  0.0  0.7  1.0  1
   1.20

This input contains the information necessary to code up your program,
we start with the first line, which contains the number of atoms,
the number electrons and the total number of basis functions as integer
values.
None of this information is necessary to read the input file, but is
included for convenience, *e.g.*, such that you can allocate memory before
starting to read the rest of the file.

Starting from the second line we expect a tuple of the three Cartesian
coordinates in Bohr, the nuclear charges in multiples of electron charge
and the number of basis functions this particular atom.
In the lines after position and identity, we find the exponents of our basis
functions, the number of lines following corresponds to the number of
basis functions for this particular atom.

.. tabs::

   .. tab:: make
   
      .. important::
   
         You can compile your program using
   
         .. code-block:: text
   
            make
            ...
            gfortran ... -o ./scf
   
         The resulting binary will be placed in the current working directory.
         Run it with
   
         .. code-block:: text
   
            ./scf
            Here could start a Hartree-Fock calculation
   
         The starting code provides the possibility to pass the input file
         as command line argument to your program with
   
         .. code-block:: text
   
            ./scf molecules/h2.in
            Here could start a Hartree-Fock calculation
   
         The input file will always be opened to the ``input`` unit in ``app/main.f90``.
   
   .. tab:: fpm
   
      .. important::
   
         The starting code is already setup as fpm project.
         The package manifest should contain the following content
   
         .. literalinclude:: ../scf/fpm.toml
            :caption: fpm.toml
            :language: toml
   
         You can run your program with
   
         .. code-block:: text
   
            fpm run
            ...
             + build/gfortran_debug/app/scf
            Here could start a Hartree-Fock calculation
   
         The starting code provides the possibility to pass the input file
         as command line argument to your program with
   
         .. code-block:: text
   
            fpm run -- molecules/h2.in
            ...
             + build/gfortran_debug/app/scf "molecules/h2.in"
            Here could start a Hartree-Fock calculation
   
         The input file will always be opened to the ``input`` unit in ``app/main.f90``.


.. admonition:: Exercise 1

   Before you start coding the input reader for this format, try
   to write with the specifications inputs for the following systems:

   1. A helium atom in a double zeta basis set with exponents 2.5 and 1.0.
   2. A helium-hydrogen cation with a bond distance of 2 Bohr in a minimal
      basis set on both atoms and an exponent of 1.25 for each atom.

.. admonition:: Solutions 1
   :class: tip, toggle

   For a single atom, the choice of the position is unimportant, so we
   can write something like

   .. code-block:: bash
      :caption: he.in
      :linenos:

      1 2 2
      0.0  0.0  0.0  2.0  2
      2.5
      1.0

   The helium-hydrogen cation looks similar to the dihydrogen, except for
   the changed nuclear charge.

   .. code-block:: bash
      :caption: heh+.in
      :linenos:

      2 2 2
      0.0  0.0 -1.0  2.0  1
      1.25
      0.0  0.0  1.0  1.0  1
      1.25

.. admonition:: Exercise 2

   1. Code an input reader that can read all the provided input files.
   2. Make sure the input file is read correctly by printing all data read
      to the terminal.

.. note::

    For printing matrices and vectors, we provide convenience functions.

    .. code-block:: fortran

       call write_matrix(matrix, name="Matrix")
       call write_vector(vector, name="Vector")

    We highly recommend using these functions instead of ``write(*,*)`` to
    print your data, as they provide a more structured output (for Fortran's
    column-major order) and are easier to read.

.. admonition:: Exercise 3
  
    Create a mapping between basis functions and atoms. You need to store the 
    information about which basis function belongs to which atom. This is
    necessary, because when looping over the basis functions during the
    integral calculation, you require the aufpunkt of each basis function, which
    is equivalent to the atom position. Hence, the mapping should give you the 
    atom index if the basis function index is provided. 

    1. What is the optimal (minimal) layout for saving the mapping? Consider
       which information is given (loop over basis functions) and which
       information is needed (aufpunkt).
    2. How does the mapping look like for LiH?

Classical Contributions
~~~~~~~~~~~~~~~~~~~~~~~

First, we start by computing the classical nuclear repulsion energy, *i.e.*,
the Coulomb energy between the nuclei.

.. admonition:: Exercise 3

   1. Which data is needed in the computation?
   2. Code up the nuclear repulsion energy in a separate ``subroutine``.
      Write the resulting energy in a meaningful way to the terminal.
   3. Evaluate the Coulomb-law for the dihydrogen molecule at 1.4 Bohr
      distance and compare it with your program.
      Can you run your ``subroutine`` multiple times and get the
      same result? If not recheck your code.
   4. Check what happens if you calculate the nuclear repulsion energy for
      a single atom. Do you get the expected result?
   5. Is there a (intrinsic) Fortran function that simplifies the calculation? 

Classical contributions to the total energy do not dependent on
the density or wavefunction and can already be calculated before
starting with the self-consistent field procedure.
It is usually a good idea to evaluate these contributions first,
because they are less expensive to compute and, for the purpose
of this course, easier to implement.

Basis Set Setup
~~~~~~~~~~~~~~~

This Hartree-Fock program will use contracted Gaussian-type orbitals to
expand the molecular orbitals in atomic orbitals.
We will use an STO-6G basis set, *i.e.*, we use the best representation of a
Slater-type orbital by six primitive Gaussian-type orbitals.

This is the first time you will use an external library function, therefore
we will clarify to you how to read and use an ``interface``.
In your program, you will call a provided ``subroutine`` to perform
the expansion from the Slater orbital to six primitive Gaussian-type orbitals.

The final call in your program might look somewhat similar to this:

.. code-block:: fortran

   call expand_slater(ng, zeta, exponents, coefficients)

To understand why the ``subroutine expand_slater`` takes four arguments,
we look up its ``interface``:

.. code-block:: fortran

   interface
   subroutine expand_slater(ng, zeta, exponents, coefficients)
   import wp
   integer,  intent(in)  :: ng   !< number of primitive Gaussian functions
   real(wp), intent(in)  :: zeta !< exponent of slater function
   real(wp), intent(out) :: exponents(:)    !< of primitive Gaussian functions
   real(wp), intent(out) :: coefficients(:) !< of primitive Gaussian functions
   end subroutine expand_slater
   end interface

An ``interface`` provides the necessary information on how to
invoke a ``subroutine`` or ``function`` without concerning
you with the implementation details.

.. note::

    For programmers coming from C or C++, it is similar to an ``extern``
    declaration in a header file for a function.

Usually, you do not have to write an ``interface`` since they
are conveniently created and handled for you by your compiler.

.. admonition:: Exercise 4

   1. Create a ``parameter`` storing the information about the
      number of primitive Gaussian functions you want to expand your Slater
      function in.
   2. What is the optimal layout for saving the exponents and coefficients
      of the primitive Gaussian functions? Which quantities from the input
      do you need to determine the amount of memory to store them?
   3. Allocate enough space to store all the primitive exponents and coefficients
      from the expansion for calculating the integrals later.
   4. Loop over all basis functions, perform the expansion for each and save the
      resulting primitive Gaussians to the respective arrays.

One-Electron Integrals
~~~~~~~~~~~~~~~~~~~~~~

Note that the chosen basis set is very simple, we only allow
spherical basis function (*s*-functions), also the contraction depth of
each function is the same. Usually, one would choose more sophisticated
basis sets for quantitative calculations, but the basic principle remains
the same.

Integral calculations can quickly be very obscure depending on the way
a basis set is stored and mainly handle implementation-specific details
of the program to perform the integral evaluation in some clever way.
We use a simple basis set here to teach you the basic principle of
integral evaluation.

We start with the simple one-electron integrals. For Hartree-Fock, we need
two-center overlap integrals, two-center kinetic energy integrals and
three-center nuclear attraction integrals.
To make things easier, we provide the implementation for all three integrals
over contracted Gaussian orbitals, let's check out the ``interface``:

.. code-block:: fortran

   interface
   !> one electron integrals over spherical Gaussian functions
   subroutine oneint(xyz, chrg, r_a, r_b, alp, bet, ca, cb, sab, tab, vab)
      real(wp), intent(in)  :: xyz(:, :) !< position of all atoms in atomic units, dim: [3, nat]
      real(wp), intent(in)  :: chrg(:) !< nuclear charges, dim: nat
      real(wp), intent(in)  :: r_a(:) !< aufpunkt of orbital a, dim: 3
      real(wp), intent(in)  :: r_b(:) !< aufpunkt of orbital b, dim: 3
      real(wp), intent(in)  :: alp(:) !< Gaussian exponents of the primitives at a
      real(wp), intent(in)  :: bet(:) !< Gaussian exponents of the primitives at b
      real(wp), intent(in)  :: ca(:) !< contraction coeffients of primitives at a
      real(wp), intent(in)  :: cb(:) !< contraction coeffients of primitives at b
      real(wp), intent(out) :: sab !< overlap integral <a|b>
      real(wp), intent(out) :: tab !< kinetic energy integral <a|T|b>
      real(wp), intent(out) :: vab !< nuclear attraction integrals <a|Σ z/r|b>
   end subroutine oneint
   end interface

The most important information is that we need *two* centers for the calculation,
meaning we have to implement it as a loop over all orbital pairs (= pairs of basis
functions).
Additionally, we have to account for the different resolutions of the inputs of the
subroutine ``oneint``: While the aufpunkte are atom-resolved, the exponents, contraction
coefficients and integrals are orbital-resolved (relate to the basis functions).


.. admonition:: Exercise 5

   1. Which matrices can you compute from the one-electron integrals?
   2. Allocate space for the necessary matrices.
   3. How can you map between atom- and orbital-resolved variables?
   4. Loop over all pairs and calculate all the matrix elements.

.. admonition:: Exercise 6

   Symmetric matrices, like the overlap, can be stored in two ways,
   as full *N×N* matrix with ``dimension(n,n)`` or as packed matrix
   in a one-dimensional array with ``dimension(n*(1+n)/2)``, like:

   .. math::

      \begin{pmatrix}
      1 & 2 & 4 & 7 & \cdots \\
        & 3 & 5 & 8 & \\
        &   & 6 & 9 & \\
        &   &   & 10& \\
      \vdots & & & & \ddots \\
      \end{pmatrix}
      \Leftrightarrow
      \begin{pmatrix}
      1 \\ 2 \\ 3 \\ 4 \\ \vdots \\
      \end{pmatrix}
      \qquad
      \begin{pmatrix}
      a_{11} & a_{12} & a_{13} & a_{14} & \cdots \\
             & a_{22} & a_{23} & a_{24} & \\
             &        & a_{33} & a_{34} & \\
             &        &        & a_{44} & \\
      \vdots &        &        &        & \ddots \\
      \end{pmatrix}
      \Leftrightarrow
      \begin{pmatrix}
      a_{11} \\ a_{12} \\ a_{22} \\ a_{13} \\ \vdots \\
      \end{pmatrix}

   1. Check if the matrices you have calculated are symmetric.
   2. Pack all symmetric matrices.

   The eigenvalue solver provided can only work with symmetric packed matrices,
   therefore you should have an unpacked to packed conversion routine ready for
   the next exercise.

   .. note::

      The ``ẁrite_matrix`` subroutine can be used to print packed matrices as 
      well.

Symmetric Orthonormalizer
~~~~~~~~~~~~~~~~~~~~~~~~~

The eigenvalue problem we have to solve is a general one given by

.. math::
   \mathbf F \mathbf C = \mathbf S \mathbf C \boldsymbol\varepsilon

where **F** is the Fock matrix, **C** is the matrix of the eigenvectors,
**S** is the overlap matrix and **ε** is a diagonal matrix of the eigenvalues.
While this is in principle possible with a more elaborated solver routine,
we want to solve the following problem instead:

.. math::
   \mathbf F^\prime \mathbf C^\prime = \mathbf C^\prime \boldsymbol\varepsilon

For this reason, we have to find a transformation from **F** to **F'**
and back from **C'** to **C**.
We choose the Loewdin orthonormalization or symmetric orthonormalization
by calculating **X** = **S**:sup:`-1/2`, to invert (or perform any operation
with) a matrix we have to diagonalize it first and perform the operation on the
eigenvalues **s**.

.. math::
   \mathbf X = \mathbf S^{-1/2} = \mathbf U \mathbf s^{-1/2} \mathbf U^\top
   \qquad
   \text{where}
   \quad
   \mathbf s = \mathbf U^\top \mathbf S \mathbf U

.. admonition:: Exercise 7

   1. Use **F'** = **X**:sup:`T` **FX** and **C** = **XC'** to show that the
      general eigenvalue problem and the transformed one are equivalent.
   2. Write a ``subroutine`` to calculate the symmetric orthonormalizer
      **X** from the overlap matrix.
   3. Make sure that **X**:sup:`T` **SX** = **1** and that you are not overwriting
      your overlap matrix in the diagonalization.
   4. Do you see why **X** is called symmetric orthonormalizer?

To solve the actual eigenvalue problem we will use the linear algebra package
(LAPACK), namely the subroutine ``dspev``, which stands for double precision,
symmetric packed eigenvalue problem, for more details you can look up the
`documentation <http://www.netlib.org/lapack/explore-html/d4/d0f/dspev_8f.html>`_.
Since LAPACK routines can be somewhat unintuitive to work with on the first
encounter we provide a wrapper called ``solve_spev``.
As usual we checkout the interface in ``src/linear_algebra.f90``:

.. code-block:: fortran

   interface
   subroutine solve_spev(matrix, eigval, eigvec, stat)
     import wp
     !> symmetric packed matrix to diagonalize, dim: n*(n+1)/2
     !  matrix is overwritten by the LAPACK solver
     real(wp), intent(inout) :: matrix(:)
     !> contains eigenvalues on exit, dim: n
     real(wp), intent(out) :: eigval(:)
     !> contains eigenvectors on exit, dim: [n, n]
     real(wp), intent(out) :: eigvec(:, :)
     !> error status, if not provided the routine will stop the program
     integer, intent(out), optional :: stat
   end subroutine solve_spev
   end interface

.. note::

   The linear algebra package (LAPACK) and the basic linear algebra subprograms
   (BLAS) are commonly used libraries in many quantum chemical programs,
   as they provide a computationally efficient and uniform interface to many
   linear algebra problems.


Initial Guess
~~~~~~~~~~~~~

There are now two possible ways to initialize your density matrix **P**.

1. Provide a guess for the orbitals
2. Provide a guess for the Hamiltonian

If you happen to have converged orbitals around, the first method would be
the most suitable.
Alternatively, you can provide a model Hamiltonian, usually a tight-binding
or extended Hückel Hamiltonian is used here.
The simplest possible model Hamiltonian we have readily available is the
core Hamiltonian **H**:sub:`0` = **T** + **V**.

The initial density matrix **P** is obtained from the orbital coefficients by

.. math::
   \mathbf P = \mathbf C\ \mathbf n_\text{occ} \mathbf C^\top

where **n**:sub:`occ` is a diagonal matrix with the occupation numbers of the
orbitals.
For the restricted Hartree-Fock case *n*:sub:`*i*,occ` is two for occupied orbitals
and zero for virtual orbitals.

.. admonition:: Exercise 8

   1. By using **F** = **H**:sub:`0` as an initial guess for the Fock matrix we
      effectively set the density matrix **P** to zero.
      What does this mean from a physical point of view?
   2. Add the initial guess to your program.
   3. Diagonalize the initial Fock matrix (use the symmetric orthonormalizer)
      to obtain a set of guess orbital coefficients **C**.
   4. Calculate the initial density matrix **P** resulting from those orbital
      coefficients.

Hartree-Fock Energy
~~~~~~~~~~~~~~~~~~~

The Hartree-Fock energy is given by

.. math::
    E_\text{HF} = \frac12\mathrm{Tr}\{(\mathbf H_0 + \mathbf F) \mathbf P\}

where Tr denotes the trace.

.. admonition:: Exercise 9

   You should have now an initial Fock matrix **F** and an initial density
   matrix **P**.

   1. Write a ``subroutine`` to calculate the Hartree-Fock energy.
   2. Calculate the initial Hartree-Fock energy.

Two-Electron Integrals
~~~~~~~~~~~~~~~~~~~~~~

We have ignored the two-electron integrals for a while now, up to now
they were not important, but we will now need to calculate them to
perform a self-consistent field procedure.
The four-center two-electron integrals are the most expensive quantity
in every Hartree-Fock calculation and there exist many clever algorithms
to avoid calculating them all together, we will again go the straight-forward
way and calculate them the naive way.

Again we will check the ``interface`` of the ``twoint`` routine
in ``src/integrals.f90``.

.. admonition:: Exercise 10

   1. Allocate enough space to store the two-electron integrals.
   2. Create a (nested) loop to calculate all two-electron integrals.

.. admonition:: Exercise 11

   The four-center integrals have an eight-fold symmetry relation we want to use
   when calculating such an expensive integral to speed up the calculation.
   Rewrite your loops to only calculate the unique integrals:

   .. math::
      (\mu\nu|\kappa\lambda) =
      (\nu\mu|\kappa\lambda) =
      (\mu\nu|\lambda\kappa) =
      (\nu\mu|\lambda\kappa) =
      (\kappa\lambda|\mu\nu) =
      (\kappa\lambda|\nu\mu) =
      (\lambda\kappa|\mu\nu) =
      (\lambda\kappa|\nu\mu)

   1. Write down all the indices for the four-center integrals and figure out
      unique relation between the indices (it is similar to packing matrices).
   2. Implement this eight-fold-symmetry in your calculation.
   3. You can also pack the two-electron integrals like you packed your matrices
      (you need to pack them three times, the bra and the ket separately,
      and then the packed bra and ket again).
      Note that this will make it later more difficult to access the values 
      again, therefore it is optional.

Self Consistent Field Procedure
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now you have everything together to build the self-consistent field loop.
Remember, the Fock matrix in terms of the two-electron integrals is given by

.. math::
   F_{\mu\nu} = H_{\mu\nu} + \sum_\lambda \sum_\kappa \left( 
   P_{\lambda\kappa} \cdot (\mu\nu|\kappa\lambda)
   - \frac12 P_{\lambda\kappa} \cdot (\mu\lambda|\kappa\nu) \right)
   
Make sure your definition of the density matrix matches the definition of the
Fockian.

.. admonition:: Exercise 12

   1. First, you need to construct a new Fock matrix from the density matrix.
   2. Construct a loop that performs your self-consistent field calculation.
   3. Copy or move (whatever you find more appropriate) the necessary steps
      inside the SCF loop.
   4. Define a convergence criterion using a conditional statement (``if``)
      based on the energy change and/or the change in the density matrix
      and ``exit`` the SCF loop.
   5. Compare your final HF energies with

      ============= ========================
       Input          E(RHF) / E\ :sub:`h`
      ============= ========================
       H\ :sub:`2`    --1.127785613
       He             --2.860251227
       Be            --14.568567143
      ============= ========================

   6. Calculate the proton affinity of H\ :sub:`2`.
      H\ :sub:`3`\ :sup:`+` has a D\ :sub:`3h` structure with
      *R*\ :sub:`HH` = 1.7 Bohr

Dissociation Curves
~~~~~~~~~~~~~~~~~~~

To calculate a dissociation curve we could either implement this functionality
in the program itself or use an external script to automatically write input
files and read the program output to collect the necessary data.
Since this approach is common for computional chemistry, we will use a scripting
approach here as well.

Modify the provided ``bash`` script to calculate a dissociation curve:

.. code-block:: bash
   :caption: h2-diss.bash

   #!/usr/bin/env bash

   # stop on errors
   set -eu

   # Ensure non-localized printout
   export LC_NUMERIC=en_US.UTF-8

   # put the name of your program here ("./scf" or "fpm run --")
   program=echo
   # unique pattern to find the final energy
   pattern='final SCF energy'
   # output file for plotting
   datafile=plot.dat

   # scan distances
   start_distance=1.4
   last_distance=5.0
   step=0.1

   read -r -d 'END' input <<-EOF
      2 2 2
      0.0  0.0   0.0  1.0  1
      1.20
      0.0  0.0  DIST  1.0  1
      1.20
      END
   EOF

   tmpinp=temporary.inp
   tmpout=temporary.out

   # cleanup
   [ -f $datafile ] && rm -v $datafile

   steps=$(seq $start_distance $step $last_distance | wc -l)
   printf "Scanning from %.3f Bohr to %.3f Bohr in %d steps\n" \
      $start_distance $last_distance $steps

   for distance in $(seq $start_distance $step $last_distance | sed s/,/./)
   do
      # generate the input file
      echo "$input" | sed s/DIST/$distance/ > $tmpinp
      # perform the actual calculation on the input file
      2>&1 $program $tmpinp > $tmpout
      # get the energy from the program output
      energy=$(grep "$pattern" $tmpout | awk '{printf "%f",$(NF)}' | tail -1)
      # if there is no energy to be found, we complain
      if [ -z "$energy" ]
         then
            1>&2 printf "ERROR!\n"
            1>&2 printf "'%s' cannot be found in '%s' output\n" "$pattern" "$program"
            1>&2 printf "please inspect '%s' and '%s'\n" "$tmpinp" "$tmpout"
            exit 1
      fi
      # otherwise we write to the logfile
      printf "Current energy is %.8f Hartree for distance %.3f Bohr\n" \
         $energy $distance
      printf "%8.3f %12.8f\n" $distance $energy >> $datafile
   done

   # cleanup
   [ -f $tmpinp ] && rm $tmpinp
   [ -f $tmpout ] && rm $tmpout


.. admonition:: Exercise 13

   1. The above script only contains dummies and can be executed without performing
      a calculation. Perform such a dry-run to understand how the script is working.
   2. Modify it to match your program and plot the resulting dissociation curve.
   
.. note::

   In case the script exits an error message like

   .. code-block:: text

      script.bash: Zeile 34: printf: 1.4: Ungültige Zahl.
      script.bash: Zeile 34: printf: 5.0: Ungültige Zahl.
      Scanning from 1,000 Bohr to 5,000 Bohr in 37 steps
      
   your language settings might be set to something other then English (in this case German).
   To overwrite the localisation settings run the script with
   
   .. code-block:: text

      LC_NUMERIC=en_US.UTF-8 bash script.bash


Properties
----------

With a working RHF program we have access to a wavefunction and we can use it
to calculate certain properties.

Partial Charges
~~~~~~~~~~~~~~~

The total number of electrons in a system is given by

.. math::
   N_{el} = \sum_{\mu}^{N} \sum_{\nu}^{N} {P}_{\mu\nu} {S}_{\nu\mu}
   = \sum_{\mu}^{M} (\mathbf{PS})_{\mu\mu}.

If your SCF wavefunction does not return its input number of electrons, the
wavefunction is most certainly not properly normalized. In this case recheck
your SCF and integral implementation.

Based on the above formula, Mulliken concluded that the number of electrons
associated with a particular nucleus is equal to the number of electrons
associated with its basis functions. Thus the partial Mulliken charge of atom *A*
is defined as:

.. math::
   q_{A} = Z_{A} - \sum_{\mu \in A} (\mathbf{PS})_{\mu\mu}

.. admonition:: Exercise 14

   1. Code a subroutine to perform a Mulliken atomic population analysis
   2. Determine the Mulliken atomic partial charges in LiH using the input file
      provided.

Charge Density
~~~~~~~~~~~~~~

In terms of real contracted (Gaussian) basis functions, the electron density is
given by

.. math::
   \rho(\mathbf{r}) = \sum_{\mu} \sum_{\nu} {P}_{\mu \nu}
   \psi_{\mu}(\mathbf{r}) \psi_{\nu}(\mathbf{r}).

To calculate the product of the two basis functions the Gaussian product
theorem can be used

.. math::

   \phi(\alpha, \mathbf r - \mathbf R_A) \cdot
   \phi(\beta, \mathbf r - \mathbf R_B)
   = K_{AB} \cdot \phi\left(\alpha + \beta,
   \mathbf r - \frac{\alpha\mathbf R_A + \beta\mathbf R_B}{\alpha + \beta}\right)

where *K*\ :sub:`AB` is given by

.. math::

   K_{AB} = \left(\frac{2\alpha\beta}{(\alpha+\beta)\pi}\right)^{\frac34}
   \exp[-\alpha\beta/(\alpha+\beta)\cdot R^2_{AB}]

Note that the norming constants of the Gaussians have been absorbed into the
contraction coefficients already.

.. admonition:: Exercise 15

   1. Code a subroutine to calculate the electron density *ρ* at a given point
      in cartesian space.
   2. Calculate and plot the electron density along the axis connecting the bonds
      in H\ :sub:`2` and through the Be and He atoms. Plot them together.

Geometry Optimization
---------------------

The next task is to expand your program to perform a simple geometry 
optimization.

Numerical Derivatives
~~~~~~~~~~~~~~~~~~~~~

The simplest way to perform geometry optimizations is by using the information
from the energy derivative. As we do not want to code analytical derivatives
of the Hartree--Fock energy expression, we will resort to numerical derivatives
instead. This requires to evalulate several SCF energies in one program run.
Since you coded your SCF in a subroutine, this should not be an issue.
Nevertheless, try to run the subroutine several times to check if the code
is correctly allocating and initializing all variables. If errors show up now,
you can fix them before adding much more code to your program.

Copy and modify your input files such that for each parameter (cartesian atomic
coordinates and Slater exponents) *θ*\ :sub:`i`, it contains an integer
indicating whether a numerical gradient with respect to *θ*\ :sub:`i` is to be
calculated. Create a new subroutine that will allow the variation of one
parameter *θ*\ :sub:`i` at a time as necessary for each element of the
numerical gradient

.. math::
   \frac{\delta E}{\delta \theta_{i}} \approx \frac{E(\theta_{1}, \ldots, 
   \theta_{i} + \Delta \theta, \ldots, \theta_{n}) - E(\theta_{1}, \ldots, 
   \theta_{i} - \Delta \theta, \ldots, \theta_{n})}{2 \Delta \theta}.

Specifically, the geometric derivative is given by (leaving out all other 
parameters the energy depends on)

.. math::

   \frac{\delta E}{\delta R_{i}} \approx \frac{E(R_{i} + \Delta R) - E(R_{i} -
   \Delta R)}{2 \Delta R}.

.. admonition:: Exercise 16

   1. Code a numerical gradient that allows the calculation of the derivative of
      the HF energy with respect to atom positions and Slater exponents.
      Design your code such that the optimizations can be run separately, i.e.,
      use separate subroutines for the position gradient and the Slater 
      exponent gradient.
   2. Save your gradient components to arrays analogous to the ones for the atom
      positions and Slater exponents.
      We shall call the conceptual combination of them the gradient vector **g**.
   3. By definition, in which direction does a gradient point? How is this
      different from the force?

Steepest Descent
~~~~~~~~~~~~~~~~

Similar to the SCF, parameter optimizations are performed iteratively and
depend on a convergence criterion concerning energy and/or size of gradient.

Code a variant of the “steepest descent” optimization routine as given by:

.. math::

   \Theta^{k+1} = \Theta^{k} + \eta \mathbf{g}^{k}

*k* denotes the number of the optimization cycle, *Θ* is the parameter set for 
an iteration. Choose *η* to get reasonably smooth, fast convergence.

.. tip::
   
   Using a fixed *η* is a quite simple approach to optimization. Do not try to "over-optimize" *η*. There is no common best *η* for both optimizations.

.. admonition:: Exercise 17

   1. Optimize the geometry of HeH\ :sup:`+` in a minimal basis set.
   2. Optimize the Slater exponents of Be and H\ :sub:`2` (*R*:sub:`HH` =
      1.4 Bohr) in a full double-ζ basis set.
      Compare to energies for the estimated HF basis set limit:

      =========== ===================
      System      Energy/E\ :sub:`h`
      =========== ===================
      H\ :sub:`2`  --1.134
      Be          --14.573
      =========== ===================

      You won't reach these numbers, but how close do you get? What is missing?

    3. Using gradient descent with a fixed *η* 


Beyond RHF
----------

Next, we will work on extending restricted Hartree-Fock (RHF) to either 
unrestricted Hartree-Fock (UHF) or restricted second-order Møller--Plesset 
perturbation theory (MP2). Only one of these two exercises is mandatory.
Note that the UHF part also includes the calculation of the spin contamination.

Unrestricted Hartree-Fock
~~~~~~~~~~~~~~~~~~~~~~~~~

Copy and modify your RHF subroutine to create an UHF program. The input files will
need to be modified such that they contain the number of *α* and the number of
*β* electrons. By definition, the number of *α* electrons is bigger then the number
of *β* electrons. You can check the correctness of your results against the Li atom
(Slater exponents 3.5, 2.0, 0.7 and 0.3: E = --7.419629 E\ :sub:`h`) and the H
atom.

For UHF you use one set of **F**, **C** and **P** matrices for each spin to solve
the two eigenvalue problems

.. math::

   \mathbf{F}^{\alpha} \mathbf{C}^{\alpha} =
   \mathbf{S}\mathbf{C}^{\alpha}{\boldsymbol\varepsilon}^{\alpha}
   \quad \text{and} \quad
   \mathbf{F}^{\beta} \mathbf{C}^{\beta} =
   \mathbf{S}\mathbf{C}^{\beta}{\boldsymbol\varepsilon}^{\beta}

concurrently. They are coupled only through the formation of the Fock matrix
*via* the Coulomb interaction as demonstrated for **F**\ :sup:`α` below:

.. math::

   {F}^{\alpha}_{\mu\nu} = {h}_{\mu\nu}
   + \sum_{\lambda} \sum_{\kappa} \Bigl( ( {P}^{\alpha}_{\lambda\kappa}
   + {P}^{\beta}_{\lambda\kappa} ) \left({\mu\nu}|{\kappa\lambda}\right)
   - {P}^{\alpha}_{\lambda\kappa} \left({\mu\lambda}|{\kappa\nu}\right) \Bigr)

Note that the calculation of **P** differs in the occupation number and for
closed-shell test cases provided your UHF program must give RHF results.
For this reason, you have to break the spatial symmetry of your system to obtain
the UHF solution, if it is available. The easiest way to do this is through the
initial guess. If you use a symmetric guess (like **P** = 0), you will only find
the RHF solution.

The UHF Energy is given by:

.. math::
   E = \frac{1}{2} \sum_{\mu\nu}
   \Bigl(
   {P}^{\alpha}_{\mu\nu}( {h}_{\nu\mu} + {F}^{\alpha}_{\nu\mu} )
   +
   {P}^{\beta}_{\mu\nu}( {h}_{\nu\mu} + {F}^{\beta}_{\nu\mu} )
   \Bigr)

.. admonition:: Exercise 18

   1. Calculate and plot the dissociation/potential curves for
      :sup:`3`\ H\ :sub:`2`, :sup:`1`\ H\ :sub:`2` and :sup:`1`\ Li\ :sub:`2`.
   2. The exchange reaction H\ :sub:`2` + H → H + H\ :sub:`2` has a linear
      symmetric transition state. Find it and its relative energy.
   3. Calculate the first ionization potential of Be.
      The experimental value is 9.3 eV.
   4. Are He\ :sub:`2`\ :sup:`+` or :sup:`3`\ He\ :sub:`2` bonded?

Spin Contamination
^^^^^^^^^^^^^^^^^^

Recall that spin contamination can occur in UHF calculations, *i.e.* deviations
of the expectation value of the square of the total spin angular momentum operator
S\ :sup:`2` from the ideal value:

.. math::

   \langle{\hat S^{2}}\rangle_{\text{UHF}} =
   \langle{\hat S^{2}}\rangle_{\text{exact}} + \Delta \\
   \langle{\hat S^{2}}\rangle_{\text{exact}} =
   \frac{N_{el}^{\alpha} - N_{el}^{\beta}}{2}
   \cdot \left( \frac{N_{el}^{\alpha} - N_{el}^{\beta}}{2} + 1 \right)

Calculate the spin contamination according to

.. math::

   \Delta = N_{el}^{\beta} - \sum_{i}^{N_{el}^{\alpha}}
   \sum_{j}^{N_{el}^{\beta}} \ \Bigl\lvert
   \langle{\chi_{i}^{\alpha}}\vert{\chi_{j}^{\beta}}\rangle
   \Bigr\rvert^{2}

where *i*, *j* indicate *α* and *β*-MOs, respectively.

.. admonition:: Exercise 19

   1. For the system:

      .. code-block:: bash

         h 0.0  0.0  -1.0
         h 0.0  0.0   0.0
         h 0.0  0.2   1.0

      and a Slater exponent of 1.24, you should find a spin contamination of
      0.004682 and a UHF energy of -1.265643.
   2. Plot the spin contamination along both RHF and UHF dissociation curves of
      H\ :sub:`2`.
   3. Why is the spin contamination of RHF always 0?

Møller--Plesset Perturbation Theory
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Using your RHF program, code a subroutine to calculate the RMP2 energy after
your RHF procedure has converged. The RMP2 correction to the HF energy
(assuming real MOs *a, b, r, s*) can be written as

.. math::
   W_{2} = \sum_{a,b=1}^{N_{el}/2} \sum_{\substack{r,s= \\ N_{el}/2+1}}^{M}
   \frac{\left({ar}|{bs}\right)\left[2 \left({ar}|{bs}\right) - \left({as}|{br}\right) \right]}
   {\varepsilon_{a} + \varepsilon_{b} - \varepsilon_{r} -\varepsilon_{s}}.

As for all correlation methods, you will need to have two-electron integrals
over MOs (denoted by Roman letters) instead of over AOs (Greek letters).
The process to obtain them is called AO-MO-Transformation.
There are different algorithms possible, among them are one to scale
with *M*:sup:`8` and one with *M*:sup:`5` that takes approximately twice
as much memory.
For now, code up the straigthforward algorithm to transform the two-electron
integrals. This is the one that scales with *M*:sup:`8`.

+-------------+------------------------+-----------------------------------+
| Input       |  E(RHF) / E\ :sub:`h`  |  E\ :sub:`c`\ (MP2) / E\ :sub:`h` |
+=============+========================+===================================+
| H\ :sub:`2` |  --1.127785613         | --0.012541746                     |
+-------------+------------------------+-----------------------------------+
| He          |  --2.860251227         | --0.012686549                     |
+-------------+------------------------+-----------------------------------+
| Be          | --14.568567143         | --0.014939565                     |
+-------------+------------------------+-----------------------------------+

In contrast to the *M*:sup:`8` variant, where you transform from
:math:`\left({\mu \nu}|{\lambda \kappa}\right)` to
:math:`\left({ar}|{bs}\right)` at once, you need to transform the
two-electron integrals step-by-step:

.. math::

   \left({\mu \nu}|{\lambda \kappa}\right) \rightarrow
   \left({a \nu}|{\lambda \kappa}\right) \rightarrow
   \left({ar}|{\lambda \kappa}\right) \rightarrow
   \left({ar}|{b \kappa}\right) \rightarrow
   \left({ar}|{bs}\right)

.. admonition:: Exercise 20

   1. Copy your RMP2 routine and modify it such that your integral transformation
      scales with *M*:sup:`5`.
   2. For the provided examples H\ :sub:`2` and LiH compare the efficiency of
      both algorithms by counting the number of cycles passed through in the most
      inner loops.
      What happens if you increase the number of Slater functions by a factor of
      two?
   3. Calculate the dissociation curve of H\ :sub:`2` with RMP2.
      Plot both the RHF and RMP2 curves on a relative energy scale (in kcal/mol)
      in a minimal STO basis with Slater exponent of *ζ = 1.2*.
      As in the provided input for H\ :sub:`2`, the Hartree-Fock energy of a
      hydrogen atom is --0.4798356 E\ :sub:`h`.
      What behavior do you observe compared to RHF close to the equilibrium and
      at far distances *R* = 10 Bohr?
   4. Calculate the dissociation curve of He\ :sub:`2` between 4 and 10 Bohr with
      RHF and RMP2 and plot them on a relative energy scale (in kcal/mol).
      You will find that there is a minimum in each curve. From your knowledge
      about HF and MP2, did you expect this behavior?
      What effect could be the cause for the minimum in the RHF curve?
