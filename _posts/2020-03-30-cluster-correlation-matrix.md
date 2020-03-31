---
layout: post
title: Cluster a Correlation Matrix (in python)
---

Below is a function to rearrange variables in a correlation matrix (either pandas.DataFrame or numpy.ndarray) to group highly correlated variables near each other.

It turns a correlation matrix that looks like:

![Correlation matrix before grouping]({{site.base_url}}/images/cluster_corr_matrix/ungrouped.png)

Into one that looks like:

![Correlation matrix after grouping]({{site.base_url}}/images/cluster_corr_matrix/grouped.png)

```Python
import scipy
import scipy.cluster.hierarchy as sch

def cluster_corr(corr_array, inplace=False):
    """
    Rearranges the correlation matrix, corr_array, so that groups of highly 
    correlated variables are next to eachother 
    
    Parameters
    ----------
    corr_array : pandas.DataFrame or numpy.ndarray
        a NxN correlation matrix 
        
    Returns
    -------
    pandas.DataFrame or numpy.ndarray
        a NxN correlation matrix with the columns and rows rearranged
    """
    pairwise_distances = sch.distance.pdist(corr_array)
    linkage = sch.linkage(pairwise_distances, method='complete')
    cluster_distance_threshold = pairwise_distances.max()/2
    idx_to_cluster_array = sch.fcluster(linkage, cluster_distance_threshold, 
                                        criterion='distance')
    idx = np.argsort(idx_to_cluster_array)
    
    if not inplace:
        corr_array = corr_array.copy()
    
    if isinstance(corr_array, pd.DataFrame):
        return corr_array.iloc[idx, :].T.iloc[idx, :]
    return corr_array[idx, :][:, idx]
```

You will typically use this function right before calling `seaborn.heatmap`, as in

```Python
#sns.heatmap(df.corr()) # unclustered version
sns.heatmap(cluster_corr(df.corr()))
```
