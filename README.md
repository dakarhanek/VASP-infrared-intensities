# VASP-infrared-intensities
A simple BASH script for extraction of infared intensities from DFPT calculation output by VASP code.

This webspace is a continuation of my former project I had placed earlier at:  
<del>&nbsp; http://homepage.univie.ac.at/david.karhanek/downloads.html &nbsp;</del>   
My personal site at the University of Vienna is down/discontinued ->  
 -> **please refer to [this github.com location](https://github.com/dakarhanek/VASP-infrared-intensities/) or [DOI 10.5281/zenodo.3930989](https://dx.doi.org/10.5281/zenodo.3930989) hereafter**.

The idea of publishing the script originated in 2011 (and that's still the status quo) in a CCL mailing list:   
http://www.ccl.net/chemistry/resources/messages/2011/04/06.004-dir/     
Kind thanks to [Ralf Tonner](https://www.researchgate.net/profile/Ralf-Tonner-Zech) and [Tomáš Bučko](https://www.researchgate.net/profile/Tomas_Bucko) for their contribution.

---

intensities.sh
==========
An article about VASP **vibrational intensities** using DFPT (Density-Functional Perturbation Theory), *a.k.a.* LRT (Linear Response Theory).

DESCRIPTION
---------------
Although vibrational analysis is a kind of routine task, sometimes it needs a bit of playing around in order to be done really properly, or to get some extra information (yes, the intensities, *vide infra*).

Mostly we just want to "check" if our optimization finished to a reasonable minimum or that the transition state we found is a pretty saddle point we wanted to have... well, in these cases, we can be satisfied with changing the INCAR settings to be: <tt>IBRION=5</tt>, <tt>NFREE=2</tt>, <tt>POTIM="small_number"</tt> (whereas <tt>"small_number" &lt; 0.05</tt>). Sometimes it's okay to proceed the "infrared" analysis even at lower *k*-point grid in spite of speed.

In older VASP versions (&lt; 5), the intensities of vibrational modes could only be calculated (with obstructions) based on the change of dipole moment for each vibration direction separately. This is the same implementation as it is being routinely used while implemented in software suites like GAUSSIAN - where the corresponding source code comes from 1970's.

As of VASP 5.* version, the DFPT linear response calculations are available. Among others, the user can obtain the matrix of *Born effective charges* ([BEC](http://cms.mpi.univie.ac.at/vasp/Berry_phase/node4.html)), which refers to change of atoms' polarizabilities w.r.t. an external electric field. The BEC tensor is a key to calculate the vibrational intensities using the most modern method available (as far as I know!), using the formula by Gianozzi & Baroni (applications [01](http://dx.doi.org/10.1063/1.466753) and [02](http://dx.doi.org/10.1103/PhysRevB.57.223), theory [03](http://dx.doi.org/10.1103/RevModPhys.73.515), further reading: [David's PhD thesis](http://othes.univie.ac.at/10117/), chapter 2.10.3).

For calculations with VASP DPFT formalism, include the following in the INCAR file:

<table> <tbody> 
 <tr> <td> <tt>IBRION = 7</tt> </td> <td> switches *on* the DFPT vibrational analysis (with no symmetry constraints) (<a href="http://cms.mpi.univie.ac.at/vasp/vasp/Optical_properties_density_functional_perturbation_theory_PT.html">man</a>) </td> </tr> 
 <tr> <td> <tt>LEPSILON = .TRUE.</tt> </td> <td> enables to calculate and prints BEC tensor (<a
href="http://cms.mpi.univie.ac.at/vasp/vasp/LEPSILON_static_dielectric_matrix_ion_clamped_piezoelectric_tensor_Born_effective_charges.html">man</a>) </td> </tr>
 <tr> <td> <tt>NSW = 1</tt> </td> <td> reduces # of ionic steps to 1; unlike for <tt>IBRION=5</tt>, here the setting is very important </td> </tr> </tbody>
</table>
      
<dl> <dd><b>Note</b> In the INCAR file, please take care that you also obtain the electronic density properly converged with <tt>NELMIN=10</tt> in combination with <tt>NELMIN=120</tt> and always SWITCH OFF THE SYMMETRY with <tt>ISYM=0</tt> tag. </dd> </dl>
      
In case that you want to extract intensities from OUTCAR, submit the below script without any parameters (it reads the "OUTCAR" file in the current directory). The output is written into "<tt>intensities</tt>" subdirectory, the most imporant output is to be found in the file "<tt>intensities/results/results.txt</tt>", whereas "<tt>intensities/results/exact.res.txt</tt> contains intensities before normalization".
      
The script <tt>intensities.sh</tt> itself:
      
CODE
---------------
<pre>#!/bin/bash  
# A utility for calculating the vibrational intensities from VASP output (OUTCAR)  
# (C) David Karhanek, 2011-08-05, ICIQ Tarragona, Spain (www.iciq.es)  
  
# extract Born effective charges tensors  
printf "..reading OUTCAR"  
BORN_NROWS=`grep NIONS OUTCAR | awk '{print $12*4+1}'`  
if [ `grep 'BORN' OUTCAR | wc -l` = 0 ] ; then printf " .. FAILED! Born effective charges missing! Bye! \n\n" ; exit 1 ; fi  
grep "in e, cummulative" -A $BORN_NROWS OUTCAR > born.txt  
   
# extract Eigenvectors and eigenvalues  
if [ `grep 'SQRT(mass)' OUTCAR | wc -l` != 1 ] ; then printf " .. FAILED! Restart VASP with NWRITE=3! Bye! \n\n" ; exit 1 ; fi  
EIG_NVIBS=`grep -A 2000 'SQRT(mass)' OUTCAR | grep 'cm-1' | wc -l`  
EIG_NIONS=`grep NIONS OUTCAR | awk '{print $12}'`  
EIG_NROWS=`echo "($EIG_NIONS+3)*$EIG_NVIBS+3" | bc`  
grep -A $(($EIG_NROWS+2)) 'SQRT(mass)' OUTCAR | tail -n $(($EIG_NROWS+1)) | sed 's/f\/i/fi /g' > eigenvectors.txt  
printf " ..done\n"  
   
# set up a new directory, split files - prepare for parsing  
printf "..splitting files"  
mkdir intensities ; mv born.txt eigenvectors.txt intensities/  
cd intensities/  
let NBORN_NROWS=BORN_NROWS-1  
let NEIG_NROWS=EIG_NROWS-3  
let NBORN_STEP=4  
let NEIG_STEP=EIG_NIONS+3  
tail -n $NBORN_NROWS born.txt > temp.born.txt  
tail -n $NEIG_NROWS eigenvectors.txt > temp.eige.txt  
mkdir inputs ; mv born.txt eigenvectors.txt inputs/  
split -a 3 -d -l $NEIG_STEP temp.eige.txt temp.ei.  
split -a 3 -d -l $NBORN_STEP temp.born.txt temp.bo.  
mkdir temps01 ; mv temp.born.txt temp.eige.txt temps01/  
for nu in `seq 1 $EIG_NVIBS` ; do  
 let nud=nu-1 ; ei=`printf "%03u" $nu` ; eid=`printf "%03u" $nud` ; mv temp.ei.$eid eigens.vib.$ei   
done  
for s in `seq 1 $EIG_NIONS` ; do  
 let sd=s-1 ; bo=`printf "%03u" $s` ; bod=`printf "%03u" $sd` ; mv temp.bo.$bod borncs.$bo   
done  
printf " ..done\n"  
   
# parse deviation vectors (eig)  
printf "..parsing eigenvectors"  
let sad=$EIG_NIONS+1  
for nu in `seq 1 $EIG_NVIBS` ; do  
 nuu=`printf "%03u" $nu`  
 tail -n $sad eigens.vib.$nuu | head -n $EIG_NIONS | awk '{print $4,$5,$6}' > e.vib.$nuu.allions  
 split -a 3 -d -l 1 e.vib.$nuu.allions temp.e.vib.$nuu.ion.  
 for s in `seq 1 $EIG_NIONS` ; do  
  let sd=s-1; bo=`printf "%03u" $s`; bod=`printf "%03u" $sd`; mv temp.e.vib.$nuu.ion.$bod e.vib.$nuu.ion.$bo  
 done  
done  
printf " ..done\n"  
  
# parse born effective charge matrices (born)  
printf "..parsing eff.charges"  
for s in `seq 1 $EIG_NIONS` ; do  
 ss=`printf "%03u" $s`  
 awk '{print $2,$3,$4}' borncs.$ss | tail -3 > bornch.$ss  
done  
mkdir temps02 ; mv eigens.* borncs.* temps02/  
printf " ..done\n"  
   
# parse matrices, multiply them and collect squares (giving intensities)  
printf "..multiplying matrices, summing "  
for nu in `seq 1 $EIG_NVIBS` ; do  
 nuu=`printf "%03u" $nu`  
 int=0.0  
 for alpha in 1 2 3 ;  do            # summing over alpha coordinates  
  sumpol=0.0  
  for s in `seq 1 $EIG_NIONS` ; do   # summing over atoms  
   ss=`printf "%03u" $s`  
   awk -v a="$alpha" '(NR==a){print}' bornch.$ss > z.ion.$ss.alpha.$alpha  
   # summing over beta coordinates and multiplying Z(s,alpha)*e(s) done by the following awk script  
   paste z.ion.$ss.alpha.$alpha  e.vib.$nuu.ion.$ss | \  
   awk '{pol=$1*$4+$2*$5+$3*$6; print $0,"  ",pol}' > matr-vib-${nuu}-alpha-${alpha}-ion-${ss}  
  done  
  sumpol=`cat matr-vib-${nuu}-alpha-${alpha}-ion-* | awk '{sum+=$7} END {print sum}'`  
  int=`echo "$int+($sumpol)^2" | sed 's/[eE]/*10^/g' |  bc -l`  
 done  
 freq=`awk '(NR==1){print $8}' temps02/eigens.vib.$nuu`  
 echo "$nuu $freq $int">> exact.res.txt  
 printf "."  
done  
printf " ..done\n"  
   
# format results, normalize intensities  
printf "..normalizing intensities"  
max=`awk '(NR==1){max=$3} $3>=max {max=$3} END {print max}' exact.res.txt`  
awk -v max="$max" '{printf "%03u %6.1f %5.3f\n",$1,$2,$3/max}' exact.res.txt > results.txt  
printf " ..done\n"  
   
# clean up, display results  
printf "..finalizing:\n"  
mkdir temps03; mv bornch.* e.vib.*.allions temps03/  
mkdir temps04; mv z.ion* e.vib.*.ion.* temps04/  
mkdir temps05; mv matr-* temps05/  
mkdir results; mv *res*txt results/  
let NMATRIX=$EIG_NVIBS**2  
printf "%5u atoms found\n%5u vibrations found\n%5u matrices evaluated" \  
       $EIG_NIONS $EIG_NVIBS $NMATRIX > results/statistics.txt  
  # fast switch to clean up all temporary files  
  rm -r temps*  
cat results/results.txt  
</pre>
---

CONTACT
---------------
**Dr.rer.nat. Ing. David Karhánek**  
&#9755; [david.karhanek@gmail.com](mailto:david.karhanek@gmail.com)  
&#9784; [http://researchgate.net/profile/David_Karhanek](http://www.researchgate.net/profile/David_Karhanek)

Alternatives
---------------
If you decide to fool around with **infrared intensities** in VASP a step further, you might also be interested in:
* "VASP tools" by Sébastien Nénon: https://github.com/sebnenon/VASP_tools
* "IR" by Janine George: https://github.com/JaGeo/IR
* "ASE" environment: https://wiki.fysik.dtu.dk/ase/ase/vibrations/infrared.html
* "Phonopy-Spectroscopy" project: https://phonopy.github.io/phonopy/external-tools.html#id2

For **Raman intensities**, see:
* "VASP-RAMAN" by Alexandr Fonari & Shannon Stauffer: https://github.com/raman-sc/VASP/
