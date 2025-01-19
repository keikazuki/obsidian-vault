### Java `LinkedHashMap`: A Comprehensive Guide

**1. What is a LinkedHashMap?**

- **Definition**: A `LinkedHashMap` is a class in Java that extends `HashMap` and maintains a doubly-linked list running through all its entries.
- **Key Features**:
    - Maintains **insertion order** of keys or access order (depending on configuration).
    - Allows **one `null` key** and **multiple `null` values**.
    - Not synchronized (use `Collections.synchronizedMap` or `ConcurrentHashMap` for thread safety).
    - Provides predictable iteration order.

---

**2. Creating a LinkedHashMap**

**a. Default Constructor**

```java
LinkedHashMap<KeyType, ValueType> map = new LinkedHashMap<>();
```

**b. With Initial Capacity and Load Factor**

```java
LinkedHashMap<KeyType, ValueType> map = new LinkedHashMap<>(initialCapacity, loadFactor);
```

**c. With Access Order**

- To maintain access order instead of insertion order:

```java
LinkedHashMap<KeyType, ValueType> map = new LinkedHashMap<>(initialCapacity, loadFactor, true);
```

- **true**: Enables access-order.
- **false**: Maintains insertion-order (default).

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
LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
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

**5. Iterating Over a LinkedHashMap**

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

**a. Insertion vs Access Order**

- By default, `LinkedHashMap` maintains **insertion order**.
- If created with access-order mode (`true`), the iteration order reflects the order in which entries were **last accessed**.

**b. Access Order Example**

```java
LinkedHashMap<String, Integer> map = new LinkedHashMap<>(16, 0.75f, true);
map.put("Alice", 25);
map.put("Bob", 30);
map.put("Charlie", 35);

map.get("Alice"); // Access "Alice"

for (String key : map.keySet()) {
    System.out.println(key); // Output: Bob, Charlie, Alice (access order)
}
```

**c. Overriding `removeEldestEntry`**

- To create a **bounded map** (automatically removes the oldest entry when a size limit is exceeded):

```java
LinkedHashMap<String, Integer> map = new LinkedHashMap<>(16, 0.75f, true) {
    @Override
    protected boolean removeEldestEntry(Map.Entry<String, Integer> eldest) {
        return size() > 3; // Remove oldest if size exceeds 3
    }
};
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
    - `put()`, `get()`, `remove()`: **O(1)** (on average).
    - Iteration: **O(n)**.
- **Space Complexity**:
    - Higher than `HashMap` because of the linked list used to maintain order.

---

**9. Differences Between LinkedHashMap and Other Maps**

|Feature|LinkedHashMap|HashMap|TreeMap|
|---|---|---|---|
|Ordering|Insertion/Access Order|No ordering|Sorted Order|
|Null Keys|Allows one `null` key|Allows one `null` key|Does not allow `null` keys|
|Performance|Slightly slower than HashMap|Fast (O(1) operations)|Slower (O(log n))|
|Use Case|Predictable iteration order|General-purpose map|Sorted data access|

---

**10. Thread-Safe LinkedHashMap**

`LinkedHashMap` is not thread-safe. For concurrent access:

- Use `Collections.synchronizedMap`:

```java
Map<String, Integer> syncMap = Collections.synchronizedMap(new LinkedHashMap<>());
```

---

**11. Common Interview Questions**

1. **How does LinkedHashMap maintain order?**
    
    - It maintains a doubly-linked list of all entries, ensuring predictable iteration order (insertion or access order).
2. **What happens when you access a key in access-order mode?**
    
    - The accessed entry is moved to the end of the linked list, reflecting the latest access order.
3. **Can LinkedHashMap store null keys and values?**
    
    - Yes, it allows one `null` key and multiple `null` values.
4. **When to use LinkedHashMap over HashMap?**
    
    - Use `LinkedHashMap` when you need predictable iteration order.
5. **How to create a fixed-size LinkedHashMap?**
    
    - Override `removeEldestEntry`.

---