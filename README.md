<p align="center">
  <img src="./docs/src/assets/logo.png" alt="Grassmann.jl"/>
</p>

# Grassmann.jl

*Static dual multivector types with graded-blade indexing, product algebra, and optional origin*

[![Build Status](https://travis-ci.org/chakravala/Grassmann.jl.svg?branch=master)](https://travis-ci.org/chakravala/Grassmann.jl)
[![Build status](https://ci.appveyor.com/api/projects/status/c36u0rgtm2rjcquk?svg=true)](https://ci.appveyor.com/project/chakravala/grassmann-jl)
[![Coverage Status](https://coveralls.io/repos/chakravala/Grassmann.jl/badge.svg?branch=master&service=github)](https://coveralls.io/github/chakravala/Grassmann.jl?branch=master)
[![codecov.io](http://codecov.io/github/chakravala/Grassmann.jl/coverage.svg?branch=master)](http://codecov.io/github/chakravala/Grassmann.jl?branch=master)

This package is a work in progress providing the necessary tools to work with arbitrary dual `MultiVector` elements with optional origin. Due to the parametric type system for the generating `VectorSpace`, the Julia compiler can fully preallocate and often cache values efficiently. Both static and mutable vector types are supported.

It is currently possible to do both high-performance numerical computations with `Grassmann` and it is also currently possible to use symbolic scalar coefficients when the `Reduce` package is also loaded (compatibility instructions at bottom).

### Requirements

This requires a merged version of `ComputedFieldTypes` at https://github.com/vtjnash/ComputedFieldTypes.jl

## Direct-sum yields `VectorSpace` parametric type polymorphism ⨁

Let `N` be the dimension of a `VectorSpace{N}`.
The metric signature of the `Basis{V,1}` elements of a vector space `V` can be specified with the `V"..."` constructor by using `+` and `-` to specify whether the `Basis{V,1}` element of the corresponding index squares to `+1` or `-1`.
For example, `V"+++"` constructs a positive definite 3-dimensional `VectorSpace`.
```Julia
julia> ℝ^3 == V"+++" == VectorSpace(3)
true
```
The direct sum operator `⊕` can be used to join spaces (alternatively `+`), and `'` is an involution which toggles a dual vector space with inverted signature.
```Julia
julia> V = ℝ'⊕ℝ^3
-+++

julia> V'
+---'

julia> W = V⊕V'
-++++---*
```
The direct sum of a `VectorSpace` and its dual `V⊕V'` represents the full mother space `V*`.
```Julia
julia> collect(V) # all vector basis elements
Grassmann.Algebra{-+++,16}(e, e₁, e₂, e₃, e₄, e₁₂, e₁₃, e₁₄, e₂₃, e₂₄, e₃₄, e₁₂₃, e₁₂₄, e₁₃₄, ...)

julia> collect(V') # all covector basis elements
Grassmann.Algebra{+---',16}(f, f¹, f², f³, f⁴, f¹², f¹³, f¹⁴, f²³, f²⁴, f³⁴, f¹²³, f¹²⁴, f¹³⁴, ...)

julia> collect(W) # all mixed basis elements
Grassmann.Algebra{-++++---*,256}(e, e₁, e₂, e₃, e₄, f¹, f², f³, f⁴, e₁₂, e₁₃, e₁₄, e₁f¹, e₁f², ...
```
Not to be confused with the *dual basis*, to generate *dual numbers* it is possible to specify a degenerate `Basis` element with `ϵ` (having property `eϵ*eϵ=0`) at the first index, i.e. `V"ϵ+++"`
Optionally, an additional element can be specified as the *origin* by using `o` instead, i.e. `V"o+++"` or `V"ϵo+++"`.
These two basis elements will be interpreted in the type system such that they propagate under transformations when combining a mixed dimension `Algebra` (provided the signature is compatible).

The `Dimension{N}` of the `Algebra` corresponds to the total number of `Basis{V,1}` vectors, but even though `V"ϵo+++"` of type `VectorSpace{5,1,1}` has 5 `Basis{V,1}` elements, it can be internally recognized in the direct sum algebra as being generated by a `3`-dimensional `VectorSpace{3,0,0}` due to the encoding of the degenerate `Basis` element with `D` and origin with `O` in the `VectorSpace{N,D,O}` type.

## Generating elements and geometric algebra Λ(V)

By virtue of Julia's multiple dispatch on the field type `T`, methods can specialize on the `Dimension{N}` and `Grade{G}` and `VectorSpace{N,D,O}` via the algebra's types, as well as `Basis{V,G}`, `SValue{V,G,T}`, `MValue{V,G,T}`, `SBlade{T,V,G}`, `MBlade{T,V,G}`, `MultiVector{T,V}`, and `MultiGrade{V}` types.

The elements of the `Algebra` can be generated in many ways using the `Basis` elements created by the `@basis` macro,
```Julia
julia> using Grassmann; @basis ℝ'⊕ℝ^3 # equivalent to basis"-+++"
(-+++, e, e₁, e₂, e₃, e₄, e₁₂, e₁₃, e₁₄, e₂₃, e₂₄, e₃₄, e₁₂₃, e₁₂₄, e₁₃₄, e₂₃₄, e₁₂₃₄)
```
As a result of this macro, all of the `Basis{V,G}` elements generated by that `VectorSpace` become available in the local workspace with the specified naming.
The first argument provides signature specifications, the second argument is the variable name for the `VectorSpace`, and the third and fourth argument are the the prefixes of the `Basis` vector names (and covector basis names). By default, `V` is assigned the `VectorSpace` and `e` is the prefix for the `Basis` elements. Thus,
```Julia
julia> V # Minkowski spacetime
-+++

julia> typeof(V) # dispatch by vector space
VectorSpace{4,0,0,0x0001,0}

julia> typeof(e13) # extensive type info
Basis{-+++,2,0x0005}

julia> e13∧e2 # exterior tensor product
-1e₁₂₃

julia> ans^2 # applies geometric product
1e
```
It is entirely possible to assign multiple different bases with different signatures without any problems. In the following command, the `@basis` macro arguments are used to assign the vector space name to `S` instead of `V` and basis elements to `b` instead of `e`, so that their local names do not interfere:
```Julia
julia> @basis "++++" S b;

julia> let k = (b1+b2)-b3
           for j ∈ 1:9
               k = k*(b234+b134)
               println(k)
       end end
0 + 1e₁₄ + 1e₂₄ + 2e₃₄
0 - 2e₁ - 2e₂ + 2e₃
0 - 2e₁₄ - 2e₂₄ - 4e₃₄
0 + 4e₁ + 4e₂ - 4e₃
0 + 4e₁₄ + 4e₂₄ + 8e₃₄
0 - 8e₁ - 8e₂ + 8e₃
0 - 8e₁₄ - 8e₂₄ - 16e₃₄
0 + 16e₁ + 16e₂ - 16e₃
0 + 16e₁₄ + 16e₂₄ + 32e₃₄
```
Alternatively, if you do not wish to assign these variables to your local workspace, the versatile `Grassmann.Algebra{N}` constructors can be used to contain them, which is exported to the user as the method `Λ(V)`,
```Julia
julia> G3 = Λ(3) # equivalent to Λ(V"+++"), Λ(ℝ^3), Λ.V3
Grassmann.Algebra{+++,8}(e, e₁, e₂, e₃, e₁₂, e₁₃, e₂₃, e₁₂₃)

julia> G3.e13 * G3.e12
e₂₃
```
Another way is `Λ.V3`, then it is possible to assign the **quaternion** generators `i,j,k` with
```Julia
julia> i,j,k = Λ.V3.e32, Λ.V3.e13, Λ.V3.e21
(-1e₂₃, e₁₃, -1e₁₂)

julia> @btime i^2, j^2, k^2, i*j*k
  158.925 ns (5 allocations: 112 bytes)
(-1e, -1e, -1e, -1e)
```
The parametric type formalism in `Grassmann` is highly expressive to enable the pre-allocation of geometric algebra computations for specific sparse-subalgebras, including the representation of rotational groups, Lie bivector algebras, and affine projective geometry.

Groups such as SU(n) can be represented with the dual Grassmann’s exterior product algebra, generating a `2^(2n)`-dimensional mother algebra with geometric product from the `n`-dimensional vector space and its dual vector space. The product of the vector basis and covector basis elements form the `n^2`-dimensional bivector subspace of the full `(2n)!/(2(2n−2)!)`-dimensional bivector sub-algebra.
The package `Grassmann` is working towards making the full extent of this number system available in Julia by using static compiled parametric type information to handle sparse sub-algebras, such as the (1,1)-tensor bivector algebra.

## Constructing linear transformations from mixed tensor product ⊗

Note that `Λ.V3` gives the vector basis, and `Λ.C3` gives the covector basis:
```Julia
julia> Λ.V3
[ Info: Precomputing 8×Basis{VectorSpace{3,0,0,0},...}
Grassmann.Algebra{+++,8}(e, e₁, e₂, e₃, e₁₂, e₁₃, e₂₃, e₁₂₃)

julia> Λ.C3
[ Info: Precomputing 8×Basis{VectorSpace{3,0,0,7}',...}
Grassmann.Algebra{---',8}(f, f¹, f², f³, f¹², f¹³, f²³, f¹²³)
```
The following command yields a local 2D vector and covector basis,
```Julia
julia> mixedbasis"2"
(++--*, e, e₁, e₂, f¹, f², e₁₂, e₁f¹, e₁f², e₂f¹, e₂f², f¹², e₁₂f¹, e₁₂f², e₁f¹², e₂f¹², e₁₂f¹²)

julia> f1+2*f2
0e₁ + 0e₂ + 1f¹ + 2f²

julia> ans(e1+e2)
3e
```
The sum `f1+2*f2` is interpreted as a covector element of the dual vector space, which can be evaluated as a linear functional when a vector argument is input.
Using these in the workspace, it is possible to use the Grassmann exterior `∧`-tensor product operation to construct elements `ℒ` of the (1,1)-bivector subspace of linear transformations from the `Grade{2}` algebra.
```Julia
julia> ℒ = ((e1+2*e2)∧(3*f1+4*f2))(2)
0e₁₂ + 3e₁f¹ + 4e₁f² + 6e₂f¹ + 8e₂f² + 0f¹²
```
The element `ℒ` is a linear form which can take `Grade{1}` vectors as input,
```Julia
julia> ℒ(e1+e2)
7e₁ + 14e₂ + 0f¹ + 0f²

julia> L = [1,2] * [3,4]'; L * [1,1]
2-element Array{Int64,1}:
  7
 14
```
which is a computation equivalent to a matrix computation.

This package is still a work in progress, and the API and implementation may change as more features and algebraic operations and product structure are added.

## Symbolic coefficients by declaring an alternative scalar algebra

```Julia
julia> using GaloisFields,Grassmann

julia> const F = GaloisField(7)
𝔽₇

julia> basis"2"
(++, e, e₁, e₂, e₁₂)

julia> @btime F(3)*e1
  21.076 ns (2 allocations: 32 bytes)
3e₁

julia> @btime inv($ans)
  26.636 ns (0 allocations: 0 bytes)
5e₁
```

Due to the abstract generality of the code generation of the `Grassmann` product algebra, it is easily possible to extend the entire set of operations to other kinds of scalar coefficient types.
By default, the coefficients are required to be `<:Number`. However, if this does not suit your needs, alternative scalar product algebras can be specified with
```Julia
generate_product_algebra(SymField,:(Sym.:*),:(Sym.:+),:(Sym.:-),:svec)
```
where `SymField` is the desired scalar field and `Sym` is the scope which contains the scalar field algebra for `SymField`.

Currently, with the use of `Requires` it is feasible to automatically enable symbolic scalar computation with [Reduce.jl](https://github.com/chakravala/Reduce.jl), e.g.
```Julia
julia> using Reduce, Grassmann
Reduce (Free CSL version, revision 4590), 11-May-18 ...
```
It is **essential** that the `Reduce` package was precompiled without the extra precompilation enabled if using a Linux operating system (`ENV["REDPRE"]=0`), since the extra precompilation causes a segfault when used with `StaticArrays`.
```Julia
julia> basis"2"
(++, e, e₁, e₂, e₁₂)

julia> (:a*e1 + :b*e2) ∧ (:c*e1 + :d*e2)
0.0 + (a * d - b * c)e₁₂

julia> (:a*e1 + :b*e2) * (:c*e1 + :d*e2)
a * c + b * d + (a * d - b * c)e₁₂
```
If these compatiblity steps are followed, then `Grassmann` will automatically declare the product algebra to use the `Reduce.Algebra` symbolic field operation scope.

It should be straight-forward to easily substitute any other extended algebraic operations and for extended fields and pull-requests are welcome.

## References
* C. Doran, D. Hestenes, F. Sommen, and N. Van Acker, [Lie groups as spin groups](http://geocalc.clas.asu.edu/pdf/LGasSG.pdf), J. Math Phys. (1993)
* David Hestenes, [Universal Geometric Algebra](http://lomont.org/Math/GeometricAlgebra/Universal%20Geometric%20Algebra%20-%20Hestenes%20-%201988.pdf), Pure and Applied (1988)
* Peter Woit, [Clifford algebras and spin groups](http://www.math.columbia.edu/~woit/LieGroups-2012/cliffalgsandspingroups.pdf), Lecture Notes (2012)
