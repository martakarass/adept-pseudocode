

## Pseudocode intro notes

##### Paper reference

- Karas, M., Straczkiewicz, M., Fadel, W., Harezlak, J., Crainiceanu, C.M., Urbanek, J.K. Adaptive empirical pattern transformation (ADEPT) with application to walking stride segmentation. Biostatistics, 2019. [(Article link)](https://academic.oup.com/biostatistics/advance-article/doi/10.1093/biostatistics/kxz033/5572661?guestAccessKey=f3abdf65-a94d-4509-b7e9-cca2ff384528)

##### Naming conventions used

- Objects are indexed starting from 1 (not: 0). 

- `vector` - one-dimensional array of values. Example:

    ```
    [3 4 5 -5 2 2]
    ```
    
    is a vector that has 6 elements.
    
    *  Element `3` is at index 1 (`3` is 1-st vector element). 
    *  Element `-5` is at index 4 (`-5` is 4-th vector element). 
    *  Elements `[-5 2 2]` are at indices `[SEQUENCE FROM 4 TO 6 BY 1]`.<br/><br/>
    

- `matrix` - a matrix, i.e. two-dimensional array of values. Example: 

    ```
         [,1] [,2] [,3] [,4]
    [1,]    0    2    1    1
    [2,]    4    3    3    2
    [3,]    4    7    2    4
    ```
    
    is a [3 x 4] dimensional matrix. 
    
    * Element `0` is at [1,1]-th matrix index. 
    * Element `7` is at [3,2]-th matrix index.
    * Elements `[2 1 1]` are at `[1, SEQUENCE FROM 2 TO 4 BY 1]`-th matrix indices. <br/><br/>
    
- `list` - a type of object that can be indexed and iterated over, and can contain objects of the same class (i.e. a list of vectors, a list of matrices). 

- `scalar` - a single value. Examples: `5` is a (numeric) scalar, `4.654` is a (numeric) scalar, `"hello"` is a (character) scalar.  

- Multiplying scalar and vector. By convention, `2 * [3 4 5]` yields vector `[6 8 10]`.

- `REDEFINE` could be thought of "update object", or "overwrite object with its modified version". 


## Pseudocode 

### Function `segment_pattern` (main function)

#### Arguments

- `x` - A numeric vector. A one-dimensional time-series we want to segment pattern from. In the application of walking stride segmentation, `x` would often be a vector magnitude computed from a three-dimensional time-series of raw acceleration measurements (for details how to compute vector magnitude, see [source](https://martakarass.github.io/resources/specialtopics_materials/intro/#spherical-coordinate-system-radius-vector-magnitude)). 

- `x_fs` - A numeric scalar. Frequency at which a time-series `x` is collected, expressed in a number of observations per second.

- `templates_list` - A list of numeric vectors. Each vector represents a distinct pattern template used in segmentation. Such templates may be sourced from publicly available ones (see R data package `adeptdata` [documentation](https://cran.r-project.org/web/packages/adeptdata/adeptdata.pdf), page 4) or manually derived from some small part of data collected from individuals from population of interest. 

- `pattern_duration_vec` - A numeric vector. A grid of potential pattern durations used in segmentation. Expressed in seconds. Example: for healthy individual with assumed stride duration range between 0.7s and 1.8s, this argument could be a vector `[0.7 0.8 0.9 1.0 1.1 1.2 1.3 1.4 1.5 1.6 1.7 1.8]`.

- `similarity_measure` - A character scalar. Statistic used to compute similarity between a time-series `x` and pattern templates. Considered values: `"cov"` - covariance, `"cor"` - correlation. Default is `"cov"`.

- `x_similarity_mat_moving_average_W` - A numeric scalar. A length of a moving window used in moving average smoothing of a time-series `x` for similarity matrix computation. Expressed in seconds. Default is NULL (no smoothing is applied).

- `do_finetune` - A logical scalar. Whether to apply fine-tuning procedure in segmentation. The fine-tuning procedure tunes preliminarily identified beginning and end of a pattern in data so as they correspond to local maxima of `x` (or of smoothed version of `x`, see `x_finetune_moving_average_W` arg) found within neighbourhoods of preliminary locations. Defaults to FALSE (no fine-tuning procedure employed). In the application of walking stride segmentation, we would often want to keep it TRUE as we want to learn precise location of a pattern occurrence in data.

- `x_finetune_moving_average_W` - A numeric scalar. A length of a moving window used in moving average smoothing of a time-series `x` in fine-tuning procedure. Expressed in seconds. Default is NULL (no smoothing applied). If `do_finetune` is set to FALSE, this parameter has no action. 

- `finetune_area_wing_W` - A numeric scalar. A length of wing of the area centered at preliminarily identified beginning and end of a pattern within which we search for local maxima of `x` (or smoothed version of `x`) in fine-tuning procedure. For example, if this parameter is set to `0.2` and `do_finetune` is set to TRUE, the algorithm will search for local maxima of `x` subset (or of smoothed version of `x` subset, see `x_finetune_moving_average_W` arg) which corresponds to from 0.2 seconds before to 0.2 seconds after the preliminarily identified beginning/end of a pattern, respectively. Default is NULL. If `do_finetune` is set to FALSE, this parameter has no action. Must be such `finetune_area_wing_W` is less than half od the smallest value in `pattern_duration_vec` (after mapping from time [s] to vector length [number of vector indices] that happens in algorithm). 

#### Returns

- `out_mat` - matrix with pattern segmentation results. Each row describes one identified pattern occurrence.
    -  column 1 (`tau_i`) -  numeric (integer) scalar; index of `x` where pattern starts.
    -  column 2 (`T_i`) - numeric (integer) scalar; pattern duration, expressed in `x` vector length,
    -  column 3 (`sim_i`) - numeric scalar; similarity between a pattern and `x` (as determined with `similarity_mat` matrix value in the algorithm; does not change after fine-tuning procedure application). 

#### Pseudocode 

```
function segment_pattern(x,
                         x_fs,
                         templates_list,
                         pattern_duration_vec,
                         similarity_measure = "cov",
                         x_similarity_mat_moving_average_W = NULL,
                         do_finetune = FALSE,
                         x_finetune_moving_average_W = NULL,
                         finetune_area_wing_W = NULL
                         ){

  DEFINE n = length of vector x 
  DEFINE k = number of elements in list templates_list
  DEFINE template_rescaled_vl_vec =  scalar x_fs * vector pattern_duration_vec
  REDEFINE template_rescaled_vl_vec = sorted ascending, unique, rounded to nearest integer values of vector template_rescaled_vl_vec 
  DEFINE m = length of vector template_rescaled_vl_vec 
  
  ## Define list of matrices with rescaled pattern templates
  DEFINE templates_rescaled_list = empty list of size m
  FOR i IN SEQUENCE FROM 1 TO m BY 1:
    DEFINE vl_i = i-th element of vector template_rescaled_vl_vec
    DEFINE templates_rescaled_i = empty matrix of [k x vl_i] dimension
    FOR j IN SEQUENCE FROM 1 TO k BY 1: 
      DEFINE template_j = j-th element of list templates_list
      DEFINE template_rescaled_ij = vector rescale_template(template_j, vl_i)
      DEFINE j-th row of matrix templates_rescaled_i = vector template_rescaled_ij
    END 
    DEFINE i-th element of templates_rescaled_list = matrix templates_rescaled_i 
  END 
  
  ## Define x for the purpose of similarity matrix computation  
  ## (smooth x if chosen so via function arguments)
  IF x_similarity_mat_moving_average_W IS NOT NULL:
    DEFINE W_vl =  scalar round(x_similarity_mat_moving_average_W * x_fs)
    DEFINE x_similarity_mat = vector running_mean(x, W_vl)
  ELSE:
    DEFINE x_similarity_mat = vector x
  END
  
  ## Compute similarity matrix between rescaled pattern templates and 
  ## subsequent windows  of x
  DEFINE similarity_mat = empty matrix of [m x n] dimension
  FOR i IN SEQUENCE FROM 1 TO m BY 1:
    DEFINE templates_rescaled_i = i-th element of list templates_rescaled_list
    DEFINE similarity_mat_i = empty matrix of [k x n] dimension
    FOR j IN SEQUENCE FROM 1 TO k BY 1: 
      DEFINE template_rescaled_ij = j-th row of matrix templates_rescaled_i
      DEFINE similarity_vec_ij = vector running_similarity(x_similarity_mat, template_rescaled_ij, similarity_measure)
      DEFINE j-th row of similarity_mat_i = vector similarity_vec_ij 
    END
    DEFINE i-th row of similarity_mat = vector being column-wise maximum of matrix similarity_mat_i  
  END
  
  ## Define fine-tuning procedure objects if fine-tuning procedure was chosen
  ## via function arguments
  IF do_finetune:
    IF x_finetune_moving_average_W IS NOT NULL:
      DEFINE W_vl = scalar round(x_finetune_moving_average_W * x_fs)
      DEFINE x_finetune = vector running_mean(x, W_vl)
    ELSE:
      DEFINE x_finetune = x
    END
    DEFINE x_already_fitted = vector of length n of all values set to FALSE
    DEFINE pattern_vl_min = scalar min(template_rescaled_vl_vec) 
    DEFINE pattern_vl_max = scalar max(template_rescaled_vl_vec)
    DEFINE finetune_area_wing_vl = scalar round(finetune_area_wing_W * x_fs)
    ASSERT finetune_area_wing_vl < scalar round(0.5 * min(template_rescaled_vl_vec))
  END
  
  ## Perform iterative procedure to identify occurrences of pattern in the data x
  DEFINE out_mat = empty matrix of [0 x 3] dimension
  WHILE TRUE: 
    IF all elements in similarity_mat are NULL: 
      BREAK LOOP
    END
    DEFINE tau_tmp, s_tmp = scalar row index, scalar column index of current maximum value of similarity_mat matrix
    DEFINE similarity_mat_maxval_tmp = scalar current maximum value of similarity_mat matrix
    IF do_finetune: 
      REDEFINE tau_tmp, s_tmp = scalar, scalar of finetune(tau_tmp, s_tmp, 
                                         x_finetune, x_already_fitted,
                                         pattern_vl_min, pattern_vl_max,
                                         finetune_area_wing_vl)
      REDEFINE x_already_fitted = vector update_x_already_fitted(x_already_fitted, tau_tmp, s_tmp)
    END
    REDEFINE similarity_mat = matrix update_similarity_mat(similarity_mat, tau_tmp, s_tmp, template_rescaled_vl_vec)
    DEFINE out_tmp = vector 3-element vector of values [tau_tmp, s_tmp, similarity_mat_maxval_tmp]
    REDEFINE out_mat = update out_mat matrix by appending out_tmp vector to it
  END
  
  REDEFINE out_mat = sort rows of matrix out_mat ascending by values in 1st column 
  CONDITIONED ON output object structure can have column names: 
    REDEFINE out_mat = assign column names of matrix out_mat to be ["tau_i", "T_i", "sim_i"] 
  END 
  
  RETURN out_mat
}
```

### Function `rescale_template`

#### Arguments

- `template_vector` - A numeric vector. 
- `vector_length` - A numeric (ingerer) scalar. Cector length that `template_vector` is to be linearly interpolated into.

#### Returns

- `out` - A numeric vector of length `vector_length`, mean 0 and variance 1. 

#### Pseudocode

```
function rescale_template(template_vector, vector_length){

  DEFINE out = vector being a result of linear interpolation applied to increase/decrease number of points in template_vector to vector_length number of points 
  REDEFINE out = result of standardizing vector_out vector so it has mean 0 and variance 1
  RETURN out
}
```

### Function `running_mean`

#### Arguments

- `x` - A numeric vector. 
- `W_vl` - A numeric (integer) scalar. A length of a moving window used in moving average smoothing of `x`. Expressed in vector length.

#### Returns

- `out` - A numeric vector. The length of `out` equals the length of `x` vector. `i`-th element of `out` corresponds to a sample mean of subset of `x` at positions `[i,...,i+W-1]`; the tail of the vector where sample mean is no longer defined is filled with NULL. Examples: 
    - `running_mean(x = [1  2  3  4  5  6  7  8  9 10], W_vl = 3)` should evaluate to `[2  3  4  5  6  7  8  9 NULL NULL]`
    - `running_mean(x = [20 19 18 17 16 15 14 13 12 11 10], W_vl = 5)` should evaluate to `[18 17 16 15 14 13 12 NULL NULL NULL NULL]`

#### Pseudocode

```
function running_mean(x, W_vl){

  ASSERT TRUE W_vl>1
  
  DEFINE out = vector of the same length as x, where i-th element corresponds to sample mean of subset of x at positions [i,...,i+W-1]; the tail of the vector where sample mean is no longer defined is filled with NULL
  RETURN out
}
```

### Function `running_similarity`

#### Arguments

- `x` - A numeric vector. 
- `y` - A numeric vector. Shorter than `x`. 
- `similarity_measure` - A character scalar. Statistic used to compute similarity between a time-series `x` and pattern templates. Considered values: `"cov"` - covariance, `"cor"` - correlation. 

#### Returns

- `out` - A numeric vector. The length of `out` equals the length of `x` vector. Say `y_vl` is length of vector `y`. Then `i`-th element of `out` corresponds to a sample similarity (`"cov"` - covariance, `"cor"` - correlation) of subset of `x` at positions `[i,...,i+y_vl-1]` and `y`; the tail of the vector where sample mean is no longer defined is filled with NULL. Examples: 
    - `running_similarity([1 -1  1  0  1 -1  1  0  1 -1  1  0], [1 -1  1], "cor")` should evaluate to `[1.0000000 -0.8660254  1.0000000 -0.8660254  1.0000000 -0.8660254  1.0000000 -0.8660254  1.0000000 -0.8660254 NULL NULL]`
    - `running_similarity([1 -1  1  0  1 -1  1  0  1 -1  1  0], [1 -1  1], "cov")` should evaluate to `[1.3333333 -1.0000000  0.6666667 -1.0000000  1.3333333 -1.0000000  0.6666667 -1.0000000  1.3333333 -1.0000000 NULL NULL]`
    
#### Pseudocode

```
function running_similarity(x, y, similarity_measure){
  
  DEFINE x_vl = length of vector x
  DEFINE y_vl = length of vector y
  ASSERT TRUE x_vl>y_vl
  
  DEFINE out = vector of the same length as x, where i-th element corresponds to similarity measure ("cov" = sample covariance, "cor" = sample correlation) of subset of x at positions [i,...,i+y_vl-1] and y
  RETURN out
}
```

### Function `finetune`

#### Arguments

- `tau_tmp` - A numeric (integer) scalar. 
- `s_tmp` - A numeric (integer) scalar. 
- `x_finetune` - A numeric vector.
- `x_already_fitted` - A logical (TRUE/FALSE) vector.
- `pattern_vl_min` - A numeric (integer) scalar. 
- `pattern_vl_max` - A numeric (integer) scalar. 
- `finetune_area_wing_vl` - A numeric (integer) scalar. 

#### Returns

- `tau_new` - A numeric (integer) scalar. 
- `s_new` - A numeric (integer) scalar. 

#### Pseudocode

```
function finetune(tau_tmp, 
                  s_tmp, 
                  x_finetune, 
                  x_already_fitted,
                  pattern_vl_min, 
                  pattern_vl_max,
                  finetune_area_wing_vl){

  DEFINE tau1_tmp = tau_tmp
  DEFINE tau2_tmp = tau_tmp + s_tmp - 1
  DEFINE x_already_fitted_vl = length of vector x_already_fitted
  
  ## Define search area indices of x_finetune signal in which we search for 
  ## maximum value around tau1_tmp point
  DEFINE tau1_area_idx_min = max(tau1_tmp - finetune_area_wing_vl, 1)
  DEFINE tau1_area_idx_max = tau1_tmp + finetune_area_wing_vl
  DEFINE tau1_area_idx = vector defined as SEQUENCE FROM tau1_area_idx_min TO tau1_area_idx_max BY 1
  REDEFINE tau1_area_idx = vector defined as subset of tau1_area_idx for which x_already_fitted[tau1_area_idx] is FALSE

  ## Define search area indices of x_finetune signal in which we search for 
  ## maximum value around tau2_tmp point
  DEFINE tau2_area_idx_min = tau2_tmp - finetune_area_wing_vl
  DEFINE tau2_area_idx_max = min(tau2_tmp + finetune_area_wing_vl, finetune_area_wing_vl)
  DEFINE tau2_area_idx = vector defined as SEQUENCE FROM tau2_area_idx_min TO tau2_area_idx_min BY 1
  REDEFINE tau2_area_idx = vector defined as subset of tau2_area_idx for which x_already_fitted[tau2_area_idx] is FALSE

  ASSERT TRUE max(tau1_area_idx) < tau2_tmp
  ASSERT TRUE min(tau2_area_idx) > tau1_tmp
  
  ## Compute matrix of distances between tau2 and tau1 indices
  ## and define if these are egligible given assumed template vector length range
  DEFINE tau1_area_idx_vl = length of tau1_area_idx vector
  DEFINE tau2_area_idx_vl = length of tau2_area_idx vector
  DEFINE tau12_mat = matrix of [tau1_area_idx_vl x tau2_area_idx_vl] dimension, where [i,j]-th matrix element (i-th row and j-th column matrix entry) equals tau2_area_idx[j]-tau1_area_idx[i]+1 (j-th element of vector tau2_area_idx minus i-th element of vector tau1_area_idx plus 1)
  DEFINE tau12_mat_isvalid = matrix of [tau1_area_idx_vl x tau2_area_idx_vl] dimension, where [i,j]-th matrix element (i-th row and j-th column matrix entry) equals TRUE if (tau12_mat[i,j] <= pattern_vl_max AND tau12_mat[i,j] >= pattern_vl_min), or equals FALSE otherwise
  
  ## Identify a pair of points in the two neighbourhods which corresponds to 
  ## maximum values of of `x_finetune` signal within egligible indices
  DEFINE x_finetune_at_tau1_area = vector defined as x_finetune[tau1_area_idx] (tau1_area_idx-th elements of vector x_finetune)
  DEFINE x_finetune_at_tau2_area = vector defined as x_finetune[tau2_area_idx] (tau2_area_idx-th elements of vector x_finetune)
  DEFINE x_finetune_at_taus_areas_mat =  matrix of [tau1_area_idx_vl x tau2_area_idx_vl] dimension, where [i,j]-th matrix element (i-th row and j-th column matrix entry) equals x_finetune_at_tau1_area[i]+x_finetune_at_tau2_area[j] (i-th element of vector x_finetune_at_tau1_area plus j-th element of vector x_finetune_at_tau2_area)
  REDEFINE x_finetune_at_taus_areas_mat = matrix of [tau1_area_idx_vl x tau2_area_idx_vl] dimension where [i,j]-th element remains x_finetune_at_taus_areas_mat[i,j] if [i,j]-th element of tau12_mat_isvalid is TRUE, otherwise it is set to NULL
  
  ## Identify "fine-tuned" start and end index point of identified pattern occurence
  DEFINE whichmaxrow, whichmaxcol = row index, column index of maximum value of x_finetune_at_taus_areas_mat matrix
  DEFINE tau1_tmp = vector defined as tau1_area_idx[whichmaxrow] (whichmaxrow-th element of vector tau1_area_idx)
  DEFINE tau2_tmp = vector defined as tau2_area_idx[whichmaxcol] (whichmaxcol-th element of vector tau2_area_idx)
  DEFINE tau_new = tau1_tmp
  DEFINE s_new = tau2_tmp - tau1_tmp + 1
  
  RETURN tau_new, s_new
}
```

### Function `update_x_already_fitted`

#### Arguments

-  `x_already_fitted` - A logical (TRUE/FALSE) vector. 
- `tau_tmp` - A numeric (integer) scalar. 
- `s_tmp` - A numeric (integer) scalar. 

#### Returns

-  `x_already_fitted` - A logical (TRUE/FALSE) vector. 

#### Pseudocode

```
function update_x_already_fitted(x_already_fitted, tau_tmp, s_tmp){

  REDFINE replace_idx = vector defined as SEQUENCE FROM (tau_tmp + 1) TO (tau_tmp + s_tmp - 2) BY 1
  REDEFINE x_already_fitted = set x_already_fitted[replace_idx] to TRUE
  RETURN x_already_fitted
}
```

### Function `update_similarity_mat`

#### Arguments

- `similarity_mat` - A numeric matrix. 
- `tau_tmp` - A numeric (integer) scalar. 
- `s_tmp` - A numeric (integer) scalar. 
- `template_rescaled_vl_vec` - A numeric (integer) vector. 

#### Returns

-  `similarity_mat` - A numeric matrix. 

#### Pseudocode

```
function update_similarity_mat(similarity_mat, 
                               tau_tmp, 
                               s_tmp, 
                               template_rescaled_vl_vec){
                               
  DEFINE x_vl = number of columns in similarity_mat matrix
  DEFINE m = length of vector template_rescaled_vl_vec
  FOR i IN SEQUENCE FROM 1 TO m BY 1: 
    DEFINE s_i = i-th element of template_rescaled_vl_vec
    DEFINE null_repl_cols_min = tau_tmp - s_i + 2
    DEFINE null_repl_cols_max = tau_tmp + s_i - 2
    REDEFINE null_repl_cols_min = min(max(1, null_repl_cols_min), x_vl)
    REDEFINE null_repl_cols_max = min(max(1, null_repl_cols_max), x_vl)
    DEFINE null_repl_cols = SEQUENCE FROM null_repl_cols_min TO null_repl_cols_max BY 1
    REDEFINE similarity_mat = update similarity_mat so as elements at [i, null_repl_cols] are replaced with NULL
  END
  RETURN similarity_mat
}
```







