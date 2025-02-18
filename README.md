NamedArrays
===========

Julia type that implements a drop-in wrapper for `AbstractArray` type, providing named indices and dimensions.

[![CI](https://github.com/davidavdav/NamedArrays.jl/actions/workflows/CI.yml/badge.svg)](https://github.com/davidavdav/NamedArrays.jl/actions/workflows/CI.yml)
[![Coverage Status](https://coveralls.io/repos/github/davidavdav/NamedArrays.jl/badge.svg?branch=master)](https://coveralls.io/github/davidavdav/NamedArrays.jl?branch=main)

Idea
----

We would want to have the possibility to give each row/column/... in
an Array names, as well as the array dimensions themselves.  This
could be used for pretty-printing, indexing, and perhaps even some
sort of dimension-checking in certain matrix computations.

In all other respects, a `NamedArray` should behave the same as the underlying `AbstractArray`.

A `NamedArray` should adhere to the [interface definition](https://docs.julialang.org/en/latest/manual/interfaces/#man-interface-array-1) of an `AbstractArray` itself, if there are cases where this is not true, these should be considered bugs in the implementation of `NamedArrays`.

Synopsis
--------

```julia
julia> using NamedArrays

julia> n = NamedArray(rand(2,4))
2×4 Named Matrix{Float64}
A ╲ B │         1          2          3          4
──────┼───────────────────────────────────────────
1     │  0.640719   0.996256   0.534355   0.610259
2     │   0.67784   0.281928  0.0112326   0.672123

julia> setnames!(n, ["one", "two"], 1)         # give the names "one" and "two" to the rows (dimension 1)
(OrderedCollections.OrderedDict{Any, Int64}("one" => 1, "two" => 2), OrderedCollections.OrderedDict{Any, Int64}("1" => 1, "2" => 2, "3" => 3, "4" => 4))

julia> n["one", 2:3]
2-element Named Vector{Float64}
B  │
───┼─────────
2  │ 0.996256
3  │ 0.534355

julia> n["two", :] = 11:14
11:14

julia> n[Not("two"), :] = 4:7                  # all rows but the one called "two"
4:7

julia> n
2×4 Named Matrix{Float64}
A ╲ B │    1     2     3     4
──────┼───────────────────────
one   │  4.0   5.0   6.0   7.0
two   │ 11.0  12.0  13.0  14.0

julia> sum(n, dims=1)
1×4 Named Matrix{Float64}
 A ╲ B │    1     2     3     4
───────┼───────────────────────
sum(A) │ 15.0  17.0  19.0  21.0
```

Construction
-------

### Default names for indices and dimensions
```julia
julia> n = NamedArray([1 2; 3 4]) ## NamedArray(a::Array)
2×2 Named Matrix{Int64}
A ╲ B │ 1  2
──────┼─────
1     │ 1  2
2     │ 3  4

n = NamedArray{Int}(2, 2) ## NamedArray{T}(dims...)
2×2 Named Matrix{Int64}
A ╲ B │                 1                  2
──────┼─────────────────────────────────────
1     │             33454  72058693549555969
2     │ 72339073326448640         4318994440
```

These constructors add default names to the array of type String, `"1"`,
`"2"`, ... for each dimension, and names the dimensions `:A`, `:B`,
... (which will be all right for 26 dimensions to start with; 26
dimensions should be enough for anyone:-).  The former initializes
the NamedArray with the Array `a`, the latter makes an uninitialized
NamedArray of element type `T` with the specified dimensions `dims...`.

### Lower level constructors

The key-lookup for names is implemented by using `DataStructures.OrderedDict`s for each dimension.  At a lower level, you can construct `NamedArrays` this way:
```julia
julia> using DataStructures

julia> n = NamedArray([1 3; 2 4], ( OrderedDict("A"=>1, "B"=>2), OrderedDict("C"=>1, "D"=>2) ),
                      ("Rows", "Cols"))
2×2 Named Matrix{Int64}
Rows ╲ Cols │ C  D
────────────┼─────
A           │ 1  3
B           │ 2  4
```
This is the basic constructor for a NamedArray.  The second argument `names` must be a tuple of `OrderedDict`s whose range (the values) are exacly covering the range `1:size(a,dim)` for each dimension.   The keys in the various dictionaries may be of mixed types, but after construction, the type of the names cannot be altered.  The third argument `dimnames` is a tuple of the names of the dimensions themselves, and these names may be of any type.

### Vectors of names

```julia
# NamedArray{T,N}(a::AbstractArray{T,N}, names::NTuple{N,Vector}, dimnames::NTuple{N})
julia> n = NamedArray([1 3; 2 4], ( ["a", "b"], ["c", "d"] ), ("Rows", "Cols"))
2×2 Named Matrix{Int64}
Rows ╲ Cols │ c  d
────────────┼─────
a           │ 1  3
b           │ 2  4

# NamedArray{T,N}(a::AbstractArray{T,N}, names::NTuple{N,Vector})
julia> n = NamedArray([1 3; 2 4], ( ["a", "b"], ["c", "d"] ))
2×2 Named Matrix{Int64}
A ╲ B │ c  d
──────┼─────
a     │ 1  3
b     │ 2  4

julia> n = NamedArray([1, 2], ( ["a", "b"], ))  # note the comma after ["a", "b"] to ensure evaluation as tuple
2-element Named Vector{Int64}
A  │
───┼──
a  │ 1
b  │ 2

# Names can also be set with keyword arguments
julia> n = NamedArray([1 3; 2 4]; names=( ["a", "b"], ["c", "d"] ), dimnames=("Rows", "Cols"))
2×2 Named Matrix{Int64}
Rows ╲ Cols │ c  d
────────────┼─────
a           │ 1  3
b           │ 2  4
```
This is a more friendly version of the basic constructor, where the range of the dictionaries is automatically assigned the values `1:size(a, dim)` for the `names` in order. If `dimnames` is not specified, the default values will be used (`:A`, `:B`, etc.).

In principle, there is no limit imposed to the type of the `names` used, but we discourage the use of `Real`, `AbstractArray` and `Range`, because they have a special interpretation in `getindex()` and `setindex`.

Indexing
------

### `Integer` indices

Single and multiple integer indices work as for the underlying array:

```julia
julia> n[1, 1]
1

julia> n[1]
1
```

Because the constructed `NamedArray` itself is an `AbstractArray`, integer indices always have precedence:

```julia
julia> a = rand(2, 4)
2×4 Matrix{Float64}:
 0.272237  0.904488  0.847206  0.20988
 0.533134  0.284041  0.370965  0.421939

julia> dodgy = NamedArray(a, ([2, 1], [10, 20, 30, 40]))
2×4 Named Matrix{Float64}
A ╲ B │       10        20        30        40
──────┼───────────────────────────────────────
2     │ 0.272237  0.904488  0.847206   0.20988
1     │ 0.533134  0.284041  0.370965  0.421939

julia> dodgy[1, 1] == a[1, 1]
true

julia> dodgy[1, 10] ## BoundsError
ERROR: BoundsError: attempt to access 2×4 Matrix{Float64} at index [1, 10]
```
In some cases, e.g., with contingency tables, it would be very handy to be able to use named Integer indices.  In this case, in order to circumvent the normal `AbstractArray` interpretation of the index, you can wrap the indexing argument in the type `Name()`
```julia
julia> dodgy[Name(1), Name(30)] == a[2, 3] ## true
true
```

### Named indices

```julia
julia> n = NamedArray([1 2 3; 4 5 6], (["one", "two"], [:a, :b, :c]))
2×3 Named Matrix{Int64}
A ╲ B │ a  b  c
──────┼────────
one   │ 1  2  3
two   │ 4  5  6


julia> n["one", :a] == 1
true

julia> n[:, :b] == [2, 5]
true

julia> n["two", [1, 3]] == [4, 6]
true

julia> n["one", [:a, :b]] == [1, 2]
true

```

This is the main use of `NamedArrays`.  Names (keys) and arrays of names can be specified as an index, and these can be mixed with other forms of indexing.

### Slices

The example above just shows how the indexing works for the values, but there is a slight subtlety in how the return type of slices is determined

When a single element is selected by an index expression, a scalar value is returned.  When an array slice is selected, an attempt is made to return a NamedArray with the correct names for the dimensions.


```julia
julia> n[:, :b] ## this expression drops the singleton dimensions, and hence the names
2-element Named Vector{Int64}
A   │
────┼──
one │ 2
two │ 5

julia> n[["one"], [:a]] ## this expression keeps the names
1×1 Named Matrix{Int64}
A ╲ B │ a
──────┼──
one   │ 1
```

### Negation / complement

There is a special type constructor `Not()`, whose function is to specify which elements to exclude from the array.  This is similar to negative indices in the language R.  The elements in `Not(elements...)` select all but the indicated elements from the array.

```julia
julia> n[Not(1), :] == n[[2], :] ## note that `n` stays 2-dimensional
true

julia> n[2, Not(:a)] == n[2, [:b, :c]]
true

julia> dodgy[1, Not(Name(30))] == dodgy[1, [1, 2, 4]]
true
```
Both integers and names can be negated.

### Dictionary-style indexing

You can also use a dictionary-style indexing, if you don't want to bother about the order of the dimensions, or make a slice using a specific named dimension:
```julia
julia> n[:A => "one"] == [1, 2, 3]
true

julia> n[:B => :c, :A => "two"] == 6
true

julia> n[:A=>:, :B=>:c] == [3, 6]
true

julia> n[:B=>[:a, :b]] == [1 2; 4 5]
true

julia> n[:A=>["one", "two"], :B=>:a] == [1, 4]
true

julia> n[:A=>[1, 2], :B=>:a] == [1, 4]
true

julia> n[:A=>["one"], :B=>1:2] == [1 2]
true

julia> n[:A=>["three"]] # Throws ArgumentError when trying to access non-existent dimension.
ERROR: ArgumentError: Elements for A => ["three"] not found.
```

### Assignment

Most index types can be used for assignment as LHS
```julia
julia> n[1, 1] = 0
0

julia> n["one", :b] = 1
1

julia> n[:, :c] = 101:102
101:102

julia> n[:B=>:b, :A=>"two"] = 50
50

julia> n
2×3 Named Matrix{Int64}
A ╲ B │   a    b    c
──────┼──────────────
one   │   0    1  101
two   │   4   50  102
```

General functions
--

### Access to the names of the indices and dimensions

```julia
julia> names(n::NamedArray) ## get all index names for all dimensions
2-element Vector{Vector}:
 ["one", "two"]
 [:a, :b, :c]

julia> names(n::NamedArray, 1) ## just for dimension `1`
2-element Vector{String}:
 "one"
 "two"

julia> dimnames(n::NamedArray) ## the names of the dimensions
2-element Vector{Symbol}:
 :A
 :B
```

### Setting the names after construction

Because the type of the keys are encoded in the type of the `NamedArray`, you can only change the names of indices if they have the same type as before.

```julia
 setnames!(n::NamedArray, names::Vector, dim::Integer)
 setnames!(n::NamedArray, name, dim::Int, index:Integer)
 setdimnames!(n::NamedArray, name, dim:Integer)
```

sets all the names of dimension `dim`, or only the name at index `index`, or the name of the dimension `dim`.

### Enameration

Similar to the iterator `enumerate` this package provides an `enamerate` function for iterating simultaneously over both names and values.

```julia
enamerate(a::NamedArray)
```
For example:
```julia
julia> n = NamedArray([1 2 3; 4 5 6], (["one", "two"], [:a, :b, :c]))
2×3 Named Matrix{Int64}
A ╲ B │ a  b  c
──────┼────────
one   │ 1  2  3
two   │ 4  5  6

julia> for (name, val) in enamerate(n)
           println("$name ==  $val")
       end
("one", :a) ==  1
("two", :a) ==  4
("one", :b) ==  2
("two", :b) ==  5
("one", :c) ==  3
("two", :c) ==  6
```

### Aggregating functions

Some functions, when operated on a NamedArray, will a name for the singleton index:
```julia
julia> sum(n, dims=1)
1×3 Named Matrix{Int64}
 A ╲ B │ a  b  c
───────┼────────
sum(A) │ 5  7  9

julia> prod(n, dims=2)
2×1 Named Matrix{Int64}
A ╲ B │ prod(B)
──────┼────────
one   │       6
two   │     120

Aggregating functions are `sum`, `prod`, `maximum`,  `minimum`,  `mean`,  `std`.

### Convert

```julia
convert(::Type{Array}, a::NamedArray)
```

 converts a NamedArray to an Array by dropping all name information.  You can also directly access the underlying array using `n.array`, or use the accessor function `array(n)`.

Methods with special treatment of names / dimnames
--------------------------------------------------

### Concatenation

If the names are identical
for the relevant dimension, these are retained in the results.  Otherwise,
the names are reinitialized to the default "1", "2", ...

In the concatenated direction, the names are always re-initialized.  This may change is people find we should put more effort to check the concatenated names for uniqueness, and keep original names if that is the case.

```julia
julia> hcat(n, n)
2×6 Named Matrix{Int64}
A ╲ hcat │ 1  2  3  4  5  6
─────────┼─────────────────
one      │ 1  2  3  1  2  3
two      │ 4  5  6  4  5  6

julia> vcat(n, n)
4×3 Named Matrix{Int64}
vcat ╲ B │ a  b  c
─────────┼────────
1        │ 1  2  3
2        │ 4  5  6
3        │ 1  2  3
4        │ 4  5  6

```


### Transposition

```julia
julia> n'
3×2 Named LinearAlgebra.Adjoint{Int64, Matrix{Int64}}
B ╲ A │ one  two
──────┼─────────
a     │   1    4
b     │   2    5
c     │   3    6

julia> circshift(n, (1, 2))
2×3 Named Matrix{Int64}
A ╲ B │ b  c  a
──────┼────────
two   │ 5  6  4
one   │ 2  3  1

```
Similar functions: `adjoint`, `transpose`, `permutedims` operate on the dimnames as well.

```julia
julia> rotl90(n)
3×2 Named Matrix{Int64}
B ╲ A │ one  two
──────┼─────────
c     │   3    6
b     │   2    5
a     │   1    4

julia> rotr90(n)
3×2 Named Matrix{Int64}
B ╲ A │ two  one
──────┼─────────
a     │   4    1
b     │   5    2
c     │   6    3
```

 ### Reordering of dimensions in NamedVectors

```julia
julia> v = NamedArray([1, 2, 3], ["a", "b", "c"])
3-element Named Vector{Int64}
A  │
───┼──
a  │ 1
b  │ 2
c  │ 3

julia> Combinatorics.nthperm(v, 4)
3-element Named Vector{Int64}
A  │
───┼──
b  │ 2
c  │ 3
a  │ 1

julia> Random.shuffle(v)
3-element Named Vector{Int64}
A  │
───┼──
b  │ 2
a  │ 1
c  │ 3

julia> reverse(v)
3-element Named Vector{Int64}
A  │
───┼──
c  │ 3
b  │ 2
a  │ 1

julia> sort(1 ./ v)
3-element Named Vector{Float64}
A  │
───┼─────────
c  │ 0.333333
b  │      0.5
a  │      1.0
```

operate on the names of the rows as well

 ### Broadcasts

In broadcasting, the names of the first argument are kept

```julia
julia> ni = NamedArray(1 ./ n.array)
2×3 Named Matrix{Float64}
A ╲ B │        1         2         3
──────┼─────────────────────────────
1     │      1.0       0.5  0.333333
2     │     0.25       0.2  0.166667

julia> n .+ ni
┌ Warning: Using names of left argument
└ @ NamedArrays ~/werk/julia/NamedArrays.jl/src/arithmetic.jl:25
2×3 Named Matrix{Float64}
A ╲ B │       a        b        c
──────┼──────────────────────────
one   │     2.0      2.5  3.33333
two   │    4.25      5.2  6.16667

julia> n .- v'
2×3 Named Matrix{Int64}
A ╲ B │ a  b  c
──────┼────────
one   │ 0  0  0
two   │ 3  3  3

```
This is implemented through `broadcast`.

## Further Development

The current goal is to reduce complexity of the implementation.  Where possible, we want to use more of the `Base.AbstractArray` implementation.

A longer term goal is to improve type stability, this might have a repercussion to the semantics of some operations.

## Related Packages

The Julia ecosystem now has a number of packages implementing the general idea of attaching names to arrays. For some purposes they may be interchangeable. For others, flexibility or speed or support for particular functions may make one preferable.

* [AxisArrays.jl](https://github.com/JuliaArrays/AxisArrays.jl) is of comparable age. It attaches a Symbol to each dimension; this is part of the type thus cannot be mutated after creation.

* [DimensionalData.jl](https://github.com/rafaqz/DimensionalData.jl), [AxisKeys.jl](https://github.com/mcabbott/AxisKeys.jl) and [AxisIndices.jl](https://github.com/Tokazama/AxisIndices.jl) are, to first approximation, post-Julia 1.0 re-writes of that. DimensionalData.jl similarly builds in dimension names, in AxisKeys.jl they are provided by composition with NamedDims.jl, and AxisIndices.jl does not support them. All allow some form of named indexing but the notation varies.

Packages with some overlap but a different focus include:

* [NamedDims.jl](https://github.com/invenia/NamedDims.jl) only attaches a name to each dimension, allowing `sum(A, dims = :time)` and `A[time=3]` but keeping indices the usual integers.

* [LabelledArrays.jl](https://github.com/SciML/LabelledArrays.jl) instead attaches names to individual elements, allowing `A.second == A[2]`.

* [OffsetArrays.jl](https://github.com/JuliaArrays/OffsetArrays.jl) shifts the indices of an array, allowing say `A[-3]` to `A[3]` to be the first & last elements.
