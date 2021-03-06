# Objects

A "geometric object", or "object" for short, is a first class value that represents an OpenSCAD script.
* An object contains named fields, which correspond to the top level definitions in a script.
  These fields are referenced as `object.name`.
* An object contains geometry, a sequence of shapes and objects,
  which correspond to the top level geometry statements in a script.
  Geometry is referenced using [sequence operations](Sequences.md).
* Objects are constructed by `script(filename)`, which reads an object from a script file,
  and by `{script}`, which is an object literal.
* Objects may be transformed.
  `object(name1=val1,...)` customizes an object, re-evaluating its script with specified
  definitions overridden by new values, returning a new object.

Objects have multiple roles in OpenSCAD2.
* Objects are the replacement for groups in the CSG tree.
* A geometric model is represented by an object, which encapsulates both its geometry and its parameters.
* Library scripts are referenced as objects. The `include` and `use` operators now take objects as arguments.
  Library scripts with tweakable top-level parameters, like
  [gridbeam.scad](https://github.com/openscad/MCAD/blob/master/gridbeam.scad),
  are parameterized objects.
* A set of model parameters, by itself, can be represented as an object.
* The ability to access fields and customize a parameterized model or library
  makes new programming idioms possible.

## Scripts denote Objects

If OpenSCAD2 has a [declarative semantics](Declarative_Semantics.md),
then an OpenSCAD script must have a meaning&mdash;what is it?
The answer: a script denotes an object.

A script consists of a set of top level definitions, plus some top level geometry statements.
An object is what a script denotes, so an object is a set of named fields, plus a sequence of shapes.

For example,

```
// lollipop.scad
radius   = 10; // candy
diameter = 3;  // stick
height   = 50; // stick

translate([0,0,height]) sphere(r=radius);
cylinder(d=diameter,h=height);
```

The lollipop script denotes an object with 3 named fields
(radius, diameter, height) and two shapes, a sphere and a cylinder.

## The `script` function
To reference a script file, use the `script` function.
Given a filename argument,
it reads a script file, and returns the resulting object.

If you just want to drop a single lollipop into your model,
you could do this:
```
// add a lollipop to our geometry list
script("lollipop.scad");
```

If you want to reference the same script more than once,
you can give it a name.
For example,
```
lollipop = script("lollipop.scad");
```

Note: this is consistent with the way that all other external files are
referenced: you pass a filename argument to a function that reads the file
and returns a value.
Other examples are `import()`, `dxf_cross()` and `surface()`.

## The Object API

Objects are [First Class Values](First_Class_Values.md).

An object contains a set of named fields that can be referenced using `.` notation.
* `lollipop.height` is the stick height

An object is a geometric object containing a sequence of shapes.
It can be used in any context requiring a shape or sequence of shapes.
For example,
* `scale(10) lollipop`
* `intersection() lollipop // intersection of the stick and candy`

Objects support all of the
[generic sequence operations](Sequences.md):
* `len(lollipop) == 2`
* `lollipop[1] == cylinder(3,50)`

## Object Literals

The [First Class Values](First_Class_Values.md) principle requires object literals.

An object literal is a script surrounded by brace brackets.
```
// define the lollipop object
lollipop = {
  radius   = 10; // candy
  diameter = 3;  // stick
  height   = 50; // stick

  translate([0,0,height]) sphere(r=radius);
  cylinder(d=diameter,h=height);
};

// now add some lollipops to our geometry list
translate([-50,0,0]) lollipop;
translate([50,0,0]) lollipop;
```

Since objects are the replacement for groups,
you can group shapes using `{shape1; shape2;}`.
And this makes object literals
a backwards-compatible reinterpretation of the `{...}` syntax.

## Customization

`object(name1=value1, name2=value2, ...)` customizes an object
by overriding specified definitions with new values,
re-evaluating the script and returning a new object.
  
```
// add a lollipop with bigger candy to the geometry list
lollipop(radius=15);
```

A script can be customized on the command line with the `-D` flag.
```
openscad -Dname1=value1 -Dname2=value2 ... myscript.scad
```

The new Customizer GUI under development does the same thing, only interactively.

Customization is a relatively fast operation, since most of the
structure of the new object is shared with the base object.

Customization is also a limited operation that is only good for
overriding parameters in an object. Two things you can't do
with customization, that you can do with `overlay`:
* You can't add new fields to the object.
* You can't create a dependency between two fields
  in the new object, such that customizing one field updates the other.

See [Object Inheritance](Inheritance.md)
for more powerful ways to transform an object.

<!--
## Parameter Sets
A set of model parameters can be represented as an object.
This is used by a couple of idioms.

Some authors create scad files containing sets of model parameters.
For example, Nophead's Mendel90 project has `config.scad` which looks, in part, like this:
```
...a long list of default configuration settings...
include <machine.scad> // override defaults for a particular machine
```
Here's my translation of this code into OpenSCAD2:
```
defaults = {
...a long list of default configuration settings...
};
include defaults with script("machine.scad");
```

Some authors represent a parameter set as an array of name/value pairs,
and use `search` to look up a parameter. In OpenSCAD2, a parameter array
can be replaced by an object literal, and you can use `paramset.name`
to look up a parameter. This way, it's also easy to organize parameters into a hierarchy.

### `apply`
`apply(base_object, parameter_set)` customizes the base object
with the specified parameters. It has the same semantics as customization.
Unlike the `overlay` operator, this will report an error if `parameter_set`
contains fields not within `base_object`.
-->

## The CSG Tree

Objects are the replacement for groups in the CSG tree.

The root of the CSG tree is the object denoted by the main script.
The leaf nodes of the CSG tree are shapes, and the non-leaf nodes are objects.

A list of shapes and objects is operationally equivalent to
an object containing the same geometry sequence.
However, if such a list is used in a context where geometry is required,
then the list is implicitly converted to an object.
For example, if you write
```
union()([ cube(12,true), sphere(8) ]);
```
then the list is converted to an object, so the above code is equivalent to
```
union() { cube(12,true); sphere(8); }
```
The conversion from list to object reports an error if non-geometric values occur in the list.
This conversion ensures that only objects and shapes appear in the final CSG tree.

The modifier characters `%`, `#` and `!` take a geometry value as an argument,
and set a modifier flag in the value. If the argument is a list, it's first converted
to an object, since only shape and object values contain storage for modifier flags.

The undocumented `group` module has always been an identity function that returns its children:
```
group(){ cube(12,true); sphere(8); }
```
In OpenSCAD2, the above call to `group()` returns an object, and is equivalent to just
```
       { cube(12,true); sphere(8); }
```
