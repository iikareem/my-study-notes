---
tags:
  - algorithms
  - code
  - difference-array
  - prefix-sum
  - problem-solving
---

# Prefix Sum and Difference Array

A prefix sum array is an array where each element at index **i** represents the sum of all elements from the start of the original array up to index **i**. It's useful for efficiently calculating the sum of elements in a range of an array in O(1) time after O(n) preprocessing.

#### Definition

For an array arr of length n, the prefix sum array prefix_sum is defined as:

- prefix_sum[0] = arr[0]
- prefix_sum[i] = prefix_sum[i-1] + arr[i] for i from 1 to n-1

#### Example

Given arr = [1, 2, 3, 4]:

- prefix_sum[0] = 1
- prefix_sum[1] = 1 + 2 = 3
- prefix_sum[2] = 3 + 3 = 6
- prefix_sum[3] = 6 + 4 = 10

So, prefix_sum = [1, 3, 6, 10].

#### Use Case

To find the sum of elements from index l to r in arr:

- Sum = prefix_sum[r] - prefix_sum[l-1] (if l > 0), or prefix_sum[r] (if l = 0).
- Example: Sum from index 1 to 2 = prefix_sum[2] - prefix_sum[0] = 6 - 1 = 5 (i.e., 2 + 3 = 5).

### Difference Array

A **difference array** is used to represent an array in terms of the differences between consecutive elements. It’s particularly useful for handling range update queries efficiently, such as adding a value to all elements in a given range.

#### Definition

For an array arr of length n, the difference array diff is defined as:

- diff[0] = arr[0]
- diff[i] = arr[i] - arr[i-1] for i from 1 to n-1

#### Reconstructing the Original Array

The original array can be reconstructed from the difference array:

- arr[0] = diff[0]
- arr[i] = diff[i] + arr[i-1] for i from 1 to n-1

#### Example

Given arr = [1, 3, 6, 10]:

- diff[0] = arr[0] = 1
- diff[1] = arr[1] - arr[0] = 3 - 1 = 2
- diff[2] = arr[2] - arr[1] = 6 - 3 = 3
- diff[3] = arr[3] - arr[2] = 10 - 6 = 4

So, diff = [1, 2, 3, 4].

Reconstructing back:

- arr[0] = diff[0] = 1
- arr[1] = diff[1] + arr[0] = 2 + 1 = 3
- arr[2] = diff[2] + arr[1] = 3 + 3 = 6
- arr[3] = diff[3] + arr[2] = 4 + 6 = 10

#### Use Case

For range updates (e.g., add k to elements from index l to r):

1. Update diff[l] += k (start of range).
2. Update diff[r+1] -= k (end of range, if r+1 < n).
3. Reconstruct the array by computing the prefix sum of diff.

#### Applications

- Efficiently handle range updates (e.g., adding a value to a range of elements).
- Problems involving frequent updates and queries on arrays.

## See also

- [[Two Pointers]]
- [[Code MOC]]
