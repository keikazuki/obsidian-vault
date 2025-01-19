### Java `LinkedHashSet`: A Comprehensive Guide

**1. What is a LinkedHashSet?**

- **Definition**: A `LinkedHashSet` is a part of Java’s `java.util` package that extends `HashSet` and maintains the **insertion order** of elements.
- **Key Features**:
    - Ensures **unique elements** (like `HashSet`).
    - Maintains **insertion order** (unlike `HashSet`).
    - Allows one `null` element.
    - Not synchronized (use `Collections.synchronizedSet` for thread safety).

---

**2. Creating a LinkedHashSet**

**a. Default Constructor**

```java
LinkedHashSet<Type> set = new LinkedHashSet<>();
```

**b. With Initial Capacity**

```java
LinkedHashSet<Type> set = new LinkedHashSet<>(initialCapacity);
```

**c. With Initial Capacity and Load Factor**

```java
LinkedHashSet<Type> set = new LinkedHashSet<>(initialCapacity, loadFactor);
```

**d. From Another Collection**

```java
LinkedHashSet<Type> set = new LinkedHashSet<>(existingCollection);
```

---

**3. Common Operations**

|Method|Description|
|---|---|
|`add(element)`|Adds the element to the set if it is not already present.|
|`remove(element)`|Removes the specified element from the set.|
|`contains(element)`|Checks if the set contains the specified element.|
|`size()`|Returns the number of elements in the set.|
|`isEmpty()`|Checks if the set is empty.|
|`clear()`|Removes all elements from the set.|
|`iterator()`|Returns an iterator over the elements in the set.|

---

**4. Adding and Removing Elements**

**a. Adding Elements**

```java
LinkedHashSet<String> set = new LinkedHashSet<>();
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

**5. Iterating Over a LinkedHashSet**

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

**a. Maintains Insertion Order**

- `LinkedHashSet` preserves the order in which elements were inserted.

```java
LinkedHashSet<Integer> set = new LinkedHashSet<>();
set.add(3);
set.add(1);
set.add(2);
System.out.println(set); // Output: [3, 1, 2]
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

**7. Internal Working**

- `LinkedHashSet` is implemented using a combination of `HashSet` and a doubly-linked list.
- **HashSet** ensures unique elements using a hash table.
- **Doubly-Linked List** maintains the insertion order by linking entries in the order they were added.

---

**8. Performance**

- **Time Complexity**:
    - `add()`, `remove()`, `contains()`: **O(1)** (on average).
    - Iteration: **O(n)**.
- **Space Complexity**:
    - Higher than `HashSet` due to the doubly-linked list used to maintain order.

---

**9. Thread-Safe LinkedHashSet**

`LinkedHashSet` is not synchronized. For concurrent access:

- Use `Collections.synchronizedSet`:

```java
Set<String> syncSet = Collections.synchronizedSet(new LinkedHashSet<>());
```

---

**10. Differences Between LinkedHashSet and Other Collections**

|Feature|LinkedHashSet|HashSet|TreeSet|
|---|---|---|---|
|Ordering|Insertion order|No order|Sorted order|
|Null Values|Allows one `null`|Allows one `null`|Does not allow `null`|
|Performance|Slightly slower than HashSet|Fast (O(1))|Slower (O(log n))|
|Use Case|Predictable iteration order|General-purpose set|Sorted data access|

---

**11. Advanced Features**

**a. Bulk Operations**

- **Adding All Elements**

```java
LinkedHashSet<String> set1 = new LinkedHashSet<>();
set1.add("Alice");
set1.add("Bob");

LinkedHashSet<String> set2 = new LinkedHashSet<>();
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

**12. Use Cases**

1. **Preserving Insertion Order**
    
    - Use `LinkedHashSet` when you need unique elements and must maintain the order of insertion.
2. **Efficient Lookups**
    
    - Like `HashSet`, `LinkedHashSet` offers constant-time performance for `add`, `remove`, and `contains`.
3. **Filtering Duplicates**
    
    - Use `LinkedHashSet` to filter duplicates while preserving the order of elements in a collection.

---

**13. Common Interview Questions**

1. **How does LinkedHashSet maintain order?**
    
    - `LinkedHashSet` uses a doubly-linked list to maintain insertion order and a hash table to ensure unique elements.
2. **What happens if you insert duplicate elements?**
    
    - Duplicate elements are ignored; `LinkedHashSet` only stores unique elements.
3. **Can LinkedHashSet store `null` values?**
    
    - Yes, it allows one `null` value.
4. **How is LinkedHashSet different from HashSet?**
    
    - `LinkedHashSet` maintains insertion order, while `HashSet` does not.
5. **When should you use LinkedHashSet over TreeSet or HashSet?**
    
    - Use `LinkedHashSet` when you need both unique elements and predictable insertion order.

---