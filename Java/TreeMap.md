### Java `TreeMap`: A Comprehensive Guide

**1. What is a TreeMap?**

- **Definition**: A `TreeMap` is a part of Java’s `java.util` package that implements the `NavigableMap` interface and is based on a **Red-Black Tree**.
- **Key Features**:
    - Stores key-value pairs in **sorted order** (based on keys).
    - By default, it uses the **natural ordering** of keys (`Comparable`) or a custom **Comparator**.
    - Does not allow `null` keys but allows multiple `null` values.
    - Not synchronized (use `Collections.synchronizedMap` for thread safety).

---

**2. Creating a TreeMap**

**a. Default Constructor**

```java
TreeMap<KeyType, ValueType> map = new TreeMap<>();
```

**b. Using Custom Comparator**

```java
TreeMap<KeyType, ValueType> map = new TreeMap<>(Comparator.reverseOrder());
```

**c. Using Another Map**

```java
TreeMap<KeyType, ValueType> map = new TreeMap<>(existingMap);
```

---

**3. Common Operations**

|Method|Description|
|---|---|
|`put(key, value)`|Inserts or updates a key-value pair.|
|`get(key)`|Retrieves the value associated with the specified key.|
|`remove(key)`|Removes the key-value pair for the specified key.|
|`containsKey(key)`|Checks if the map contains the specified key.|
|`containsValue(value)`|Checks if the map contains the specified value.|
|`size()`|Returns the number of key-value pairs.|
|`isEmpty()`|Checks if the map is empty.|
|`firstKey()`|Returns the smallest key.|
|`lastKey()`|Returns the largest key.|
|`floorKey(key)`|Returns the largest key ≤ given key.|
|`ceilingKey(key)`|Returns the smallest key ≥ given key.|
|`subMap(fromKey, toKey)`|Returns a view of the portion of this map within the range.|
|`keySet()`|Returns a set of all keys.|
|`values()`|Returns a collection of all values.|
|`entrySet()`|Returns a set of all key-value pairs (entries).|
|`clear()`|Removes all key-value pairs from the map.|

---

**4. Adding and Accessing Elements**

**a. Adding Elements**

```java
TreeMap<String, Integer> map = new TreeMap<>();
map.put("Alice", 25);
map.put("Bob", 30);
map.put("Charlie", 35);
```

**b. Accessing Elements**

```java
int age = map.get("Alice"); // Retrieves 25
```

**c. Handling Missing Keys**

```java
Integer age = map.getOrDefault("David", 0); // Returns 0 if "David" is not present
```

---

**5. Iterating Over a TreeMap**

**a. Iterating Over Keys**

```java
for (String key : map.keySet()) {
    System.out.println("Key: " + key);
}
```

**b. Iterating Over Values**

```java
for (Integer value : map.values()) {
    System.out.println("Value: " + value);
}
```

**c. Iterating Over Entries**

```java
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println("Key: " + entry.getKey() + ", Value: " + entry.getValue());
}
```

---

**6. Key Characteristics**

**a. Sorted Order**

- Keys are stored in **ascending natural order** or a custom order if a comparator is provided.
- Example:

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(3, "Three");
map.put(1, "One");
map.put(2, "Two");
System.out.println(map); // Output: {1=One, 2=Two, 3=Three}
```

**b. Custom Comparator**

```java
TreeMap<Integer, String> map = new TreeMap<>(Comparator.reverseOrder());
map.put(3, "Three");
map.put(1, "One");
map.put(2, "Two");
System.out.println(map); // Output: {3=Three, 2=Two, 1=One}
```

**c. Range Views**

- **Submap**: Extracts a portion of the map.

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(1, "One");
map.put(2, "Two");
map.put(3, "Three");
map.put(4, "Four");

System.out.println(map.subMap(2, 4)); // Output: {2=Two, 3=Three}
```

**d. Navigational Methods**

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(1, "One");
map.put(3, "Three");
map.put(5, "Five");

System.out.println(map.floorKey(4)); // Output: 3 (largest key ≤ 4)
System.out.println(map.ceilingKey(2)); // Output: 3 (smallest key ≥ 2)
System.out.println(map.firstKey()); // Output: 1
System.out.println(map.lastKey());  // Output: 5
```

---

**7. Removing Elements**

**a. Remove by Key**

```java
map.remove("Bob");
```

**b. Remove by Key and Value**

```java
map.remove("Charlie", 35); // Removes only if key "Charlie" maps to value 35
```

---

**8. Performance**

- **Time Complexity**:
    - `put()`, `get()`, `remove()`: **O(log n)**.
    - Iteration: **O(n)**.
- **Space Complexity**:
    - Depends on the number of entries and the Red-Black Tree structure.

---

**9. Differences Between TreeMap and Other Maps**

|Feature|TreeMap|HashMap|LinkedHashMap|
|---|---|---|---|
|Ordering|Sorted Order|No ordering|Insertion/Access Order|
|Null Keys|Not allowed|Allows one `null` key|Allows one `null` key|
|Performance|Slower (O(log n))|Faster (O(1))|Slightly slower than HashMap|
|Use Case|Sorted data access|General-purpose map|Predictable iteration order|

---

**10. Thread-Safe TreeMap**

`TreeMap` is not thread-safe. For concurrent access:

- Use `Collections.synchronizedMap`:

```java
Map<Integer, String> syncMap = Collections.synchronizedMap(new TreeMap<>());
```

---

**11. Common Interview Questions**

1. **How does TreeMap maintain order?**
    
    - It uses a **Red-Black Tree**, ensuring keys are stored in sorted order.
2. **What happens if you insert a `null` key?**
    
    - A `NullPointerException` is thrown because `TreeMap` does not allow `null` keys.
3. **Can TreeMap store duplicate keys?**
    
    - No, keys must be unique. Duplicate keys overwrite the previous value.
4. **When to use TreeMap over HashMap or LinkedHashMap?**
    
    - Use `TreeMap` when sorted key access is required.
5. **How is TreeMap different from TreeSet?**
    
    - `TreeMap` stores key-value pairs, while `TreeSet` stores only keys.

---