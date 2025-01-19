### Java ArrayList: A Comprehensive Guide

**1. What is an ArrayList?**
- **Definition**: An `ArrayList` is a resizable array in Java, part of the `java.util` package.
- **Key Features**:
  - Dynamically resizable (unlike arrays with fixed size).
  - Allows duplicate elements.
  - Maintains the insertion order.
  - Non-synchronized (not thread-safe).

---

**2. Creating an ArrayList**
```java
import java.util.ArrayList;

// Generic Syntax
ArrayList<Type> list = new ArrayList<Type>();
```

- Example:
```java
ArrayList<String> names = new ArrayList<>();
ArrayList<Integer> numbers = new ArrayList<>();
```

---

**3. Common Operations**

**a. Adding Elements**
```java
list.add(element);
list.add(index, element); // Inserts at a specific index
```

Example:
```java
names.add("Alice");
names.add("Bob");
names.add(1, "Charlie"); // Inserts "Charlie" at index 1
```

---

**b. Accessing Elements**
```java
element = list.get(index);
```

Example:
```java
String name = names.get(1); // Gets the element at index 1 ("Charlie")
```

---

**c. Modifying Elements**
```java
list.set(index, newElement);
```

Example:
```java
names.set(1, "David"); // Changes "Charlie" to "David"
```

---

**d. Removing Elements** (O(n))
```java
list.remove(index); // Removes element at specified index
list.remove(Object); // Removes first occurrence of specified object
```

Example:
```java
names.remove(1); // Removes "David"
names.remove("Alice"); // Removes "Alice"
```

---

**e. Checking Size**
```java
int size = list.size();
```

---

**f. Iterating Over an ArrayList**

1. **For Loop**
```java
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}
```

2. **Enhanced For Loop**
```java
for (String name : names) {
    System.out.println(name);
}
```

3. **Using Iterator**
```java
Iterator<String> iterator = names.iterator();
while (iterator.hasNext()) {
    System.out.println(iterator.next());
}
```

---

**g. Clearing the List**
```java
list.clear();
```

---

**4. Useful Methods**

| Method                | Description                               |
|-----------------------|-------------------------------------------|
| `contains(Object)`    | Checks if the list contains an element.  |
| `isEmpty()`           | Checks if the list is empty.             |
| `indexOf(Object)`     | Returns the index of the first occurrence of the element. |
| `lastIndexOf(Object)` | Returns the index of the last occurrence of the element. |
| `toArray()`           | Converts the list to an array.           |
| `subList(start, end)` | Extracts a portion of the list.           |

Example:
```java
if (names.contains("Bob")) {
    System.out.println("Bob is in the list!");
}
```

---

**5. Sorting**
```java
import java.util.Collections;

Collections.sort(list); // Ascending order
Collections.reverse(list); // Descending order
```

Example:
```java
ArrayList<Integer> numbers = new ArrayList<>();
numbers.add(3);
numbers.add(1);
numbers.add(2);

Collections.sort(numbers); // [1, 2, 3]
Collections.reverse(numbers); // [3, 2, 1]
```

---

**6. Converting to Other Collections**

**a. Convert to Array**
```java
String[] array = names.toArray(new String[0]);
```

**b. Convert from Array**
```java
List<String> list = Arrays.asList(array);
```

---

**7. Differences Between Array and ArrayList**

| Feature               | Array                 | ArrayList             |
|-----------------------|-----------------------|-----------------------|
| Size                  | Fixed                | Dynamic               |
| Performance           | Faster               | Slower (due to resizing) |
| Data Type             | Primitive and Objects| Objects only          |
| Methods               | No built-in methods  | Rich API support      |

---

**8. Thread-Safe ArrayList**
For a thread-safe ArrayList, use `Collections.synchronizedList`:
```java
List<String> synchronizedList = Collections.synchronizedList(new ArrayList<>());
```

---

**9. Common Interview Questions**
1. **How does ArrayList work internally?**
   - Uses a dynamic array.
   - When the capacity is exceeded, it grows by 50% of the current size.
   - Array elements are copied to a new array during resizing.

2. **What is the default initial capacity of an ArrayList?**
   - Default capacity is **10**.

3. **Can an ArrayList store null values?**
   - Yes, it can store multiple `null` values.

4. **What happens if you access an invalid index?**
   - Throws `IndexOutOfBoundsException`.

---