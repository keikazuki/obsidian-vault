### Java PriorityQueue: A Comprehensive Guide

**1. What is a PriorityQueue?**

- **Definition**: A `PriorityQueue` in Java, part of the `java.util` package, is a queue that orders its elements based on their natural ordering or a custom comparator specified during its creation. It is a specialized data structure where elements are prioritized, ensuring that the element with the highest or lowest priority (depending on the ordering) is always served first.
- **Key Features**:
    - Elements are processed in priority order, not insertion order.
    - Implements the `Queue` interface.
    - Does not allow `null` elements.
    - Not thread-safe (use `PriorityBlockingQueue` for thread-safe operations).

---

**2. Characteristics of PriorityQueue**

- By default, elements are ordered in **natural order** (ascending for numbers, lexicographical for strings).
- It is a **min-heap** by default:
    - The head of the queue is the smallest element.
- Can be configured as a **max-heap** using a custom comparator.

---

**3. Creating a PriorityQueue**

**a. Using Default Constructor**

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
```

**b. Using Custom Comparator**

```java
PriorityQueue<Integer> pq = new PriorityQueue<>(Comparator.reverseOrder()); // Max-heap
```

**c. Specifying Initial Capacity**

```java
PriorityQueue<Integer> pq = new PriorityQueue<>(10); // Initial capacity of 10
```

**d. Using a Collection**

```java
List<Integer> list = Arrays.asList(3, 1, 4, 1, 5);
PriorityQueue<Integer> pq = new PriorityQueue<>(list);
```

---

**4. Common Operations**

|Method|Description|
|---|---|
|`add(E e)`|Inserts the specified element into the queue.|
|`offer(E e)`|Inserts the specified element (preferable over `add`).|
|`peek()`|Retrieves, but does not remove, the head of the queue.|
|`poll()`|Retrieves and removes the head of the queue.|
|`remove(Object o)`|Removes a single instance of the specified element.|
|`clear()`|Removes all elements from the queue.|
|`size()`|Returns the number of elements in the queue.|
|`toArray()`|Converts the queue to an array.|

---

**5. Adding and Removing Elements**

**a. Adding Elements**

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.add(5);
pq.add(1);
pq.add(3);
```

**b. Removing Elements**

```java
int head = pq.poll(); // Removes and retrieves the smallest element (1)
int next = pq.peek(); // Retrieves the next smallest element (3) without removing
```

---

**6. Iterating Over a PriorityQueue**

The iteration does not guarantee any specific order.

**a. Using For-Each Loop**

```java
for (int num : pq) {
    System.out.println(num);
}
```

**b. Using Iterator**

```java
Iterator<Integer> iterator = pq.iterator();
while (iterator.hasNext()) {
    System.out.println(iterator.next());
}
```

---

**7. Custom Comparator**

**a. Max-Heap Example**

```java
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.add(5);
maxHeap.add(1);
maxHeap.add(3);

System.out.println(maxHeap.poll()); // Outputs 5 (largest element)
```

**b. Custom Object Example**

```java
class Task {
    String name;
    int priority;

    Task(String name, int priority) {
        this.name = name;
        this.priority = priority;
    }
}

// Comparator to sort tasks by priority
Comparator<Task> taskComparator = (t1, t2) -> Integer.compare(t1.priority, t2.priority);

PriorityQueue<Task> taskQueue = new PriorityQueue<>(taskComparator);

taskQueue.add(new Task("Task1", 3));
taskQueue.add(new Task("Task2", 1));
taskQueue.add(new Task("Task3", 2));

Task highestPriorityTask = taskQueue.poll();
System.out.println(highestPriorityTask.name); // Outputs "Task2"
```

---

**8. Key Points**

- **Heap Property**: `PriorityQueue` uses a **binary heap** internally.
    - Min-heap for natural order.
    - Max-heap or custom order using a comparator.
- **Capacity Growth**: The capacity grows dynamically as needed.
- **Time Complexity**:
    - Insertion: **O(log n)**.
    - Deletion (poll): **O(log n)**.
    - Peek: **O(1)**.

---

**9. Example Use Cases**

**a. Task Scheduling** Tasks with higher priority are executed first.

**b. Dijkstra’s Algorithm** To find the shortest path in a graph, a `PriorityQueue` is used to process nodes based on their current distance.

**c. Merging K Sorted Lists** Merge multiple sorted lists using a `PriorityQueue` to always process the smallest element first.

---

**10. Differences Between PriorityQueue and Other Collections**

|Feature|PriorityQueue|ArrayList|LinkedList|
|---|---|---|---|
|Ordering|Priority order|Insertion order|Insertion order|
|Random Access|No|Yes|No|
|Use Case|Processing by priority|Dynamic list storage|Queue/Stack operations|
|Thread Safety|No|No|No|

---

**11. Interview Questions**

1. **How does PriorityQueue work internally?**
    
    - Uses a binary heap.
    - The smallest (or highest-priority) element is always at the root.
2. **Can PriorityQueue store `null` elements?**
    
    - No, `null` values are not allowed.
3. **How can you implement a max-heap with PriorityQueue?**
    
    - Use a custom comparator with `Comparator.reverseOrder()`.
4. **What is the time complexity of adding/removing elements?**
    
    - Insertion: **O(log n)**.
    - Removal (poll): **O(log n)**.
5. **Can PriorityQueue contain duplicate elements?**
    
    - Yes, duplicates are allowed.

---