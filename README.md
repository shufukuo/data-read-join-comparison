# A simple comparison
Comaring processing time of loading CSV files and joining data frames in Julia, Python, and R.

### Data resource
MovieLens 20M Dataset: https://grouplens.org/datasets/movielens/

The datasets links.csv, movies.csv, ratings.csv, and tags.csv are used for the test.

### Hardware/Software
* Intel Core i7-8550U CPU
* 32 GB DDR4 2 DIMM RAM
* 512 GB PCIe M.2 SSD
* Windows 10 Home
* Julia 1.0.3
* Python 3.6.5 (Anaconda3 5.2.0)
* R 3.5.2

## Loading CSV files

### Julia
The pure-Julia library CSV.jl is used.
<pre><code>using CSV

t0 = time_ns()

for s in ["links", "movies", "ratings", "tags"]
    eval(
        Meta.parse( s * " = " * "CSV.read(\"" * s * ".csv\")" )
    )
end

(time_ns() - t0) / 1e9
</code></pre>
It takes about 20 seconds to load the 4 datasets.
<pre><code>julia> (time_ns() - t0) / 1e9
20.311318892
</code></pre>
In this comparison, I only record the total memory usage after processing for all 3 programming languages, and there is a total of 1522.2 MB RAM allocated by Julia. But the object size of `ratings` (the biggest dataset among the 4 datasets, 20,000,263 rows by 4 columns) is checked, and the following is obtained.
<pre><code>
julia> @time Base.summarysize(ratings)
161.765047 seconds (564.16 M allocations: 20.599 GiB, 7.00% gc time)
695727632
</code></pre>
The object `ratings` takes about 663 MB.

### Python
The pandas library is used for comparison.
<pre><code>import pandas as pd
import time

t0 = time.time()

for s in ["links", "movies", "ratings", "tags"]:
    exec( s + """ = pd.read_csv( '""" + s + """.csv')""" )

print( time.time() - t0 )
</code></pre>
<pre><code>>>> print( time.time() - t0 )
7.060875177383423
</code></pre>
The processing time is about 7 seconds. The total memory usage is 672.4 MB, and the object `ratings` takes about 610 MB in Python.
<pre><code>>>> import sys
>>> sys.getsizeof(ratings)
640008520
</code></pre>

### R
In R, the basic `utils::read.csv` and `data.table::fread` from the package data.table are compared.

#### utils::read.csv

<pre><code>system.time(

for ( s in c("links", "movies", "ratings", "tags") ) {
    assign( s, read.csv( paste( s, ".csv", sep="" ) ) )
}

)
</code></pre>
<pre><code> user  system elapsed 
67.65    1.39   69.13
</code></pre>
The basic `read.csv` function takes more than a minute to load the 4 datasets, and the total memory usage is up to 2635.5 MB.
<pre><code>> object.size(ratings)
400006368 bytes
</code></pre>
However, the object `ratings` takes only about 381 MB. It might be a garbage collection issue.

#### data.table::fread
<pre><code>require(data.table)

system.time(
    for ( s in c("links", "movies", "ratings", "tags") ) {
        assign( s, fread( paste( s, ".csv", sep="" ) ) )
    }
)
</code></pre>
<pre><code>user  system elapsed 
2.71    0.44    0.56
</code></pre>
<pre><code>> object.size(ratings)
400006992 bytes
</code></pre>
The processing time is less than 3 seconds. The total memory usage is 452.6 MB, and the object `ratings` takes about 381 MB.

#### microbenchmark
Additionally, the packge microbenckmark can be used to do a quick benckmarking easily in R. In this case, I only try loading ratings.csv, and I also compare `read.csv.sql` function in the packege sqldf.
<pre><code>require(sqldf)
require(data.table)
require(microbenchmark)

microbenchmark( times=10L,
    read.csv     = read.csv("ratings.csv"),
    read.csv.sql = read.csv.sql("ratings.csv"),
    fread        = fread("ratings.csv")
)
</code></pre>
This results in the following output.
<pre><code>

</code></pre>
