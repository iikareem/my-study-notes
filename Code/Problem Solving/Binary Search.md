---
tags:
  - algorithms
  - binary-search
  - code
  - problem-solving
---

# Binary Search

Binary search is an efficient algorithm for finding a target value in a sorted array (or list). It works by repeatedly dividing the search space in half, leveraging the sorted nature of the data to eliminate half of the remaining elements with each step.

Binary search relies on this monotonic property to efficiently find a target value in a sorted array. Because the sequence is monotonic (sorted), binary search can:

- **Prerequisites**: The array must be sorted (ascending or descending).
- - Initialize two pointers: low (start of array) and high (end of array).
- Compute the middle index: mid = (low + high) // 2. or left  + (right - left) / 2
- Compare the middle element with the target:
    - If array[mid] == target, you’ve found the target.
    - If array[mid] < target, search the right half (low = mid + 1).
    - If array[mid] > target, search the left half (high = mid - 1).
- Repeat until low > high (target not found) or the target is found.

### **Lower Bound**

The **lower bound** of a value x in a sorted array is the **index of the first element** that is **greater than or equal to** x (i.e., arr[i] >= x). If no such element exists, it returns the length of the array (n).

#### **Why Use Lower Bound?**

- Useful for finding the insertion point of x in a sorted array to maintain sorted order.
- Helps in problems like finding the first occurrence of a value or counting elements less than x.

#### **Algorithm**

- Use a binary search variant:
    - If arr[mid] >= x, the current mid is a candidate, but check the left half for a smaller index (high = mid - 1).
    - If arr[mid] < x, search the right half (low = mid + 1).
    - The answer is the low pointer when the loop ends.

### **Upper Bound**

The **upper bound** of a value x in a sorted array is the **index of the first element** that is **strictly greater than** x (i.e., arr[i] > x). If no such element exists, it returns the length of the array (n).

#### **Why Use Upper Bound?**

- Useful for finding the insertion point for x in a sorted array if duplicates are allowed.
- Helps in problems like finding the number of elements less than or equal to x.

#### **Algorithm**

- Use a binary search variant:
    - If arr[mid] > x, the current mid is a candidate, but check the left half for a smaller index (high = mid - 1).
    - If arr[mid] <= x, search the right half (low = mid + 1).
    - The answer is the low pointer when the loop ends.

![[Pasted image 20250527225034.png]]

### **Common Pitfalls**

1. **Unsorted Array**: Binary search, lower bound, and upper bound require a sorted array. Always verify or sort first.
2. **Integer Overflow**: When calculating mid, use mid = low + (high - low) // 2 to avoid overflow in languages like C++ for large arrays.
3. **Edge Cases**:
    - Empty array: Return -1 for binary search, 0 for lower/upper bound.
    - x not in array: Binary search returns -1; lower/upper bound return appropriate indices.

- **Binary Search**: When you need to find an exact element or check its existence.
- **Lower Bound**: When you need the first position where x could be inserted or the first occurrence of >= x.
- **Upper Bound**: When you need the first position where an element greater than x exists or to count elements <= x.

## See also

- [[Two Pointers]]
- [[Code MOC]]
