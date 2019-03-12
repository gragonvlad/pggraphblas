# summary

pggraphblas is a postgres extension that bridges [The GraphBLAS
API](http://graphblas.org) with the
[PostgreSQL](https://postgresql.org) object relational database.

GraphBLAS is a sparse linear algebra API optimized for processing
graphs encoded as sparse matrices and vectors.  In addition to common
real/integer matrix algebra operations, GraphBLAS supports up to 960
different "semiring" algebra operations, that can be used as basic
building blocks to implement a wide variety of graph algorithms.

pggraphblas leverages the expertise in the field of sparse matrix
programming by [The GraphBLAS Forum](http://graphblas.org) and uses
the
[SuiteSparse:GraphBLAS](http://faculty.cse.tamu.edu/davis/GraphBLAS.html)
API implementation. SuiteSparse:GraphBLAS is brought to us by the work
of [Dr. Tim Davis](http://faculty.cse.tamu.edu/davis/welcome.html),
professor in the Department of Computer Science and Engineering at
Texas A&M University.  [News and
information](http://faculty.cse.tamu.edu/davis/news.html) can provide
you with a lot more background information, in addition to the
references below.

# intro

For a long time, mathematicians have known that matrices are powerful
representations of graphs, as described [in this mathmatical
introduction to
GraphBLAS](http://www.mit.edu/~kepner/GraphBLAS/GraphBLAS-Math-release.pdf)
by [Dr. Jermey Kepner](http://www.mit.edu/~kepner/) head and founder
of [MIT Lincoln Laboratory Supercomputing
Center](http://news.mit.edu/2016/lincoln-laboratory-establishes-supercomputing-center-0511).

As Kepner's paper describes, there are two useful matrix
representations of graphs: [Adjacency
Matrices](https://en.wikipedia.org/wiki/Adjacency_matrix) and
[Incidence Matrices](https://en.wikipedia.org/wiki/Incidence_matrix).
For this introduction we will focus on the adjacency type as they are
simpler, but the same ideas apply to both, and it is easy to switch
back and forth between them.

![Alt text](./docs/AdjacencyMatrix.svg)

(Image Credit: [Dr. Jermey Kepner](http://www.mit.edu/~kepner/))

On the left is a *directed* graph, and on the right, the adjacency
matrix that represents it. The matrix has a row and column for every
vertex.  If there is an going from node A to B, then there will be a
value present in the intersection of As row with Bs column.  For
example, vertex 1 connects to 4, so there is a value (dot) at the
intersction of the first row and the fourth column.  4 also connects
*back* to 1 so there are two values in the matrix to represent these
two edges, the one at the (1, 4) position and the other at the (4,1)
position.

One practical problem with matrix-encoding graphs is that most
real-world graphs tend to be sparse, as above, only 12 of 49 possible
elements have a value. Those that have values tend to be scattered
uniformally across the matrix (for "typical" graphs), so dense linear
algebra libraries like BLAS or numpy do not encode or operate on them
efficiently, as the relevant data is mostly empty memory with actual
data elements spaced far apart.  This wastes memory and cpu resources,
and defeats CPU caching mechanisms.

For example, suppose a fictional social network has 1 billion users,
and each user has about 100 friends, which means there are about 100
billion (1e+11) connections in the graph.  A dense matrix large enough
to hold this graph would need (1 billion)^2 or
(1,000,000,000,000,000,000), a "quintillion" elements, but only 1e+11
of them would have meaningful values, leaving only 0.0000001 of the
matrix being utilized.

By using a sparse matrix instead of dense, only the elements used are
actually stored in the matrix. The parts of the matrix with no value
are interpreted as an "algebraic zero" value, which might not be the
actual number zero, but other values like positive or negative
infinity depending on the particular semiring operations applied to
the matrix.  The math used with sparse matrices is exactly the same as
dense, the sparsity of the data doesn't matter to the math, but it
does matter to how efficiently the matrix is implemented internally.

pggraphblas is a postgres extension that provides access to two new
types: `matrix` and `vector`, as well as the GraphBLAS api to
manipulate these types.  Aggregate functions are provided to build
matrices from SQL queries, and set-returning functions are also
provided to turn graphs back into relational sets.  From a PostgreSQL
point of view, matrices look a little bit like arrays, being stored as
variable length column values.

# matrix multplication

They key operation of GraphBLAS is the matrix multiply as provided by
the `mxm` (matrix times matrix), `mxv` (matrix times vector), and
`vxm` (vector times matrix) functions.  Matrix multplication has a
remarkable property of being useful for finding the neighbors of any
node in a graph algorithm.  By using different combinations of
operations (semiring) different graph algorithms can step and
accumulate different results, interpreting the data in unique ways,
even over the same graphs.

![Alt text](./docs/AdjacencyMatrixBFS.svg)

(Image Credit: [Dr. Jermey Kepner](http://www.mit.edu/~kepner/))

Above is the same graph and matrix from before, shown here as `A`.
Next to it we see a *vector* `v`.  When you multiply the transpose of
A and v, the result vector contains all of the neighboring nodes for
the nodes specified in `v`.  In this case, `v` contains a value for
node 4, shown in red.  By multiplying the matrix by the input vector,
the result is the output vector with the 4's two neighboring nodes, 1
and 3.  Interating this multiplication process produces the most
common graph operation: [breadth-first
search](https://en.wikipedia.org/wiki/Breadth-first_search).

    create function bfs(A matrix, source bigint) returns vector as $$
    declare
        n bigint := nrows(A);                          -- The number of result rows.
        v vector := vector_integer(n);                 -- int32 result vector of vertex levels.
        q vector := assign(vector_bool(n), false);     -- bool mask of completed vertices.
        level integer := 0;                            -- Start at level 1.
        not_done bool := true;                         -- Flag to indicate still work to do.
    begin
        q := set_element(q, source, true);             -- Set the source element to done.

        while not_done and level <= n loop             -- While still work to do.
            v := assign(v, level, mask=>q);            -- Assign the current level to all

            q := mxv(transpose(A), q, q,               -- Multiply q<mask> = T(A)q,
                semiring=>'lor_land_bool',             -- using LOR_LAND_BOOL semiring
                mask=>v,                               -- only those *not* masked
                dmask=>'scmp',                         -- by complementing the mask
                doutp=>'replace');                     -- clearing results in q first

            not_done := reduce_bool(q);                -- are there more neighbors?

            level := level + 1;                        -- increment the level
        end loop;
        return v;
    end;
    $$ language plpgsql;

 The above code is written in `plpgsql` which is postgres' procedural
 query language.  This language works well with pggraphblas
 algorithmic approach.

# references

[One page poster summary of GraphBLAS](https://resources.sei.cmu.edu/asset_files/Poster/2016_020_001_484268.pdf)

[GraphBLAS: A Programming Specification for Graph Analysis](https://resources.sei.cmu.edu/asset_files/Presentation/2016_017_001_474272.pdf)

[Lower Latency Graph Queries in Cypher with Redis GraphRoi Lipman, Redis LabsTim Davis, Texas A&M U](https://www.youtube.com/watch?v=xnez6tloNSQ)

[A good introduction to semirings Part I](https://www.youtube.com/watch?v=Gd_VT_Nj8Xw) [Part II](https://www.youtube.com/watch?v=dluPFbuq6zs)

[A good introduction to abstract algebra](https://www.youtube.com/playlist?list=PLi01XoE8jYoi3SgnnGorR_XOW3IcK-TP6)

[Graph algorithms via SuiteSparse:GraphBLAS: triangle counting and K-truss](http://faculty.cse.tamu.edu/davis/GraphBLAS_files/Davis_HPEC18.pdf)

# development

If you have docker installed, run `./test.sh psql` to build a docker
container with postgres:11 and GraphBLAS compiled with debug symbols
on.  This will eventually drop you into a psql interpreter.  You can
run the tests from that point with `\i /tests/test.sql` There are many
tests, in general one for each integrated feature.

# types

GraphBLAS has an extremely rich type system that encompases not only
common artithmetic algegbraic operations like addition and
multiplication, but also many other operations using algebraic
structures called semirings.  The combinations of 7 matrix types and
960 semiring operations offer a huge number of building blocks for
solving graph problems.

PostgreSQL is a strongly typed language that comes with many built-in
data types, the following types map from GraphBLAS to PostgreSQL:

    - GrB_BOOL maps to bool
    - GrB_INT8 maps to char (aka character(1))
    - GrB_INT16 maps to smallint
    - GrB_INT32 maps to integer
    - GrB_INT64 maps to bigint
    - GrB_FP32 maps to real
    - GrB_FP64 maps to float (aka "double precision")

Postgres does not support unsigned integers like GraphBLAS (is there a
workaround?  support the uint extension?)

pggraphblas matrix and vector objects have an intrinsic type of data
that they store.  So, a matrix can store booleans, and various sized
integers and floats.  Once a matrix is created with a specific type it
cannot be changed.

When combining matrices (and vectors) it is possible to not only use
the common arithmetic plus and multiply operators on any given type,
it is also possible to mix various semiring operations on any
supported type.  Put example here of {R,min,+,0,+inf}.

# API

Pggraphblas tries to adhere closely to the spirit of the GraphBLAS C
API.

## dense vector

Dense vectors contain as many elements as their size.  They can be
constructed from arrays by calling `vector(array[])` or casting from
an array.  At the moment, there is no human parsable text
representation for vectors or matrices, so pggraphblas provides a
`print` function that can give a text description of vectors or
matrices:

    postgres=# select print(vector(array[1,2,3]));
                                     print                                 
    -----------------------------------------------------------------------
                                                                          +
     GraphBLAS vector: A->V                                               +
     nrows: 3 ncols: 1 max # entries: 3                                   +
     format: standard CSC vlen: 3 nvec_nonempty: 1 nvec: 1 plen: 1 vdim: 1+
     hyper_ratio 0.0625                                                   +
     GraphBLAS type:  int32_t size: 4                                     +
     number of entries: 3                                                 +
     column: 0 : 3 entries [0:2]                                          +
         row 0: int32 1                                                   +
         row 1: int32 2                                                   +
         row 2: int32 3                                                   +

    (1 row)

Vectors and matrices can be compared for equality:

    postgres=# select vector(array[1,2,3]) = array[1,2,3]::vector;
     t
    
As this vector is dense its size and number of values are the same:

    postgres=# select size(vector(array[1,2,3])) = nvals(array[1,2,3]::vector);
    t

The supported types are bigint (64 bit), integer (32 bit), smallint
(16 bit), bool (t/f), real (32 bit single precision) and float (64 bit
double precision).  While GraphBLAS supports unsigned types, all
currently supported types are signed but unsigned may be supported in
the future.

Sparse vectors can be constructed by calling vector with two array
arguments, the first array are the indexes, and must be coercible to
bigint, and the second array are the values, and can be any supported
type:

    postgres=# select print(vector(array[1,4,9], array[1,2,3]));
                                     print                                 
    -----------------------------------------------------------------------
                                                                          +
     GraphBLAS vector: A->V                                               +
     nrows: 3 ncols: 1 max # entries: 3                                   +
     format: standard CSC vlen: 3 nvec_nonempty: 1 nvec: 1 plen: 1 vdim: 1+
     hyper_ratio 0.0625                                                   +
     GraphBLAS type:  int32_t size: 4                                     +
     number of entries: 3                                                 +
     column: 0 : 3 entries [0:2]                                          +
         row 1: int32 1                                                   +
         row 4: int32 2                                                   +
         row 9: int32 3                                                   +

    (1 row)

Both arrays must be the exact same length.  The size of the vector
will be the maximum element from the first list, but the vector will
only store the values that are defined.  This is the "sparse" nature
of the object, it can be efficiently used in matrix math involving
millions of elements, when only a few actual values are in play.

If you need to create a spare vector whose size is larger than the
maximum index element, an optional size parameter can be passed to the
constructor:

    postgres=# select print(vector(array[1,2,3], array[0,1,2], 20));
                                     print                                  
    ------------------------------------------------------------------------
                                                                           +
     GraphBLAS vector: A->V                                                +
     nrows: 20 ncols: 1 max # entries: 3                                   +
     format: standard CSC vlen: 20 nvec_nonempty: 1 nvec: 1 plen: 1 vdim: 1+
     hyper_ratio 0.0625                                                    +
     GraphBLAS type:  int32_t size: 4                                      +
     number of entries: 3                                                  +
     column: 0 : 3 entries [0:2]                                           +
         row 0: int32 1                                                    +
         row 1: int32 2                                                    +
         row 2: int32 3                                                    +

    (1 row)

Vectors can be added and multiplied in element-wise fashion using
either operator or functional notation:

    postgres=# select vector(array[1,2,3]) + vector(array[1,2,3]) = vector(array[2,4,6]);
     t

    postgres=# select vector(array[1,2,3]) * vector(array[1,2,3]) = vector(array[1,4,9]);
     t

    postgres=# select ewise_add(vector(array[1,2,3]), vector(array[1,2,3])) = vector(array[2,4,6]);
     t

    postgres=# select ewise_mult(vector(array[1,2,3]) * vector(array[1,2,3])) = vector(array[1,4,9]);
     t

Element wise operations also have another important property,
elementwise addition takes the union of all the vector positions of
both operands, whereas elementwise multiplication takes only the
common intersection of it's operand's indexes.  This propery is also
true for matrices as shown below.

## matrix

Matrices can be constructed with `matrix(bigint[], bigint[], array[])`
where the first array are the row indexes, the second array the column
indexes, and the third array the values.  Matrices support all the
same types as vectors.  A useful way to construct matrices is with
array aggregate functions to build them from tables, for example:

    postgres=# create table test (
        i integer,
        j integer
        );
    CREATE TABLE
    
    postgres=# insert into test (i, j) values
        (1, 4),
        (1, 2),
        (2, 7),
        (2, 5),
        (3, 6),
        (4, 3),
        (4, 1),
        (5, 6),
        (6, 3),
        (7, 3);
    INSERT 0 10

    postgres=# select print(matrix(array_agg(i), array_agg(j), array_agg(true))) from test;
                                       print                                   
    ---------------------------------------------------------------------------
                                                                              +
     GraphBLAS matrix: A->M                                                   +
     nrows: 10 ncols: 10 max # entries: 10                                    +
     format: standard CSR vlen: 10 nvec_nonempty: 7 nvec: 10 plen: 10 vdim: 10+
     hyper_ratio 0.0625                                                       +
     GraphBLAS type:  bool size: 1                                            +
     number of entries: 10                                                    +
     row: 1 : 2 entries [0:1]                                                 +
         column 2: bool 1                                                     +
         column 4: bool 1                                                     +
     row: 2 : 2 entries [2:3]                                                 +
         column 5: bool 1                                                     +
         column 7: bool 1                                                     +
     row: 3 : 1 entries [4:4]                                                 +
         column 6: bool 1                                                     +
     row: 4 : 2 entries [5:6]                                                 +
         column 1: bool 1                                                     +
         column 3: bool 1                                                     +
     row: 5 : 1 entries [7:7]                                                 +
         column 6: bool 1                                                     +
     row: 6 : 1 entries [8:8]                                                 +
         column 3: bool 1                                                     +
     row: 7 : 1 entries [9:9]                                                 +
         column 3: bool 1                                                     +

    (1 row)

Matrices can also be turned back into relational tuples using
`matrix_elements_<type>`:

    postgres=# select * from matrix_elements_bool((select matrix(array_agg(i), array_agg(j), array_agg(true)) from test));
     row | col | value 
    -----+-----+-------
       1 |   2 | t
       1 |   4 | t
       2 |   5 | t
       2 |   7 | t
       3 |   6 | t
       4 |   1 | t
       4 |   3 | t
       5 |   6 | t
       6 |   3 | t
       7 |   3 | t
    (10 rows)

Empty matrices can be constructed with bigint arguments:

    postgres=# select print(matrix_integer(10, 10));
                                       print                                    
    ----------------------------------------------------------------------------
                                                                               +
     GraphBLAS matrix: A->M                                                    +
     nrows: 10 ncols: 10 max # entries: 0                                      +
     format: hypersparse CSR vlen: 10 nvec_nonempty: 0 nvec: 0 plen: 1 vdim: 10+
     hyper_ratio 0.0625                                                        +
     GraphBLAS type:  int32_t size: 4                                          +
     empty                                                                     +
     number of entries: 0                                                      +

    (1 row)    

## mxm

Matrices can be multiplied using a default "plus/times" semiring via
the '*' operator, or called explicity using functional notation so
that different semirings can be used to carry out the operation.
Functional notation also allows passing other graphblas options like a
masking matrix, an accumulator operation, and a graphblas descriptor
object which can specify replacement and transposition rules for
inputs and outputs.

## mxv

Matrix-vector multiplication.

## vxm

Vector-matrix multiplication.

## ewise add

Elementwise matrix addition.

## ewise mul

Elementwise matrix multiplication.

## xtract

Extracting subgraphs.

## assign

Assigning subgraphs.

## apply

Applying functions to graphs.

## select

Selecting elements of a matrix.

## reduce

Matrix and vector reduction.

## transpose

Matrix transpose.

## kron

Kronecker product.

## random_graph

Random graph generation.
