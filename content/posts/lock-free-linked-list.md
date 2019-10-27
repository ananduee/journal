---
title: "Lock Free Linked List"
date: 2019-10-27T14:09:18+05:30
draft: false
---

This is my first blog post. Through this blog I want to learn and explore different paradigms of programming and software development. So this blog will mostly be in format of notes of things which I have read and some of my thoughts around them. I will try to ensure that I always metion links which I have used to research on the topic.

On of my personal favuorite paradigms in programming is asynchronous programming so wanted to start with a very basic lock free data structure "Lock free linked list".

### What does lock free programming mean ?

Let's say we have some data which others can be read as well as change. Depending on the methods we provide to change the data we might leave data in in-consistent state. for example :-

Let's say we have a variable ```int value = 0``` and we provide a method to increment this value. Here is how code of "increment" method would look :-

```java
void increment() {
    value = value + 1;
}
```

Till time we call this method from one thread or one by one everything will be fine. Now lets say multiple threads start calling this method parallely. Here is java code for the same here we are calling "increment" method 10000 times. We are only using 2 threads so roughly each thread will call increment 5000 times.

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

class IncrementTesting
{
    public static int value = 0;

    public static void main (String[] args) throws java.lang.Exception
    {
        final int loopSize = 10000; // You can change loop size to experiment.
        // Ensure you use more than 1 thread.
        final ExecutorService executor = Executors.newFixedThreadPool(2); 
        final CompletableFuture<Integer>[] workerTasks = new CompletableFuture[loopSize];
        for (int i = 0; i < loopSize; ++i) {
            // Start work asynchronously.
            workerTasks[i] = CompletableFuture.supplyAsync(() -> increment(), executor); 
        }
        // Wait for all async tasks to finish.
        CompletableFuture.allOf(workerTasks).join();
        System.out.println(value);
        executor.shutdown();
    }

    private static int increment() {
        value = value + 1;
        return value;
    }
}
```

On running this code e would expect this programm to print "10000", However here is output of this programme on my laptop :-

1. Run number 1 - 9994
1. Run number 2 - 9992
1. Run umber 3 - 9997
1. Run number 4 - 9998

In fact every run will give you a different output in some runs you can even get the desired "10000" value. If we see following are the things which are happening in method increment :-

1. Read the current value.
1. Increase the value by one
1. Set this value to existing variable.

If multiple threads are performing this action in parallel it might happen that a thread 1 (T1) has read the value and wants to increase it now another thread T2 reads this value before T1 fnishes setting the value in such cases value won't be increased in T2 as it has previous value.

Traditional way to solve this problem is by using locks. In java we can just make the method "increment" [synchronized](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html). This will ensure this method can only be called by one thread at a time. If multiple threads want to invoke this method they will automatically wait for this method to finish for active thread.

However locks/synchronization have a cost. In the above example second thread will always have to wait for first thread to finish, this effectively means that their is no advantage of using multiple threads as at all times their is only 1 method doing the work.

So general idea of lock free programming is - Minimize usage of locks by using [optimistic design](https://en.wikipedia.org/wiki/Optimistic_concurrency_control), [CAS](https://en.wikipedia.org/wiki/Compare-and-swap) etc. We can never eliminate locks fully as lock free programming poses certain limitations on how a programme can be written.

### Compare and swap (CAS)

As per wikipedia "CAS compares the contents of a memory location with a given input value and if input value is same as current value it modifies the contents of that memory location to a new given value.". So CAS can be imagined as function taking 3 arguments.

1. memory location
1. expected value at the memory location
1. new value to set at memory location

Most of modern CPU's support this operation by default i.e. In 1 operation you can compare value in a memory location and if found equal change it to a new value.

Java provides this through "AtomicInteger", "AtomicBoolean", "AtomicLong" etc classes. If you look at "compareAndSet" method inside "AtomicInteger" it is calling "Unsafe" class of java to hardware specific operation.

Here is how out programme would look with compare and swap operation

```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.atomic.AtomicInteger;

class CASTesting
{
    public static AtomicInteger value = new AtomicInteger(0);

