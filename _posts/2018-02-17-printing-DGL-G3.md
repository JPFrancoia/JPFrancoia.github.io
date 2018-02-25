---
layout: posts
title:  "Printing DGLs with an Ultimaker 3"
image: printed_G3_G4.jpg

excerpt: "In this post I'll demonstrate the printing capabilities of the latest 3D
printer made by Ultimaker: the Ultimaker 3. I'll test them on the model of
a DGL G3 I could not print before with a \"traditional\" 3D printer."

---

In the [Cronin group](http://www.chem.gla.ac.uk/cronin/), we currently have
(among others) two Ultimaker 2 and two Ultimaker 2+. They are good quality
Fuse Deposition Modeling (FDM) printers, but because they have only one
extruder, they can only print one material at a time.  

When printing complicated models with overhangs (parts of the model that
rest on nothing), it is common to also print some supports. These supports
are here to provide a surface on which overhangs can be printed on. They
are supposed to be easy to remove once the printing process is finished,
but let's not lie: a) it is not always easy to remove them b) the final
aspect does not look great. It requires manual post-processing of the printed
model to make it look nice, or to make it functional if it is a part of a
mechanical assembly. That is a point where FDM printers fall short compared
to "industrial" 3D printers that can print soluble supports.

Recently, Ultimaker released its last FDM printer: the [Ultimaker
3](https://ultimaker.com/en/products/ultimaker-3). While the difference
between an Ultimaker 2 and an Ultimaker 2+ was slim, the Ultimaker 3 has tons
of new features and is really different to any FDM printer made before. I will
not describe all these new features here, but I will say it has two extruders
(which means it can print two materials at a time). And Ultimaker sells [a
soluble material](https://ultimaker.com/en/products/materials/pva), a polymer
called [PolyVinyl Alcohol](https://en.wikipedia.org/wiki/Polyvinyl_alcohol)
(PVA).

I wanted to test the dual extruder, and I decided to print a model I knew was
just impossible to print: a DGL G3 (for more information about DGLs, see [this
publication](http://onlinelibrary.wiley.com/wol1/doi/10.1002/chem.201704147/abstract)).
Here is what it looks like in a CPK representation:

<p align="center">
  <img width="500" src="{{ site.baseurl }}/images/cpk.jpg">
</p>

You can download the PDB [here]({{ site.baseurl }}/attached/G3.pdb). I used
[VMD](http://www.ks.uiuc.edu/Research/vmd/) to render a "Surf" representation
of this molecule to a STL file (format used by 3D printing softwares to
prepare the printing process).  


Then I printed this model with [this PolyLactic Acid
(PLA)](http://www.eumakers.com/en/filamento-pla-azzurro-perla.html?___SID=U)
in the first extruder, and Ultimaker PVA in the second one. The second
extruder was used to print the supports. I printed this model with layers
of 100 µm, and here is what it looks like once printed:

<p align="center">
  <img width="700" src="{{ site.baseurl }}/images/pva_support.jpg">
</p>

For the post-processing part, I simply sonicated the model for a few hours at
40°C in water. No manual intervention was required. When I took the model
out of the sonicator, it looked absolutely perfect. I could not notice any
support residue:

<p align="center">
  <img width="700" src="{{ site.baseurl }}/images/printed_G3.jpg">
</p>

I also repeated the same procedure for a fourth generation DGL (DGL G4,
PDB [here]({{ site.baseurl }}/attached/G4.pdb), on the right of the picture):

<p align="center">
  <img width="700" src="{{ site.baseurl }}/images/printed_G3_G4.jpg">
</p>

I think the combination dual extruder/soluble material allows FDM printers
to fill the gap between them and more "industrial" printers, for a much
lower price.
