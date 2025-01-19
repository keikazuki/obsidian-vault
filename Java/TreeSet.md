### Java `TreeSet`: A Comprehensive Guide

**1. What is a TreeSet?**

- **Definition**: A `TreeSet` is a part of Java’s `java.util` package that implements the `NavigableSet` interface and is based on a **TreeMap**.
- **Key Features**:
    - Stores **unique elements** in **sorted order**.
    - By default, elements are sorted in their **natural order** (e.g., ascending for numbers, lexicographical for strings).
    - Can use a **custom comparator** for ordering.
    - Does **not allow `null` elements**.
    - Not synchronized (use `Collections.synchronizedSet` for thread safety).

---

**2. Creating a TreeSet**

**a. Default Constructor**

```java
TreeSet<Type> set = new TreeSet<>();
```

**b. Using Custom Comparator**

```java
TreeSet<Type> set = new TreeSet<>(Comparator.reverseOrder());
```

**c. From Another Collection**

```java
TreeSet<Type> set = new TreeSet<>(existingCollection);
```

---

**3. Common Operations**

|Method|Description|
|---|---|
|`add(element)`|Adds the element to the set if it is not already present.|
|`remove(element)`|Removes the specified element from the set.|
|`contains(element)`|Checks if the set contains the specified element.|
|`isEmpty()`|Checks if the set is empty.|
|`size()`|Returns the number of elements in the set.|
|`clear()`|Removes all elements from the set.|
|`iterator()`|Returns an iterator over the elements in the set.|
|`first()`|Returns the smallest element in the set.|
|`last()`|Returns the largest element in the set.|
|`floor(element)`|Returns the greatest element ≤ specified element.|
|`ceiling(element)`|Returns the smallest element ≥ specified element.|
|`subSet(from, to)`|Returns a view of elements in the range `[from, to)`.|

---

**4. Adding and Removing Elements**

**a. Adding Elements**

```java
TreeSet<String> set = new TreeSet<>();
set.add("Alice");
set.add("Bob");
set.add("Charlie");
```

**b. Removing Elements**

```java
set.remove("Bob");
```

---

**5. Iterating Over a TreeSet**

**a. Using For-Each Loop**

```java
for (String name : set) {
    System.out.println(name);
}
```

**b. Using Iterator**

```java
Iterator<String> iterator = set.iterator();
while (iterator.hasNext()) {
    System.out.println(iterator.next());
}
```

---

**6. Key Characteristics**

**a. Sorted Order**

- Elements are stored in ascending order (by default).

```java
TreeSet<Integer> set = new TreeSet<>();
set.add(3);
set.add(1);
set.add(2);
System.out.println(set); // Output: [1, 2, 3]
```

**b. Custom Ordering**

- Use a custom comparator for descending or other orders:

```java
TreeSet<Integer> set = new TreeSet<>(Comparator.reverseOrder());
set.add(3);
set.add(1);
set.add(2);
System.out.println(set); // Output: [3, 2, 1]
```

**c. No `null` Elements**

- `TreeSet` does not allow `null` elements.

```java
TreeSet<String> set = new TreeSet<>();
set.add(null); // Throws NullPointerException
```

**d. Unique Elements**

- Duplicate elements are not allowed.

```java
set.add(1);
set.add(1); // Ignored, no duplicates allowed
```

---

**7. Navigational Methods**

- **Retrieve Boundaries**:

```java
TreeSet<Integer> set = new TreeSet<>();
set.add(10);
set.add(20);
set.add(30);

System.out.println(set.first()); // Output: 10
System.out.println(set.last());  // Output: 30
```

- **Find Closest Matches**:

```java
System.out.println(set.floor(25));  // Output: 20 (greatest ≤ 25)
System.out.println(set.ceiling(25)); // Output: 30 (smallest ≥ 25)
```

---

**8. Subsets and Range Views**

- **SubSet**: Returns a view of the elements in the specified range `[from, to)`.

```java
TreeSet<Integer> set = new TreeSet<>();
set.add(10);
set.add(20);
set.add(30);

System.out.println(set.subSet(10, 30)); // Output: [10, 20]
```

- **HeadSet**: Returns a view of elements less than the specified element.

```java
System.out.println(set.headSet(20)); // Output: [10]
```

- **TailSet**: Returns a view of elements greater than or equal to the specified element.

```java
System.out.println(set.tailSet(20)); // Output: [20, 30]
```

---

**9. Internal Working**

- **TreeSet** is backed by a **TreeMap**.
- Elements are stored in a balanced **Red-Black Tree**, ensuring sorted order and efficient operations.

---

**10. Performance**

- **Time Complexity**:
    - `add()`, `remove()`, `contains()`: **O(log n)**.
    - Iteration: **O(n)**.
- **Space Complexity**:
    - Depends on the number of elements and the tree structure.

---

**11. Thread-Safe TreeSet**

`TreeSet` is not synchronized. For concurrent access:

- Use `Collections.synchronizedSet`:

```java
Set<String> syncSet = Collections.synchronizedSet(new TreeSet<>());
```

---

**12. Differences Between TreeSet and Other Sets**

|Feature|TreeSet|HashSet|LinkedHashSet|
|---|---|---|---|
|Ordering|Sorted Order|No order|Insertion order|
|Null Values|Does not allow `null`|Allows one `null`|Allows one `null`|
|Performance|Slower (O(log n))|Faster (O(1))|Slightly slower|
|Use Case|Sorted data access|General-purpose set|Predictable iteration|

---

**13. Common Use Cases**

1. **Sorted Data Storage**:
    
    - Use `TreeSet` to store and retrieve elements in sorted order efficiently.
2. **Range Queries**:
    
    - Easily extract subsets of elements within a specific range.
3. **Avoiding Duplicates with Sorting**:
    
    - Ensure unique elements while maintaining order.

---

**14. Common Interview Questions**

1. **How does TreeSet maintain sorted order?**
    
    - `TreeSet` uses a **Red-Black Tree** to store elements in sorted order.
2. **Can TreeSet store `null` elements?**
    
    - No, inserting `null` into a `TreeSet` throws a `NullPointerException`.
3. **What happens when duplicate elements are added?**
    
    - Duplicate elements are ignored because `TreeSet` ensures unique elements.
4. **How is TreeSet different from TreeMap?**
    
    - `TreeSet` is a set and stores only values, while `TreeMap` is a map and stores key-value pairs.
5. **What is the time complexity of TreeSet operations?**
    
    - Basic operations like `add()`, `remove()`, and `contains()` have a time complexity of **O(log n)**.

---