    public static void main (String[] args) throws java.lang.Exception
    {
        final int loopSize = 10000; // You can change loop size to experiment.
        final ExecutorService executor = Executors.newFixedThreadPool(10);
        final CompletableFuture<Integer>[] workerTasks = new CompletableFuture[loopSize];
        for (int i = 0; i < loopSize; ++i) {
            workerTasks[i] = CompletableFuture.supplyAsync(() -> increment(), executor); // Start work asynchronously.
        }
        CompletableFuture.allOf(workerTasks).join(); // Wait for all async tasks to finish.
        System.out.println(value);
        executor.shutdown();
    }

    private static int increment() {
        int currentValue = value.intValue();
        while (!value.compareAndSet(currentValue, currentValue + 1)) {
            currentValue = value.intValue();
        }
        return currentValue;
    }
}
```

No matter how many threads we use here we will always get 10,000 which is our desired value.

The only thing which has changed here is that we used "AtomicIntger" instead of primitive "int".  AtomicIntger provides us atomic compare and set operation. We then assign value on which we want to increment in a seperate variable and loop till our desired value is set.

Good thing here is that none of the threads will be blocked so their is no deadlock scenario to worry about. (Deadlock is a scenario when one therad is waiting for another therad to release a lock and that lock is never released or original thread dies while lock is still acquired and does not release the lock before dying.)

### Linked list

To allow arbitary insertion we will implement a sorted linked list this implemention can be easily modifed to create a normal linked list/stack/queue. Let's assume following structure of Node in the linked list :-

```java
class Node {
    int value;
    Node next;
}
```

Let's consider an ordered linked list :-


```no-highlight
+----------+    +-------+    +-------+   +-------+    +--------+
|   head   ------   1   ------  3    -----   7   ------  tail  |
+----------+    +-------+    +-------+   +-------+    +--------+                                                           
```

In such a list insertion requires finding postion of the node i.e node after which this node should be inserted. Let's say we want to insert 5 in this list. It will require two operations making 7 as next node of 5 and then changing next node of 3 to 5.

Operation 1 :-

```no-highlight
+----------+    +-------+    +-------+           +-------+      +--------+
|   head   ------   1   ------  3    -------------   7   --------   tail |
+----------+    +-------+    +-------+          /+-------+      +--------+
                                               /                          
                                              /                           
                                             /                            
                                            /                             
                                  +-------+/                              
                                  |   5   /                               
                                  +-------+                                 
```

Operation 2 :-

```no-highlight
+----------+    +-------+    +-------+ +-------+   +-------+    +--------+
|   head   ------   1   ------  3    ---  5    ----- 7     ------   tail |
+----------+    +-------+    +-------+ +-------+   +-------+    +--------+
```

Operation 1 is safe and simple and can be done directly. for Operation 2 we will need to use Compare and swap to change next of 3. CAS will ensure that swap only suceeds if next node of 3 is still 5 in case another insertion has happened i.e 6 has been inserted in parallel during next CAS attempt we will mark 6 as next node 5. This will keep on going till CAS is not successful.

Similarly if we think of delete operation i.e we need to delete "5" then we need to find node before the node to delete which would be "3" and change the value of next to "7". We will need to ensure CAS for this operation too. Here CAS will ensure that next node of "3" remains "5" before we commit. Let's say while this deletion was happening their is another parallel insert between "5" and "7". In this case a single CAS while deletion will not be enough as it will be succesfull. So single CAS can not detect that a parallel insertion and deletion operation is happening at a node.

Algorithm proposeed in Tim harris paper handles this case using two CAS operaations first one to mark node which will be deleted and second CAS operation to actually change the next node. After first operation node is said to be logically deleted and after second operation node is said to be physically deleted.

Following is code for this.

```java
import java.util.concurrent.atomic.AtomicMarkableReference;

/**
 * Node represents single item in linked list. Here next pointer is atomic markable reference so that it can hold
 * additional boolean value if a node is logically deleted.
 *
 * one thing to note here is if next pointer is marked to true then it means current Node has been logically deleted.
 */
class Node {
    int value;
    AtomicMarkableReference<Node> next;

    public Node(int value) {
        this.value = value;
        this.next = new AtomicMarkableReference<>(null, false);
    }
}

/**
 * utility object used for performing search operation on the list. This object will not be exposed outside this class.
 * for a given search operation following would be values of 2 fields of this object :-
 * 1. curr - first node with equal or greater value than search keyword.
 * 2. pred - previous node of curr.
 */
