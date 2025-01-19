### Java `HashSet`: A Comprehensive Guide

**1. What is a HashSet?**

- **Definition**: A `HashSet` is a part of Java’s `java.util` package that implements the `Set` interface, backed by a `HashMap`. It stores unique elements and does not allow duplicates.
- **Key Features**:
    - Does not allow duplicate elements.
    - Allows one `null` element.
    - No guaranteed order of elements (unlike `LinkedHashSet` or `TreeSet`).
    - Offers constant-time performance for basic operations like `add`, `remove`, and `contains` (on average).
    - Not synchronized (use `Collections.synchronizedSet` for thread safety).

---

**2. Creating a HashSet**

**a. Default Constructor**

```java
HashSet<Type> set = new HashSet<>();
```

**b. With Initial Capacity**

```java
HashSet<Type> set = new HashSet<>(initialCapacity);
```

**c. With Initial Capacity and Load Factor**

```java
HashSet<Type> set = new HashSet<>(initialCapacity, loadFactor);
```

**d. From Another Collection**

```java
HashSet<Type> set = new HashSet<>(existingCollection);
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

---

**4. Adding and Removing Elements**

**a. Adding Elements**

```java
HashSet<String> set = new HashSet<>();
set.add("Alice");
set.add("Bob");
set.add("Charlie");
```

**b. Removing Elements**

```java
set.remove("Bob");
```

**c. Handling Duplicates**

```java
set.add("Alice"); // Duplicate, will not be added
```

---

**5. Iterating Over a HashSet**

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

**a. Unordered Collection**

- Elements in a `HashSet` are unordered and can change order over time.

```java
HashSet<Integer> set = new HashSet<>();
set.add(3);
set.add(1);
set.add(2);
System.out.println(set); // Output order may vary: [1, 2, 3]
```

**b. Unique Elements**

- Duplicate elements are not allowed.

```java
set.add(1);
set.add(1); // Ignored, no duplicates allowed
```

**c. Allows `null`**

- Only one `null` element is permitted.

```java
set.add(null);
```

---

**7. HashSet and Hashing**

- **Hashing**: HashSet uses a `HashMap` internally to store elements. The hash value of each element is computed to determine its bucket location.
- **Collision Resolution**: When two elements have the same hash value, they are stored in the same bucket as a linked list or tree (depending on the implementation).

---

**8. Performance**

- **Time Complexity**:
    - `add()`, `remove()`, `contains()`: **O(1)** (on average).
    - Iteration: **O(n)**.
- **Space Complexity**:
    - Depends on the number of elements and the load factor.

---

**9. Thread-Safe HashSet**

HashSet is **not synchronized**. For concurrent access:

- Use `Collections.synchronizedSet`:

```java
Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());
```

---

**10. Differences Between HashSet and Other Sets**

|Feature|HashSet|LinkedHashSet|TreeSet|
|---|---|---|---|
|Ordering|No order|Insertion order|Sorted order|
|Null Values|Allows one `null`|Allows one `null`|Does not allow `null`|
|Performance|Fast (O(1))|Slightly slower|Slower (O(log n))|
|Use Case|General-purpose set|Predictable iteration order|Sorted data access|

---

**11. Advanced Features**

**a. Bulk Operations**

- **Adding All Elements**

```java
HashSet<String> set1 = new HashSet<>();
set1.add("Alice");
set1.add("Bob");

HashSet<String> set2 = new HashSet<>();
set2.addAll(set1); // Adds all elements from set1 to set2
```

- **Removing All Elements**

```java
set1.removeAll(set2); // Removes all elements in set2 from set1
```

- **Retaining Common Elements**

```java
set1.retainAll(set2); // Retains only elements present in both sets
```

---

**12. Common Interview Questions**

1. **How does HashSet work internally?**
    
    - HashSet is backed by a `HashMap`. When an element is added, it is stored as a key in the HashMap with a dummy value.
2. **What happens if you insert duplicate elements?**
    
    - Duplicate elements are ignored because `HashSet` only stores unique values.
3. **Can a HashSet store `null` elements?**
    
    - Yes, it allows one `null` element.
4. **What is the default initial capacity and load factor?**
    
    - Default capacity: **16**.
    - Default load factor: **0.75**.
5. **How does HashSet handle collisions?**
    
    - Collisions are handled using linked lists or trees in the underlying `HashMap`.

---

**13. Differences Between HashSet and HashMap**

|Feature|HashSet|HashMap|
|---|---|---|
|Implementation|Backed by a HashMap|A key-value pair collection|
|Key-Value Pair|Stores only keys|Stores keys and values|
|Null Handling|Allows one `null` key|Allows one `null` key and multiple `null` values|

---