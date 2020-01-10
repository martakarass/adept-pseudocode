* [Pseudocode intro notes](#pseudocode-intro-notes)
     * [Paper reference](#paper-reference)
     * [Naming conventions used](#naming-conventions-used)
* [Pseudocode](#pseudocode)
   * [Function `segment_pattern` (main function)](#function-segment_pattern-main-function)
      * [Arguments](#arguments)
      * [Returns](#returns)
      * [Pseudocode](#pseudocode-1)
   * [Function `rescale_template`](#function-rescale_template)
      * [Arguments](#arguments-1)
      * [Returns](#returns-1)
      * [Pseudocode](#pseudocode-2)
   * [Function `running_mean`](#function-running_mean)
      * [Arguments](#arguments-2)
      * [Returns](#returns-2)
      * [Pseudocode](#pseudocode-3)
   * [Function `running_similarity`](#function-running_similarity)
      * [Arguments](#arguments-3)
      * [Returns](#returns-3)
      * [Pseudocode](#pseudocode-4)
   * [Function `finetune`](#function-finetune)
      * [Arguments](#arguments-4)
      * [Returns](#returns-4)
      * [Pseudocode](#pseudocode-5)
   * [Function `update_x_already_fitted`](#function-update_x_already_fitted)
      * [Arguments](#arguments-5)
      * [Returns](#returns-5)
      * [Pseudocode](#pseudocode-6)
   * [Function `update_similarity_mat`](#function-update_similarity_mat)
      * [Arguments](#arguments-6)
      * [Returns](#returns-6)
      * [Pseudocode](#pseudocode-7)

## Pseudocode intro notes

### Paper reference

- Karas, M., Straczkiewicz, M., Fadel, W., Harezlak, J., Crainiceanu, C.M., Urbanek, J.K. Adaptive empirical pattern transformation (ADEPT) with application to walking stride segmentation. Biostatistics, 2019. [(Article link)](https://academic.oup.com/biostatistics/advance-article/doi/10.1093/biostatistics/kxz033/5572661?guestAccessKey=f3abdf65-a94d-4509-b7e9-cca2ff384528)

### Naming conventions used

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

- `scalar` - a single value. Examples: `5` is a numeric (integer) scalar, `4.654` is a numeric scalar, `"hello"` is a character scalar, `TRUE` and `FALSE` are logical scalars.  

- Multiplying scalar and vector. By convention, `2 * [3 4 5]` yields vector `[6 8 10]`.

- `REDEFINE` could be thought of "update object", or "overwrite object with its modified version". 

- `SEQUENCE` is used to define vector of equally spaced numeric scalars. Example: `[SEQUENCE FROM 5 TO 10 BY 1]` yields `[5  6  7  8  9 10]`.

- `x[1 2 3]` is used to define subseting vector `x` to keep only elements at indices `[1 2 3]` of vector `x`. Examples: 
    
    ```
    x = [11 12 13 14 15 16 17 18 19 20]
    x[2]
    ```
    yields `12`. 

    ```
    x = [11 12 13 14 15 16 17 18 19 20]
    x[2 3 4]
    ```
    yields `[12 13 14]`. 
    
    ```
    x[SEQUENCE FROM 1 TO 10 BY 3]
    ```
    yields `[11 14 17 20]`. 


## Pseudocode 

### Function `segment_pattern` (main function)

#### Arguments

- `x` - A numeric vector. A one-dimensional time-series we want to segment a pattern from. In the application of walking strides segmentation, `x` would often be a vector magnitude computed from a three-dimensional time-series of raw acceleration measurements (for details how to compute vector magnitude, see [source](https://martakarass.github.io/resources/specialtopics_materials/intro/#spherical-coordinate-system-radius-vector-magnitude)). 

- `x_fs` - A numeric scalar. Frequency at which data vector `x` is collected, expressed in a number of observations per second.

- `templates_list` - A list of numeric vectors. Each vector represents a distinct pattern template used in segmentation. Such templates may be sourced from publicly available ones (see R data package `adeptdata` [documentation](https://cran.r-project.org/web/packages/adeptdata/adeptdata.pdf), page 4) or manually derived from some small part of data collected from individuals from population of interest (see example in  [source](https://cran.r-project.org/web/packages/adept/vignettes/adept-strides-segmentation.html#segmentation-with-approach-2-derive-stride-templates-semi-manually)). 

- `pattern_duration_vec` - A numeric vector. A grid of potential pattern durations used in segmentation. Expressed in seconds. Example: for healthy individual with assumed stride duration range between 0.7s and 1.8s, this argument could be a vector `[0.7 0.8 0.9 1.0 1.1 1.2 1.3 1.4 1.5 1.6 1.7 1.8]`.

- `similarity_measure` - A character scalar. Statistic used to compute similarity matrix between `x` and pattern templates. Considered values: `"cov"` - covariance, `"cor"` - correlation. Default is `"cov"`.

- `x_similarity_mat_moving_average_W` - A numeric scalar. A length of a moving window used in moving average smoothing of `x` for similarity matrix computation. Expressed in seconds. Default is NULL (no smoothing is applied).

- `do_finetune` - A logical scalar. Whether to apply fine-tuning procedure in segmentation. The fine-tuning procedure tunes preliminarily identified beginning and end of a pattern in data so as they correspond to local maxima of `x` (or of smoothed version of `x`, see `x_finetune_moving_average_W` arg) found within neighbourhoods of preliminary locations. Defaults to FALSE (no fine-tuning procedure employed). In the application of walking stride segmentation, we would often want to keep it TRUE as we want to learn precise locations of a pattern occurrence in data.

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
  FOR i IN [SEQUENCE FROM 1 TO m BY 1]:
    DEFINE vl_i = i-th element of vector template_rescaled_vl_vec
    DEFINE templates_rescaled_i = empty matrix of [k x vl_i] dimension
    FOR j IN [SEQUENCE FROM 1 TO k BY 1]: 
      DEFINE template_j =  vector defined as j-th element of list templates_list
      DEFINE template_rescaled_ij = vector defined as rescale_template(template_j, vl_i)
      DEFINE j-th row of matrix templates_rescaled_i = vector template_rescaled_ij
    END 
    DEFINE i-th element of templates_rescaled_list = matrix templates_rescaled_i 
  END 
  
  ## Define x for the purpose of similarity matrix computation  
  ## (smooth x if chosen so via function arguments)
  IF x_similarity_mat_moving_average_W IS NOT NULL:
    DEFINE W_vl =  scalar round(x_similarity_mat_moving_average_W * x_fs)
    DEFINE x_similarity_mat = vector defined as running_mean(x, W_vl)
  ELSE:
    DEFINE x_similarity_mat = vector x
  END
  
  ## Compute similarity matrix between rescaled pattern templates and 
  ## subsequent windows  of x
  DEFINE similarity_mat = empty matrix of [m x n] dimension
  FOR i IN [SEQUENCE FROM 1 TO m BY 1]:
    DEFINE templates_rescaled_i = vector defined as i-th element of list templates_rescaled_list
    DEFINE similarity_mat_i = empty matrix of [k x n] dimension
    FOR j IN [SEQUENCE FROM 1 TO k BY 1]: 
      DEFINE template_rescaled_ij = vector defined as j-th row of matrix templates_rescaled_i
      DEFINE similarity_vec_ij = vector defined as running_similarity(x_similarity_mat, template_rescaled_ij, similarity_measure)
      DEFINE j-th row of similarity_mat_i = vector similarity_vec_ij 
    END
    DEFINE i-th row of similarity_mat = vector being column-wise maximum of matrix similarity_mat_i  
  END
  
  ## Define fine-tuning procedure objects if fine-tuning procedure was chosen
  ## via function arguments
  IF do_finetune:
    IF x_finetune_moving_average_W IS NOT NULL:
      DEFINE W_vl = scalar round(x_finetune_moving_average_W * x_fs)
      DEFINE x_finetune = vector defined as running_mean(x, W_vl)
    ELSE:
      DEFINE x_finetune = x
    END
    DEFINE x_already_fitted = vector of length n of all values equal FALSE
    DEFINE pattern_vl_min = scalar min(template_rescaled_vl_vec) 
    DEFINE pattern_vl_max = scalar max(template_rescaled_vl_vec)
    DEFINE finetune_area_wing_vl = scalar round(finetune_area_wing_W * x_fs)
    ASSERT finetune_area_wing_vl < scalar round(0.5 * min(template_rescaled_vl_vec))
  END
  
  ## Perform iterative procedure to identify occurrences of pattern in data vector x
  DEFINE out_mat = empty matrix of [0 x 3] dimension
  WHILE TRUE: 
    IF all elements in similarity_mat are NULL: 
      BREAK LOOP
    END
    DEFINE tau_tmp, s_tmp = scalar and scalar defined as row index and column index of current maximum value of similarity_mat matrix
    DEFINE similarity_mat_maxval_tmp = current maximum value of similarity_mat matrix
    IF do_finetune: 
      REDEFINE tau_tmp, s_tmp = scalar and scalar defined as result of finetune(tau_tmp, s_tmp, x_finetune, x_already_fitted, pattern_vl_min, pattern_vl_max, finetune_area_wing_vl)
      REDEFINE x_already_fitted = vector defined as update_x_already_fitted(x_already_fitted, tau_tmp, s_tmp)
    END
    REDEFINE similarity_mat = matrix defined as update_similarity_mat(similarity_mat, tau_tmp, s_tmp, template_rescaled_vl_vec)
    DEFINE out_tmp =  3-element vector of values [tau_tmp, s_tmp, similarity_mat_maxval_tmp]
    REDEFINE out_mat = update matrix out_mat  by appending out_tmp vector to it
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
- `out_vl` - A numeric (integer) scalar. Vector length that `template_vector` is to be linearly interpolated into.

#### Returns

- `out` - A numeric vector obtained via linear interpolation of `template_vector` to have vector length of `out_vl`, standardized to have mean 0 and variance 1. 

#### Pseudocode

```
function rescale_template(template_vector, out_vl){

  DEFINE out = vector being a result of linear interpolation applied to increase/decrease number of points in vector template_vector to number of points defined by out_vl
  REDEFINE out = vector obtained via standardizing vector out so it has mean 0 and variance 1
  RETURN out
}
```

### Function `running_mean`

#### Arguments

- `x` - A numeric vector. 
- `W_vl` - A numeric (integer) scalar. A length of a moving window used in moving average smoothing of `x`. Expressed in vector length.

#### Returns

- `out` - A numeric vector. The length of `out` equals the length of vector `x`. An `i`-th element of `out` corresponds to a sample mean of subset of `x` at positions defined as `[SEQUENCE FROM i TO (i+W_vl-1) BY 1]`; the tail of the vector where sample mean is no longer defined is filled with NULL. Examples: 
    - `running_mean(x = [1  2  3  4  5  6  7  8  9 10], W_vl = 3)` should evaluate to `[2  3  4  5  6  7  8  9 NULL NULL]`
    - `running_mean(x = [20 19 18 17 16 15 14 13 12 11 10], W_vl = 5)` should evaluate to `[18 17 16 15 14 13 12 NULL NULL NULL NULL]`

#### Pseudocode

```
function running_mean(x, W_vl){

  ASSERT TRUE W_vl>1
  
  DEFINE out = vector of the same length as x, where i-th element corresponds to sample mean of subset of x at positions [SEQUENCE FROM i TO (i+W_vl-1) BY 1]; the tail of the vector out where sample mean is no longer defined is filled with NULL
  RETURN out
}
```

### Function `running_similarity`

#### Arguments

- `x` - A numeric vector. 
- `y` - A numeric vector. Shorter than `x`. 
- `similarity_measure` - A character scalar. Statistic used to compute similarity between a time-series `x` and pattern templates. Considered values: `"cov"` - covariance, `"cor"` - correlation. 

#### Returns

- `out` - A numeric vector. The length of `out` equals the length of `x` vector. Say `y_vl` is length of vector `y`. Then `i`-th element of `out` corresponds to a sample similarity (`"cov"` - covariance, `"cor"` - correlation) of subset of `x` at positions defined as `[SEQUENCE FROM i TO (i+y_vl-1) BY 1]` and `y`; the tail of the vector where sample mean is no longer defined is filled with NULL. Examples: 
    - `running_similarity([1 -1  1  0  1 -1  1  0  1 -1  1  0], [1 -1  1], "cor")` should evaluate to `[1.0000000 -0.8660254  1.0000000 -0.8660254  1.0000000 -0.8660254  1.0000000 -0.8660254  1.0000000 -0.8660254 NULL NULL]`
    - `running_similarity([1 -1  1  0  1 -1  1  0  1 -1  1  0], [1 -1  1], "cov")` should evaluate to `[1.3333333 -1.0000000  0.6666667 -1.0000000  1.3333333 -1.0000000  0.6666667 -1.0000000  1.3333333 -1.0000000 NULL NULL]`
    
#### Pseudocode

```
function running_similarity(x, y, similarity_measure){
  
  DEFINE x_vl = length of vector x
  DEFINE y_vl = length of vector y
  ASSERT TRUE x_vl>y_vl
  
  DEFINE out = vector of the same length as x, where i-th element corresponds to similarity measure ("cov" = sample covariance, "cor" = sample correlation) of subset of x at positions [SEQUENCE FROM i TO (i+y_vl-1) BY 1] and y; the tail of the vector out where similarity statistic is no longer defined is filled with NULL
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
  DEFINE tau1_area_idx = vector defined as [SEQUENCE FROM tau1_area_idx_min TO tau1_area_idx_max BY 1]
  REDEFINE tau1_area_idx = vector defined as subset of vector tau1_area_idx for which x_already_fitted[tau1_area_idx] is FALSE

  ## Define search area indices of x_finetune signal in which we search for 
  ## maximum value around tau2_tmp point
  DEFINE tau2_area_idx_min = tau2_tmp - finetune_area_wing_vl
  DEFINE tau2_area_idx_max = min(tau2_tmp + finetune_area_wing_vl, finetune_area_wing_vl)
  DEFINE tau2_area_idx = vector defined as [SEQUENCE FROM tau2_area_idx_min TO tau2_area_idx_min BY 1]
  REDEFINE tau2_area_idx = vector defined as subset of tau2_area_idx for which x_already_fitted[tau2_area_idx] is FALSE

  ASSERT TRUE max(tau1_area_idx) < tau2_tmp
  ASSERT TRUE min(tau2_area_idx) > tau1_tmp
  
  ## Compute matrix of distances between tau2 and tau1 indices
  ## and define if these are egligible given assumed template vector length range
  DEFINE tau1_area_idx_vl = length of tau1_area_idx vector
  DEFINE tau2_area_idx_vl = length of tau2_area_idx vector
  DEFINE tau12_mat = matrix of [tau1_area_idx_vl x tau2_area_idx_vl] dimension, where [i,j]-th matrix element is defined as (tau2_area_idx[j]-tau1_area_idx[i]+1) 
  DEFINE tau12_mat_isvalid = matrix of [tau1_area_idx_vl x tau2_area_idx_vl] dimension, where [i,j]-th matrix element is defined as TRUE if (tau12_mat[i,j] <= pattern_vl_max AND tau12_mat[i,j] >= pattern_vl_min), or defined as FALSE otherwise
  
  ## Identify a pair of points in the two neighbourhods which corresponds to 
  ## maximum values of of `x_finetune` signal within egligible indices
  DEFINE x_finetune_at_tau1_area = vector defined as x_finetune[tau1_area_idx]
  DEFINE x_finetune_at_tau2_area = vector defined as x_finetune[tau2_area_idx] 
  DEFINE x_finetune_at_taus_areas_mat =  matrix of [tau1_area_idx_vl x tau2_area_idx_vl] dimension, where [i,j]-th matrix element is defined as (x_finetune_at_tau1_area[i]+x_finetune_at_tau2_area[j])
  REDEFINE x_finetune_at_taus_areas_mat = matrix of [tau1_area_idx_vl x tau2_area_idx_vl] dimension where [i,j]-th element is defined as x_finetune_at_taus_areas_mat[i,j] if [i,j]-th element of tau12_mat_isvalid is TRUE, otherwise it is defined as NULL
  
  ## Identify "fine-tuned" start and end index point of identified pattern occurence
  DEFINE whichmaxrow, whichmaxcol = scalar and scalar row index, column index of maximum value of x_finetune_at_taus_areas_mat matrix (or first pair of those values, if multiple pairs fulfill the condition)
  DEFINE tau1_tmp = tau1_area_idx[whichmaxrow]
  DEFINE tau2_tmp = tau2_area_idx[whichmaxcol] 
  DEFINE tau_new = scalar tau1_tmp
  DEFINE s_new = scalar (tau2_tmp - tau1_tmp + 1)
  
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

  REDFINE replace_idx = vector defined as [SEQUENCE FROM (tau_tmp + 1) TO (tau_tmp + s_tmp - 2) BY 1]
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

-  `similarity_mat` - A numeric matrix. Similarity matrix updated in a way that its entries corresponding to a patter occurrence that would overlap with a newly identified pattern occurrence (defined with `tau_tmp` and `s_tmp` args) are replaced with NULL. 

#### Pseudocode

```
function update_similarity_mat(similarity_mat, 
                               tau_tmp, 
                               s_tmp, 
                               template_rescaled_vl_vec){
                               
  DEFINE x_vl = number of columns in similarity_mat matrix
  DEFINE m = length of vector template_rescaled_vl_vec
  FOR i IN [SEQUENCE FROM 1 TO m BY 1]: 
    DEFINE s_i = i-th element of template_rescaled_vl_vec
    DEFINE null_repl_cols_min = tau_tmp - s_i + 2
    DEFINE null_repl_cols_max = tau_tmp + s_i - 2
    REDEFINE null_repl_cols_min = min(max(1, null_repl_cols_min), x_vl)
    REDEFINE null_repl_cols_max = min(max(1, null_repl_cols_max), x_vl)
    DEFINE null_repl_cols = vector defined as [SEQUENCE FROM null_repl_cols_min TO null_repl_cols_max BY 1]
    REDEFINE similarity_mat = update similarity_mat so as elements at [i, null_repl_cols] are replaced with NULL
  END
  RETURN similarity_mat
}
```







