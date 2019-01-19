# A simple comparison among Julia, Python, and R
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
The pandas library is used.
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
The basic `utils::read.csv` and `data.table::fread` are used.

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
While adding a `require` command and replacing `read.csv` with `fread`, something amazing happens.
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
It finishes processing within a second! The total memory usage is 452.6 MB, and the object `ratings` takes about 381 MB.

#### microbenchmark
Additionally, the packge microbenckmark can be used to do a quick benckmarking easily in R. In this case, I only try loading the ratings.csv, and the `read.csv.sql` function in the packege sqldf is included in addition.
<pre><code>require(sqldf)
require(data.table)
require(microbenchmark)

microbenchmark( times=10L,
    read.csv     = read.csv("ratings.csv"),
    read.csv.sql = read.csv.sql("ratings.csv"),
    fread        = fread("ratings.csv")
)
</code></pre>
The output is shown below.
<pre><code>Unit: milliseconds
         expr       min         lq       mean     median        uq       max neval
     read.csv 50431.894 50615.3588 52343.2035 50972.7144 52048.159 61703.993    10
 read.csv.sql 43367.668 43581.3915 44894.9252 45259.6581 45757.378 46788.872    10
        fread   432.048   489.0825   930.0462   973.6845  1080.307  1447.417    10
</code></pre>

### Summary table
| Language/Package | Proc Time (Sec) | Reltive | MEM Total (MB) | MEM `ratings` (MB) |
| :--------------- | --------------: | ------: | -------------: | -----------------: |
| Julia/CSV        |           20.31 |  36.268 |         1522.2 |              663.5 |
| Python/pandas    |            7.06 |  12.607 |          672.4 |              610.4 |
| R/utils          |           69.13 | 123.446 |         2635.5 |              381.5 |
| R/data.table     |            0.56 |   1.000 |          452.6 |              381.5 |

## Joining data frames
Besides I/O performance, computational performance is also interesting to me. So I do some simple aggregations and joins. The part of data in the movies.csv and ratings.csv are shown below. For more details, please refer to http://files.grouplens.org/datasets/movielens/ml-20m-README.html.
<pre><code>> head(movies)
   movieId                              title                                      genres
1:       1                   Toy Story (1995) Adventure|Animation|Children|Comedy|Fantasy
2:       2                     Jumanji (1995)                  Adventure|Children|Fantasy
3:       3            Grumpier Old Men (1995)                              Comedy|Romance
4:       4           Waiting to Exhale (1995)                        Comedy|Drama|Romance
5:       5 Father of the Bride Part II (1995)                                      Comedy
6:       6                        Heat (1995)                       Action|Crime|Thriller
> head(ratings)
   userId movieId rating  timestamp
1:      1       2    3.5 1112486027
2:      1      29    3.5 1112484676
3:      1      32    3.5 1112484819
4:      1      47    3.5 1112484727
5:      1      50    3.5 1112484580
6:      1     112    3.5 1094785740
</code></pre>
To know the rating count and the mean rating of each movie, I use the pandas library in Python, and the sqldf and data.table packages in R to do the aggregation with join. The following shows the codes in Python and R.

Python/pandas
<pre><code>import numpy as np

t0 = time.time()

cnt_movie_ratings = ratings.groupby('movieId').agg( {'movieId': np.size, 'rating': np.mean} ) \
    .rename( columns = {'movieId': 'cnt', 'rating': 'mean_rating'} ) \
    .reset_index() \
    .merge( movies, how='left', on='movieId' ) \
    .sort_values( ['cnt', 'mean_rating'], ascending=[False, False] )

print( time.time()-t0 )
</code></pre>

R/sqldf
<pre><code>require(sqldf)

system.time(
    cnt_movie_ratings <- sqldf( "
        select dt1.*, dt2.title, dt2.genres
        from
            (
            select movieId, count(movieId) as cnt, avg(rating) as mean_rating
            from ratings
            group by movieId
            ) as dt1
        left join movies as dt2
        on dt1.movieId = dt2.movieId
        order by cnt desc, mean_rating desc
        "
    )
)
</code></pre>

R/data.table
<pre><code>require(data.table)

system.time(
    cnt_movie_ratings <- movies[
        ratings[ , .(cnt=.N, mean_rating=mean(rating)), keyby=movieId ],
        on="movieId"
        ] [
        order(-cnt, -mean_rating)
        ]
)
</code></pre>

And the performance is shown below. The pandas and data.table are quite fast, comparing to sqldf. I am also interested to see the performance in SAS but I am not able to do so with my PC. It would be great to know that.

| Language/Package | Proc Time (Sec) | Reltive |
| :--------------- | --------------: | ------: |
| Python/pandas    |            1.66 |   3.018 |
| R/sqldf          |           21.21 |  38.564 |
| R/data.table     |            0.55 |   1.000 |

Finally, the resulting table looks like this.
<pre><code>   movieId                                     title                           genres   cnt mean_rating
1:     296                       Pulp Fiction (1994)      Comedy|Crime|Drama|Thriller 67310    4.174231
2:     356                       Forrest Gump (1994)         Comedy|Drama|Romance|War 66172    4.029000
3:     318          Shawshank Redemption, The (1994)                      Crime|Drama 63366    4.446990
4:     593          Silence of the Lambs, The (1991)            Crime|Horror|Thriller 63299    4.177057
5:     480                      Jurassic Park (1993) Action|Adventure|Sci-Fi|Thriller 59715    3.664741
6:     260 Star Wars: Episode IV - A New Hope (1977)          Action|Adventure|Sci-Fi 54502    4.190672
</code></pre>
