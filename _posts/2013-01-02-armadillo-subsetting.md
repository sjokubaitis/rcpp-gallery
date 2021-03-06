---
title: Armadillo subsetting
author: Dirk Eddelbuettel 
license: GPL (>= 2)
tags: armadillo matrix vector featured
mathjax: true
updated: Nov 19, 2016
updateauthor: James Joseph Balamuta
summary: This example shows how to subset Armadillo vectors and matrices 
    non-contiguous views
layout: post
src: 2013-01-02-armadillo-subsetting.Rmd
---

## Introduction

Subsetting in armadillo is a popular topic that frequently appears
on StackOverflow. Prinicipally, the occurrences are due to the unique approach
that armadillo takes on performing subset operations. Within this topic, light
will be shed on the applicable use of armadillo's subsetting capabilities as it
relates to non-contiguous 
[submatrix views](http://arma.sourceforge.net/docs.html#submat). 

## Vector Subset and Assignment

To begin, we start with a recent inquiring on [StackOverflow](http://stackoverflow.com/questions/38541787/change-elements-at-given-positions-in-an-armavec-with-corresponding-elements-i) that dealt with
the proposition of isolating pre-known locations and assigning the locations
new values. The *R* equivalent would be in the lines of:


{% highlight r %}
(v     <- 1:10)     # Generate and Preview Values
{% endhighlight %}



<pre class="output">
 [1]  1  2  3  4  5  6  7  8  9 10
</pre>



{% highlight r %}
pos    <- c(2,4,10) # Replacement IDs
rv     <- c(6,3,1)  # Replacement Values
v[pos] <- rv        # Change values
v                   # Observe updated vector
{% endhighlight %}



<pre class="output">
 [1] 1 6 3 3 5 6 7 8 9 1
</pre>

Within `armadillo`, the [`submatrix views`](http://arma.sourceforge.net/docs.html#submat) 
that we are most interested in to replicate this behavior is `.elem()`. 
The main difference of subsetting with this function is the need to use an 
`arma::uvec` containing the locations of elements to extract. 
The `u` prepended to the `vec` object standards for `unsigned int`, which are
only able to store positive integer numbers (e.g. 0, 1, 2,..., 42, and so on.)


{% highlight cpp %}
#include <RcppArmadillo.h>
// [[Rcpp::depends(RcppArmadillo)]]

// [[Rcpp::export]]
arma::rowvec arma_sub(arma::rowvec x, arma::uvec pos, arma::vec vals) {
    
    x.elem(pos) = vals;  // Subset by element position and set equal to
  
    return x;
}
{% endhighlight %}

To execute this code, we would use:


{% highlight r %}
(v <- 1:10)               # Generate and Preview Values
{% endhighlight %}



<pre class="output">
 [1]  1  2  3  4  5  6  7  8  9 10
</pre>



{% highlight r %}
arma_sub(v, pos - 1, rv)  # Note the pos shift from R to C++ index
{% endhighlight %}



<pre class="output">
     [,1] [,2] [,3] [,4] [,5] [,6] [,7] [,8] [,9] [,10]
[1,]    1    6    3    3    5    6    7    8    9     1
</pre>

## Finding Values to Subset

Another common operation is to use logical indexing to ascertain where within
a vector certain conditions are met. For instance, to find values that are 
greater than or equal to 0.5 in *R* the operation is governed by:


{% highlight r %}
set.seed(112)        # Set seed for reproducibility
(v <- runif(10))     # Generate and Preview Values
{% endhighlight %}



<pre class="output">
 [1] 0.37667253 0.92201944 0.99187774 0.97723602 0.23629728 0.10706678
 [7] 0.03915213 0.72802690 0.13023496 0.39995203
</pre>



{% highlight r %}
v[v >= 0.5] <- 1     # Change values
v                    # Observe updated vector
{% endhighlight %}



<pre class="output">
 [1] 0.37667253 1.00000000 1.00000000 1.00000000 0.23629728 0.10706678
 [7] 0.03915213 1.00000000 0.13023496 0.39995203
</pre>

Within `armadillo`, the use of logical values to subset is not possible without [`arma::find`](http://arma.sourceforge.net/docs.html#find)
to replicate *R*'s logical index subset behavior. However, per the previous
discussion, the values that `.elem()` takes requires indices. Therefore,
the [`arma::find`](http://arma.sourceforge.net/docs.html#find)
function is taking logical values and convert them into index positions. This
operation is akin to the process behind *R*'s `which()` function.  Furthermore,
to specify a specific value that should be assigned to the subset, the use of
[`.fill()`](http://arma.sourceforge.net/docs.html#fill) is required.


{% highlight cpp %}
#include <RcppArmadillo.h>
// [[Rcpp::depends(RcppArmadillo)]]

// [[Rcpp::export]]
arma::rowvec arma_sub_cond(arma::rowvec x, double val, double replace) {
    
    arma::uvec ids = find(x >= val); // Find indices
  
    x.elem(ids).fill(replace);       // Assign value to condition
  
    return x;
}
{% endhighlight %}


{% highlight r %}
set.seed(112)            # Set seed for reproducibility
(v <- runif(10))         # Generate and preview Values
{% endhighlight %}



<pre class="output">
 [1] 0.37667253 0.92201944 0.99187774 0.97723602 0.23629728 0.10706678
 [7] 0.03915213 0.72802690 0.13023496 0.39995203
</pre>



{% highlight r %}
arma_sub_cond(v, 0.5, 1)
{% endhighlight %}



<pre class="output">
          [,1] [,2] [,3] [,4]      [,5]      [,6]       [,7] [,8]     [,9]
[1,] 0.3766725    1    1    1 0.2362973 0.1070668 0.03915213    1 0.130235
        [,10]
[1,] 0.399952
</pre>

## Subsetting a Matrix

As was the case with vectors, elements can be extracted from a matrix using
both the [`arma::find`](http://arma.sourceforge.net/docs.html#find) function
to obtain indices and the [`.elem()` function in submatrix views](http://arma.sourceforge.net/docs.html#submat)
to extract values. For example, to know the values of `M * M'` that are greater 
or equal to 100, one could do:


{% highlight cpp %}
#include <RcppArmadillo.h>
// [[Rcpp::depends(RcppArmadillo)]]

// [[Rcpp::export]]
arma::vec matrix_sub(arma::mat M) {
    arma::mat Z = M * M.t();                        
    arma::vec v = Z.elem( find( Z >= 100 ) ); // Find IDs & obtain values
    return v;
}
{% endhighlight %}

The result:


{% highlight r %}
(M <- matrix(1:9, 3, 3))   # Construct & Display Matrix
{% endhighlight %}



<pre class="output">
     [,1] [,2] [,3]
[1,]    1    4    7
[2,]    2    5    8
[3,]    3    6    9
</pre>



{% highlight r %}
matrix_sub(M)
{% endhighlight %}



<pre class="output">
     [,1]
[1,]  108
[2,]  108
[3,]  126
</pre>

## Obtaining Values with Matrix Subscripts

Another form of subsetting within matrices is done using matrix notation.
Matrix notation allows for the specification of an element based on
their row and column location. This question recently popped up on StackOverflow
as the non-continuous submatrix views were initially causing the user problems.


{% highlight r %}
(M <- matrix(1:9, 3, 3))   # Construct & Display Matrix
{% endhighlight %}



<pre class="output">
     [,1] [,2] [,3]
[1,]    1    4    7
[2,]    2    5    8
[3,]    3    6    9
</pre>



{% highlight r %}
(locs <- rbind(c(1, 2),    # List non-continuous locations
               c(3, 1),
               c(2, 3)))
{% endhighlight %}



<pre class="output">
     [,1] [,2]
[1,]    1    2
[2,]    3    1
[3,]    2    3
</pre>



{% highlight r %}
M[locs]                    # Subset the matrix
{% endhighlight %}



<pre class="output">
[1] 4 3 8
</pre>


As one [StackOverflow user discovered](http://stackoverflow.com/questions/37492800/extract-elements-from-a-matrix-based-on-the-row-and-column-indices-with-armadill),
the non-continuous submatrix views obtained via 
`X.submat( vector_of_row_indices, vector_of_column_indices )` does **not** provide
the traditional *R* matrix subset. Within `armadillo` 7.200, the ability to
supply multiple matrix notations to 
[`sub2ind()`](http://arma.sourceforge.net/docs.html#sub2ind) 
was added. However, it requires a different matrix format than *R*. Instead of
supplying the row and column as one *row* in the matrix, one must supply 
each observation as a *column* in the matrix. That is, the subscript matrix
must be $$2 \times n$$, where $$n$$ is the number of matrix locations. The first
row is associated with row location of the element in the matrix and 
the second row is requires the column location of the element. 


{% highlight cpp %}
#include <RcppArmadillo.h>
// [[Rcpp::depends(RcppArmadillo)]]

// [[Rcpp::export]]
arma::rowvec matrix_locs(arma::mat M, arma::umat locs) {
  
    arma::uvec eids = sub2ind( size(M), locs ); // Obtain Element IDs
    arma::vec v  = M.elem( eids );              // Values of the Elements
    
    return v.t();                               // Transpose to mimic R
}
{% endhighlight %}

The result:


{% highlight r %}
cpp_locs <- locs - 1       # Shift indices from R to C++

(cpp_locs <- t(cpp_locs))  # Transpose matrix for 2 x n form
{% endhighlight %}



<pre class="output">
     [,1] [,2] [,3]
[1,]    0    2    1
[2,]    1    0    2
</pre>



{% highlight r %}
matrix_locs(M, cpp_locs)   # Subset the matrix
{% endhighlight %}



<pre class="output">
     [,1] [,2] [,3]
[1,]    4    3    8
</pre>

## Converting between `umat` matrix types to standard `arma` types.

Awhile back, on [StackOverflow
question](http://stackoverflow.com/questions/10212247/conversion-from-armaumat-to-armamat),
someone asked how convert to convert from `arma::umat` to `arma::mat`.  The 
former is the format we have seen used for `arma::find`.

For the particular example at hand, a call to the `conv_to<T>()`
converter provided the solution. For posterity, we rewrite the answer here.


{% highlight cpp %}
#include <RcppArmadillo.h>
// [[Rcpp::depends(RcppArmadillo)]]

// [[Rcpp::export]]
arma::mat matrix_convert(arma::mat M) {
    
    arma::umat a = trans(M) > M;                      // Logical condition 
  
    arma::mat  N = arma::conv_to<arma::mat>::from(a); // Convert to double matrix
    
    return N;
}
{% endhighlight %}


{% highlight r %}
(M <- matrix(1:9, 3, 3)) # Construct & Display Matrix 
{% endhighlight %}



<pre class="output">
     [,1] [,2] [,3]
[1,]    1    4    7
[2,]    2    5    8
[3,]    3    6    9
</pre>



{% highlight r %}
matrix_convert(M)        # Boolean display of M^T > M
{% endhighlight %}



<pre class="output">
     [,1] [,2] [,3]
[1,]    0    0    0
[2,]    1    0    0
[3,]    1    1    0
</pre>