class Window {
    public Node pred;
    public Node curr;

    Window(Node pred, Node curr) {
        this.pred = pred;
        this.curr = curr;
    }
}

public class LinkedList {

    Node head;
    Node tail;

    public LinkedList() {
        head = new Node(Integer.MIN_VALUE);
        tail = new Node(Integer.MAX_VALUE);
        head.next.set(tail, false);
    }

    /**
     * Add a new element to linked list.
     * @param value - value to add.
     * @return - return false in case value is already present in list in such case insertion will be ignored. Else
     * return true is value was inserted.
     */
    public boolean add(int value) {
        while (true) {
            final Window window = find(value);
            // return false if value is already present.
            if (window.curr.value == value) {
                return false;
            } else {
                final Node newNode = new Node(value);
                newNode.next = new AtomicMarkableReference<>(window.curr, false);

                // Try setting next node of pred using CAS operation. In case CAS operation fails we will need to find
                // new window.
                if (window.pred.next.compareAndSet(window.curr, newNode, false, false)) {
                    return true;
                }
            }
        }
    }

    /**
     * Remove an element from linked list.
     *
     * @param value - value to remove.
     * @return - return true in case value was present in list and remove. return false in case value was not present
     * in list.
     */
    public boolean remove(int value) {
        while (true) {
            final Window window = find(value);

            // in case value is not present in the list return false.
            if (window.curr.value != value) {
                return false;
            } else {
                final Node next = window.curr.next.getReference();

                // try to logically delete the node by marking delete bit to true.
                boolean logicallyDeleted = window.curr.next.attemptMark(next, true);

                // If logical deletion failed i.e someone already changed the next node try to search again.
                if (!logicallyDeleted) {
                    continue;
                }

                // physically delete the node. Even if this operation fails we are okay because we are physically
                // deleting the node during find operation anyways.
                window.pred.next.compareAndSet(window.curr, next, false, false);
                return true;
            }
        }
    }

    /**
     * utility method to print the list.
     */
    public void debug() {
        Node next = head.next.getReference();
        final StringBuilder message = new StringBuilder("head -> ");
        while (next != tail) {
            message.append(next.value ).append(" -> ");
            next = next.next.getReference();
        }
        System.out.println(message.append("tail").toString());
    }

    /**
     * Method to find first node whose value is greater than or equal to value. This will return object with two
     * properties :-
     *
     * 1. curr - This represents first node whose value is greater than or equal to input.
     * 2. pred - This is previous node of curr.
     *
     * With the following definition our search operation will always return some value as out tail is initialized with
     * MaxInt which will be always greater than any input.
     *
     * @param value - keyword to search
     * @return - window object.
     */
    private Window find(int value) {
        // In case list is empty.
        if (head.next.getReference() == tail) {
            return new Window(head, tail);
        }

        Node pred = null;
        Node current = null;
        Node succ = null;

        // default status of deleted bit.
        boolean[] deleteBit = {false};

        retry:
        while (true) {
            pred = head;
            current = pred.next.getReference();

            while (true) {
                // if successor is in logical deletion deleteBit will be set to "true" else it will remain "false".
                succ = current.next.get(deleteBit);

                // if node is logically deleted we should attempt to physically delete it.
                while (deleteBit[0]) {
                    boolean valueChanged = pred.next.compareAndSet(current, succ, false, false);
                    // If physical deletion failed this means another operation is happening in parallel. Start the loop
                    // again.
                    if (!valueChanged) {
                        continue retry;
                    }

                    // If physical deletion was successful change to next value.
                    current = succ;
                    succ = current.next.get(deleteBit);
                }

                if (current.value >= value) {
                    return new Window(pred, current);
                }

                // search next node
                pred = current;
                current = succ;
            }
        }
    }

}
```

I have tried adding as many comment as possible to make it easy to understand.


### References

1. https://timharris.uk/papers/2001-disc.pdf - Main paper upon which whole article is based (highly recommended)
1. https://www.cs.cmu.edu/~410-s05/lectures/L31_LockFree.pdf 
1. http://15418.courses.cs.cmu.edu/spring2013/article/46
1. https://en.wikipedia.org/wiki/Non-blocking_linked_list
1. https://en.wikipedia.org/wiki/Compare-and-swap

