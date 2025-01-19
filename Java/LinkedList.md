### Java LinkedList: A Comprehensive Guide

**1. What is a LinkedList?**

- **Definition**: A `LinkedList` is a data structure in Java, part of the `java.util` package, that implements the `List` and `Deque` interfaces.
- **Key Features**:
    - Elements are stored as nodes, each containing data and references to the next and/or previous node.
    - Doubly linked structure: Each node links to both its predecessor and successor.
    - Dynamic size: No need to define size in advance.
    - Allows duplicate elements.
    - Maintains insertion order.

---

**2. Creating a LinkedList**

```java
import java.util.LinkedList;

// Generic Syntax
LinkedList<Type> list = new LinkedList<Type>();
```

- Example:

```java
LinkedList<String> names = new LinkedList<>();
LinkedList<Integer> numbers = new LinkedList<>();
```

---

**3. Common Operations**

**a. Adding Elements**

```java
list.add(element);       // Adds to the end
list.add(index, element); // Inserts at a specific index
list.addFirst(element);   // Adds to the beginning
list.addLast(element);    // Adds to the end
```

Example:

```java
names.add("Alice");
names.add("Bob");
names.addFirst("Charlie"); // Adds "Charlie" at the beginning
names.addLast("David");    // Adds "David" at the end
```

---

**b. Accessing Elements**

```java
element = list.get(index);      // Gets the element at a specific index
element = list.getFirst();      // Gets the first element
element = list.getLast();       // Gets the last element
```

Example:

```java
String firstName = names.getFirst(); // "Charlie"
String lastName = names.getLast();   // "David"
```

---

**c. Modifying Elements**

```java
list.set(index, newElement);
```

Example:

```java
names.set(1, "Eve"); // Changes "Alice" to "Eve"
```

---

**d. Removing Elements**

```java
list.remove(index);       // Removes element at the specified index
list.remove(Object);      // Removes the first occurrence of the object
list.removeFirst();       // Removes the first element
list.removeLast();        // Removes the last element
```

Example:

```java
names.remove(1);          // Removes "Eve"
names.remove("David");    // Removes "David"
names.removeFirst();      // Removes "Charlie"
names.removeLast();       // Removes "Bob"
```

---

**e. Checking Size**

```java
int size = list.size();
```

---

**f. Iterating Over a LinkedList**

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

|Method|Description|
|---|---|
|`contains(Object)`|Checks if the list contains an element.|
|`isEmpty()`|Checks if the list is empty.|
|`indexOf(Object)`|Returns the index of the first occurrence.|
|`lastIndexOf(Object)`|Returns the index of the last occurrence.|
|`peek()`|Retrieves the first element without removing it.|
|`poll()`|Retrieves and removes the first element.|
|`toArray()`|Converts the list to an array.|

Example:

```java
if (names.contains("Alice")) {
    System.out.println("Alice is in the list!");
}
```

---

**5. Sorting**

```java
import java.util.Collections;

Collections.sort(list);   // Ascending order
Collections.reverse(list); // Descending order
```

Example:

```java
LinkedList<Integer> numbers = new LinkedList<>();
numbers.add(3);
numbers.add(1);
numbers.add(2);

Collections.sort(numbers);   // [1, 2, 3]
Collections.reverse(numbers); // [3, 2, 1]
```

---

**6. LinkedList as a Queue or Stack**

**a. Queue Operations**

- Add elements: `offer()`, `add()`
- Remove elements: `poll()`, `remove()`
- Peek elements: `peek()`, `element()`

Example:

```java
LinkedList<String> queue = new LinkedList<>();
queue.offer("Alice");
queue.offer("Bob");
System.out.println(queue.poll()); // Removes and returns "Alice"
```

**b. Stack Operations**

- Push elements: `push()`
- Pop elements: `pop()`
- Peek elements: `peek()`

Example:

```java
LinkedList<String> stack = new LinkedList<>();
stack.push("Alice");
stack.push("Bob");
System.out.println(stack.pop()); // Removes and returns "Bob"
```

---

**7. Differences Between ArrayList and LinkedList**

|Feature|ArrayList|LinkedList|
|---|---|---|
|Storage Mechanism|Dynamic array|Doubly linked nodes|
|Access Time|Faster (O(1))|Slower (O(n))|
|Insert/Delete|Slower (O(n))|Faster (O(1) for ends)|
|Memory Usage|Less (no extra links)|More (due to links)|
|Iteration Performance|Better for random access|Better for sequential|

---

**8. Thread-Safe LinkedList** For a thread-safe `LinkedList`, use `Collections.synchronizedList`:

```java
List<String> synchronizedList = Collections.synchronizedList(new LinkedList<>());
```

---

**9. Common Interview Questions**

1. **How does LinkedList work internally?**
    
    - Each element is a node containing three parts: `data`, `next`, and `prev`.
    - Nodes are linked in a doubly linked structure.
2. **When should you use LinkedList over ArrayList?**
    
    - Use `LinkedList` for frequent insertions and deletions.
    - Use `ArrayList` for faster access and iteration.
3. **Can a LinkedList store `null` values?**
    
    - Yes, it allows multiple `null` values.
4. **What happens if you access an invalid index?**
    
    - Throws `IndexOutOfBoundsException`.

---