# Add a concept of constants to Cassowary

## Definitions

- `value:` a primitive float/double
- `variable:` a cassowary variable class/type
- `constant:` a cassowary constant class/type
- `constraint:` a cassowary constraint class/type

An alternative name for `constant` could be `read_only_variable`.

## Introduction

How can this code be improved?
```cplusplus
variable imageWidth, imageHeight;
simplex_solver solver;

solver.add_constraints({
    imageHeight == 200,
    imageWidth == 16/9 * imageHeight
});
```

The `16/9` really bugs me. It should be possible to have that variable derived from data. Imagine the following changes:
```cplusplus
+ constant aspectRatio;
variable imageWidth, imageHeight;
simplex_solver solver;

solver.add_constraints({
    imageHeight == 200,
-    imageWidth == 16/9 * imageHeight
+    imageWidth == aspectRatio * imageHeight
});
```

Furthermore a constant can be updated:
```cplusplus
constant aspectRatio;
variable imageWidth, imageHeight;
simplex_solver solver;

+ solver.set_constant(aspectRatio, 4/3);

solver.add_constraints({
    imageHeight == 200,
    imageWidth == aspectRatio * imageHeight
});

+ solver.set_constant(aspectRatio, 16/9);
```

NB: Operator overloading in examples

## Motivation

While using Cassowary for the last 2 years now we have many times encountered a pattern of problems which all seems to solvable with the same solution.

In our layout models we have several `variables` which act as `read only` variables. We have some system information we like to feed in to cassowary in order to have our layout act on those values. These are values that cassowary shouldn't maintain for several reasons.

Examples of such values maintained by the system
- a scrollviews contentOffset
- a scrollViews contentSize
- the screen dimensions
- intrinsic height and width of a text box (its ideal height given its current width)

Examples of such values from our data models
- aspectRatio of images from our image json api
- number of items in a list

Background: All our constraints are written in a proprietary (for now) json format. In those templates we define the constraints and view hierarchy.

## Proposed solution

In a nutshell: Allow read only variables maintained outside of cassowary to be involved in nonlinear expressions.

### Rules

- Constants can only be set using primitive values and/or other constants
- The values of the constants is resolved in the order they are added to the solver
- Constants aren't affected by constraints

#### "Constants can only be set using primitive values and/or other constants"

Using primitive values
```cplusplus
constant foo, bar;
simplex_solver solver;
solver.set_constant(foo, 16.0 / 9.0);
solver.set_constant(foo, bar);              // throws exception
```

Using other constants
```cplusplus
constant foo, bar;
simplex_solver solver;
solver.set_constant(bar, 16.0 / 9.0);
solver.set_constant(foo, 1.0 / bar);
```
NB: Operator overloading in examples

#### "The values of the constants is resolved in the order they are added to the solver"

```cplusplus
constant widthHeightAspectRatio, originalImageWidth, originalImageHeight;
simplex_solver solver;
solver.set_constant(originalImageWidth, 1024);
solver.set_constant(originalImageHeight, 576);
solver.set_constant(widthHeightAspectRatio, originalImageWidth / originalImageHeight);
solver.set_constant(heightWidthAspectRation, 1.0 / widthHeightAspectRatio);
```

Throw exception if a constant is not set
```cplusplus
constant foo, bar;
simplex_solver solver;
solver.set_constant(foo, bar);              // throws exception: Can not set constant `foo` because `bar` is not set
```
NB: Operator overloading in examples

#### "Constants aren't affected by constraints"

`Constants` behave like primitive `values` in `constraints` thus respecting that Cassowary is still a linear constraint solver.

You can not write:
```cplusplus
constant myConstant;
simplex_solver solver;
solver.add_constraints({ myConstant == 200 });  // throws exception: overconstrained
```

## Detailed design

No detailed design with regards to how we can implement this. Suggestions welcome.


## Alternatives considered

#### Stricter rules

Constants can only be set using primitive values

```cplusplus
constant foo, bar;
simplex_solver solver;
solver.set_constant(foo, 16.0 / 9.0);
solver.set_constant(foo, bar);              // Compilation error
```
