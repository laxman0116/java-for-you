# HashMap Internal Flow: Java 8 to 17 Deep Dive

## Table of Contents
1. [Core Architecture Changes](#core-architecture-changes)
2. [Internal Data Structure Evolution](#internal-data-structure-evolution)
3. [Hash Function and Bucket Distribution](#hash-function-and-bucket-distribution)
4. [Tree-identification Process (Java 8+)](#tree-identification-process-java-8)
5. [Put Operation Deep Flow](#put-operation-deep-flow)
6. [Get Operation Deep Flow](#get-operation-deep-flow)
7. [Resize Mechanism](#resize-mechanism)
8. [Memory Layout and Node Types](#memory-layout-and-node-types)
9. [Performance Optimizations](#performance-optimizations)
10. [Version-Specific Changes](#version-specific-changes)

## Core Architecture Changes

### Java 7 vs Java 8+ Fundamental Shift

**Java 7 and Earlier:**
- Pure array + linked list structure
- O(n) worst-case performance for collision chains
- Head insertion in collision chains (caused infinite loops during resize)

**Java 8+:**
- Hybrid array + linked list + red-black tree structure
- O(log n) worst-case performance when tree-ified
- Tail insertion in collision chains
- Dynamic switching between list and tree based on threshold

## Internal Data Structure Evolution

### Node Types Hierarchy

```java
// Base Node (Java 8+)
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}

// Tree Node (extends LinkedHashMap.Entry)
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
}
```

### Internal Fields

```java
public class HashMap<K,V> {
    transient Node<K,V>[] table;           // The bucket array
    transient Set<Map.Entry<K,V>> entrySet; // Cache for entrySet()
    transient int size;                     // Number of key-value pairs
    transient int modCount;                 // Modification counter
    int threshold;                          // Next size to resize
    final float loadFactor;                 // Load factor (default 0.75)
    
    // Constants
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 16
    static final int MAXIMUM_CAPACITY = 1 << 30;
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final int TREEIFY_THRESHOLD = 8;
    static final int UNTREEIFY_THRESHOLD = 6;
    static final int MIN_TREEIFY_CAPACITY = 64;
}
```

## Hash Function and Bucket Distribution

### Hash Calculation Process

```java
// Step 1: Object's hashCode()
int h = key.hashCode();

// Step 2: Hash perturbation (Java 8+)
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// Step 3: Bucket index calculation
int index = (n - 1) & hash;  // n is table.length
```

### Why Hash Perturbation?

The `h ^ (h >>> 16)` operation:
- Mixes higher bits with lower bits
- Reduces collision probability
- Particularly effective when table size is small
- Preserves speed (single XOR operation)

### Bucket Index Calculation

Using `(n-1) & hash` instead of `hash % n`:
- Requires table size to be power of 2
- Bitwise AND is faster than modulo
- `n-1` creates a mask that preserves lower bits
- Example: if n=16, then n-1=15 (binary: 1111)

## Tree-identification Process (Java 8+)

### When Tree-identification Occurs

1. **Threshold Conditions:**
   - Collision chain length ≥ 8 (`TREEIFY_THRESHOLD`)
   - Table capacity ≥ 64 (`MIN_TREEIFY_CAPACITY`)
   - If table < 64, resize instead of tree-ify

2. **Tree-identification Process:**
   ```java
   final void treeifyBin(Node<K,V>[] tab, int hash) {
       // Convert linked list to red-black tree
       // Maintain insertion order via prev/next pointers
       // Balance tree for O(log n) operations
   }
   ```

### When UnTree-identification Occurs

- During resize when tree size ≤ 6 (`UNTREEIFY_THRESHOLD`)
- Converts red-black tree back to linked list
- Hysteresis prevents constant switching

### Red-Black Tree Properties

1. Every node is red or black
2. Root is always black
3. Red nodes have black children
4. All paths from root to leaves have same number of black nodes
5. Maintains O(log n) height guarantee
![image](https://github.com/user-attachments/assets/599ed25d-8149-46a2-91ea-db9a9c74ada1)

## Put Operation Deep Flow

### Complete Put Process

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // Step 1: Initialize table if empty
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // Step 2: Check if bucket is empty
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        
        // Step 3: Check if first node matches
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        
        // Step 4: Handle tree node
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        
        // Step 5: Handle linked list
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    
                    // Tree-ify if threshold reached
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                
                // Found matching key
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // Step 6: Update existing value
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    
    // Step 7: Increment modification count and size
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### Key Decision Points in Put

1. **Hash Collision Resolution:**
   - Same hash, same key → Update value
   - Same hash, different key → Add to collision chain/tree

2. **Tree vs List Decision:**
   - Collision chain length ≥ 8 → Consider Tree-identification
   - Table size < 64 → Resize instead of tree-ify
   - Table size ≥ 64 → Tree-ify the bucket

3. **Resize Trigger:**
   - Size exceeds threshold (capacity × load factor)

## Get Operation Deep Flow

### Complete Get Process

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    
    // Step 1: Check if table exists and bucket is not empty
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        
        // Step 2: Check first node (most common case)
        if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        
        // Step 3: Search in collision chain/tree
        if ((e = first.next) != null) {
            // Tree search - O(log n)
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            
            // Linear search - O(n)
            do {
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

### Performance Characteristics

- **Best Case:** O(1) - Key found in first position
- **Average Case:** O(1) - Good hash distribution
- **Worst Case (Java 7):** O(n) - All keys hash to same bucket
- **Worst Case (Java 8+):** O(log n) - Tree-ified bucket

## Resize Mechanism

### Resize Trigger Conditions

1. **Initial resize:** Table is null or empty
2. **Load factor exceeded:** size > threshold
3. **Tree-identification requirement:** Need capacity ≥ 64 for Tree-identification

### Resize Process Details

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    
    // Step 1: Calculate new capacity and threshold
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // ... handle other cases ...
    
    // Step 2: Create new table
    threshold = newThr;
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    
    // Step 3: Transfer existing entries
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                
                // Single node
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                
                // Tree node
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                
                // Linked list
                else {
                    // Preserve order and optimize placement
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    
                    do {
                        next = e.next;
                        // Clever bit manipulation
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null) loHead = e;
                            else loTail.next = e;
                            loTail = e;
                        } else {
                            if (hiTail == null) hiHead = e;
                            else hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    
                    // Place chains in new table
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

### Clever Resize Optimization

The key insight: `(e.hash & oldCap) == 0`

- When capacity doubles, each element either:
  - Stays at same index: `newIndex = oldIndex`
  - Moves by oldCap: `newIndex = oldIndex + oldCap`
- This eliminates need to recalculate hash for every element
- Preserves relative order within collision chains

## Memory Layout and Node Types

### Node Memory Overhead

**Regular Node:**
- Object header: 12-16 bytes
- int hash: 4 bytes
- K key: 8 bytes (reference)
- V value: 8 bytes (reference)
- Node<K,V> next: 8 bytes (reference)
- **Total: ~40-48 bytes per node**

**TreeNode:**
- Inherits from LinkedHashMap.Entry
- Additional tree pointers: parent, left, right, prev
- Boolean red flag
- **Total: ~80-96 bytes per node**

### Table Memory Usage

- Array of Node references: `capacity × 8 bytes`
- Default initial capacity: 16 × 8 = 128 bytes
- At load factor 0.75 with 12 elements: resize to 32 × 8 = 256 bytes

## Performance Optimizations

### Java 8+ Optimizations

1. **Tree-identification:**
   - Worst-case O(log n) instead of O(n)
   - Red-black tree maintains balance

2. **Improved Hash Function:**
   - Better bit mixing reduces collisions
   - Particularly effective for poor hash functions

3. **Resize Optimization:**
   - Clever bit manipulation for rehashing
   - Preserves order, reduces computation

4. **Tail Insertion:**
   - Prevents infinite loops during concurrent resize
   - More predictable behavior

### Java 9+ Optimizations

1. **Compact Strings Impact:**
   - String keys use less memory
   - Better cache locality

2. **JEP 180 (Java 9):**
   - Handle frequent HashMap collision patterns
   - Better hash code distribution

### Load Factor Considerations

**Default 0.75 balances:**
- **Space efficiency:** Higher load factor = more space utilization
- **Time complexity:** Lower load factor = fewer collisions
- **Resize frequency:** Affects performance due to rehashing cost

**Load Factor Impact:**
- 0.5: Fewer collisions, more memory waste
- 0.75: Good balance (default)
- 1.0: Maximum space utilization, more collisions

## Version-Specific Changes

### Java 8
- **Major:** Tree-identification with red-black trees
- **Major:** Improved hash function
- **Major:** Tail insertion in collision chains
- Functional interfaces support (forEach, compute, etc.)

### Java 9
- **Minor:** String compaction affects String keys
- Factory methods (`Map.of()`)
- Improved hash collision handling

### Java 10
- **Minor:** Local variable type inference support
- Performance improvements in resize

### Java 11 (LTS)
- **Minor:** General performance optimizations
- Better escape analysis for local HashMap instances

### Java 12-17
- **Minor:** Incremental performance improvements
- Better JIT compilation optimizations
- Switch expressions support (affects usage patterns)

### Key Behavioral Guarantees

1. **No ordering guarantee** (except insertion order in collision chains)
2. **Fail-fast iterators** (throw ConcurrentModificationException)
3. **Null key/value support** (single null key, multiple null values)
4. **Non-synchronized** (requires external synchronization or ConcurrentHashMap)

## Performance Characteristics Summary

| Operation | Average | Worst Case (Java 7) | Worst Case (Java 8+) |
|-----------|---------|--------------------|--------------------|
| Get       | O(1)    | O(n)              | O(log n)          |
| Put       | O(1)    | O(n)              | O(log n)          |
| Remove    | O(1)    | O(n)              | O(log n)          |
| Resize    | O(n)    | O(n)              | O(n)              |

The transformation from Java 7 to Java 8+ represents one of the most significant internal improvements in Java Collections Framework history, fundamentally changing HashMap's worst-case performance characteristics while maintaining backward compatibility.
