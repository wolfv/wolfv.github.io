---
layout: post
title: "Designing Fonts and Other Things with a Constraint Solver"
author: "Wolf Vollprecht"
summary: CAD programs have a constraint solver for sketches since forever, but they are generally missing from graphics software.
image: /assets/images/2020/font_design_title_image.png
categories: posts
---

<p class="subtitle">
	For exact geometrical and parameterized sketches, engineers have used constraint solvers 
	since around 30 years. However, this has never carried 
	over into the "artistic" world, even though it has the potential to make 
	certain repetetive tasks much simpler. Especially for font design and layout
	tasks a constraint system seems like a good idea.
</p>

<figure>
	<img src="/assets/images/2020/solvespace_1.png" alt="SolveSpace 2D constraints"/>
	<figcaption>SolveSpace 2D constraints with dimension constraint, 
	horizontal, vertical, and angular constraints.</figcaption>
</figure>

When sketching a _parametric_, mechanical work piece, it is essential to be able to define _constraints_ such as:

- Point on: restrict a certain point to always lie on a line, or another point
- Tangential to: make some other line always tangential to this arc or circle
- Angle constraint: so that the angle between two lines is always e.g. 90°
- Equal length: one line segment always has the same length as another

and many more. This interactively builds up a constrained system. Ideally, the
entire system is "fully constrained". Once that state is reached, no degrees of freedom remain and the construction cannot be moved without also modifying a constraint. However, especially for interactive design it might be desirable to keep moving parts of the sketch (e.g. interactively change the width of a font). 

### Freely available CAD programs for Linux

There are some open source CAD programs availble today, which implement a 2D sketch
solver:

- [FreeCAD](https://freecadweb.org), probably the most popular open source CAD toolkit has a constraint solver built in ([PlaneGCS](https://github.com/FreeCAD/FreeCAD/tree/b2ffebf1c0ecee57bf5329f4d2546fd98f3f6399/src/Mod/Sketcher/App/planegcs))
- [SolveSpace](http://solvespace.com/) is a "bare-bones" looking, slick 3D CAD toolkit that carries "solve" in it's name
  and accordingly comes with a constraint solver
- [NoteCAD](http://notecad.xyz) is a online CAD software (notecad.xyz) that is
  built from scratch with the game framework Unity in C#

For this initial experiment with font design I used SolveSpace, but any of the other two projects would have worked just as well.

<video width="100%" controls autoplay="true">
  <source src="/assets/images/2020/constrained_font.webm" type="video/webm">
  <source src="/assets/images/2020/constrained_font.mp4" type="video/mp4">
</video>

As can be seen in the video, all properties of this simple font are connected – the width of each of the stems is always equal and the radius of the curves is equal across the different letters. Obviously this is a very simple example, but more complex scenarios are definitly envisioned.

Inkscape already has an open issue for integrating with a geometric constraint solver here: [GitLab Issue](https://gitlab.com/inkscape/inbox/issues/1465). I seriously think it could be a "killer"-feature for any font editor or Inkscape!

### A standalone constraint solver

In order to make this functionality available to other software, such as Inkscape, BirdFont or FontForge, I have started to port the constraint solver found in NoteCAD to C++. C++ is a good choice because it can be compiled for any target system and offers high performance, and pybind11 makes it easy to create Python bindings in the future. Other options could have been to write it directly in Python and accelerate using JIT compilers such as JAX or numba, or to write it in Rust (I might attempt to por the solver to Rust at some point in the future, just for fun).

For now, the project is called `Adjacent`, and the source code can be found on [GitHub](https://github.com/evil-spirit/Adjacent). I am currently working on adding a simple entity system and constraints (right now it's the bare-bones solver). In the near future, I would like to support a syntax similar to this pseudo-Python:

_Note: this is not yet implemented!_

```py
p1, p2, p3 = Point(0, 5), Point(2, 1), Point(3, 3) 
line = Line(p1, p2)
constraints = ConstraintSystem()
constraints.add(POINT_ON, p3, line)
constraints.solve()
print(p1, p2, p3)
# prints newly computed points
```

Another obvious use case would be to integrate the geometric constraint solver into [scikit-geometry](https://github.com/scikit-geometry/scikit-geometry), to allow for constraining geometric constructions easily.

Lot's of exciting things ahead!

---

If you want to follow news around this subject, you can follow me on **[⇒ Twitter](https://twitter.com/@wuoulf)**