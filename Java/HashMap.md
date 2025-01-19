### Java `HashMap`: A Comprehensive Guide

**1. What is a HashMap?**

- **Definition**: A `HashMap` is a data structure in Java that implements the `Map` interface, storing key-value pairs.
- **Key Features**:
    - Allows storage of `null` values and a single `null` key.
    - Does not maintain the insertion order (use `LinkedHashMap` for that).
    - Not synchronized (use `ConcurrentHashMap` for thread safety).
    - Provides constant-time complexity for basic operations like `get` and `put` (on average).

---

**2. Creating a HashMap**

**a. Using Default Constructor**

```java
HashMap<KeyType, ValueType> map = new HashMap<>();
```

**b. Specifying Initial Capacity and Load Factor**

```java
HashMap<KeyType, ValueType> map = new HashMap<>(initialCapacity, loadFactor);
```

- **Initial Capacity**: Default is **16**.
- **Load Factor**: Default is **0.75**, meaning the map resizes when 75% full.

**c. Using Another Map**

```java
HashMap<KeyType, ValueType> map = new HashMap<>(existingMap);
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
|`keySet()`|Returns a set of all keys.|
|`values()`|Returns a collection of all values.|
|`entrySet()`|Returns a set of all key-value pairs (entries).|
|`clear()`|Removes all key-value pairs from the map.|

---

**4. Adding and Accessing Elements**

**a. Adding Elements**

```java
HashMap<String, Integer> map = new HashMap<>();
map.put("Alice", 25);
map.put("Bob", 30);
map.put("Charlie", 35);
```

**b. Accessing Elements**

```java
int age = map.get("Alice"); // 25
```

**c. Handling Missing Keys**

```java
Integer age = map.getOrDefault("David", 0); // Returns 0 if "David" is not present
```

---

**5. Iterating Over a HashMap**

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

**6. Removing Elements**

**a. Remove by Key**

```java
map.remove("Bob");
```

**b. Remove by Key and Value**

```java
map.remove("Charlie", 35); // Removes only if key "Charlie" maps to value 35
```

---

**7. Key Characteristics**

**a. Key and Value Restrictions**

- Keys must be unique.
- Values can be duplicated.
- A single `null` key is allowed.
- Multiple `null` values are allowed.

**b. Handling Collisions**

- Uses a combination of **hashing** and **linked lists** (or trees in case of too many collisions) to handle collisions.

**c. Resizing**

- HashMap resizes dynamically when the number of entries exceeds `capacity * load factor`.

---

**8. Internal Working of HashMap**

1. **Hashing**:
    
    - Keys are hashed using their `hashCode()` method.
    - The hash value is converted into a bucket index using `hash % capacity`.
2. **Collision Resolution**:
    
    - If two keys hash to the same bucket, they are stored as a linked list or a balanced tree for high collision cases.
3. **Retrieval**:
    
    - The `get(key)` operation hashes the key, identifies the bucket, and traverses the linked list/tree to find the value.

---

**9. Thread-Safe HashMap** `HashMap` is not thread-safe. For concurrent access:

- Use `Collections.synchronizedMap`:

```java
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
```

- Or use `ConcurrentHashMap`:

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
```

---

**10. Differences Between HashMap and Other Maps**

|Feature|HashMap|LinkedHashMap|TreeMap|
|---|---|---|---|
|Ordering|No ordering|Insertion order|Sorted order|
|Null Keys|Allows one `null` key|Allows one `null` key|Does not allow `null` keys|
|Performance|Fast (O(1) for `get`)|Slightly slower|Slower (O(log n))|
|Use Case|General-purpose map|Maintain order|Sorted data access|

---

**11. Performance**

- **Time Complexity**:
    - `put()`: **O(1)** (amortized).
    - `get()`: **O(1)** (amortized).
    - `remove()`: **O(1)** (amortized).
- **Space Complexity**:
    - Depends on the number of buckets and the load factor.

---

**12. Common Interview Questions**

1. **How does HashMap work internally?**
    
    - HashMap uses hashing to map keys to bucket indices. Each bucket stores entries as a linked list or tree.
2. **What happens when two keys have the same hash code?**
    
    - A collision occurs. The entries are stored in the same bucket as a linked list or a tree structure for faster access.
3. **What is the default initial capacity and load factor of a HashMap?**
    
    - Initial capacity: **16**.
    - Load factor: **0.75**.
4. **Why is HashMap not thread-safe?**
    
    - Multiple threads accessing and modifying a HashMap can cause data inconsistency.
5. **When to use HashMap over TreeMap or LinkedHashMap?**
    
    - Use `HashMap` for fast lookups and general-purpose mappings.

---