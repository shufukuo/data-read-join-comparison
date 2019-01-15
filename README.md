# A simple comparison
Comaring processing time of loading CSV files and joining data frames in Julia, Python, and R

### Data resource
MovieLens 20M Dataset: https://grouplens.org/datasets/movielens/

### Hardware
* Intel Core i7-8550U CPU
* 32 GB DDR4 2 DIMM RAM
* 512 GB PCIe M.2 SSD

### Software
* Windows 10 Home
* Julia 1.0.3
* Python 3.6.5
* R 3.5.2

***

### Loading CSV files

#### Julia
<pre><code>using CSV
t0 = time_ns()
for s in ["links", "movies", "ratings", "tags"]
	eval(
		Meta.parse( s * " = " * "CSV.read(\"" * s * ".csv\")" )
	)
end
(time_ns()-t0)/1e9
</code></pre>
