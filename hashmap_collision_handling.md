# HashMap Collision Handling: Complete Guide

## What is a Hash Collision?

A **hash collision** occurs when two different keys produce the same hash code or map to the same bucket index in the HashMap's internal array.

### Types of Collisions

1. **Hash Code Collision**: Different keys produce same `hashCode()`
2. **Bucket Index Collision**: Different hash codes map to same bucket index

```java
// Example of hash code collision
String key1 = "Aa";     // hashCode() = 2112
String key2 = "BB";     // hashCode() = 2112 (same!)

// Example of bucket index collision
// Even with different hash codes, they might map to same bucket
int hash1 = 1234;  // bucket index: 1234 & 15 = 2
int hash2 = 1250;  // bucket index: 1250 & 15 = 2 (same bucket!)
```

## Why Do Collisions Happen?

### Mathematical Inevitability
- **Pigeonhole Principle**: With infinite possible keys but finite buckets, collisions are inevitable
- **Birthday Paradox**: With only 23 people, there's 50% chance of shared birthdays
- In a 16-bucket HashMap, you need only ~5 elements for 50% collision probability

### Common Causes

1. **Poor Hash Functions**:
   ```java
   // Bad hash function - always returns same value
   public int hashCode() {
       return 1; // All objects collide!
   }
   ```

2. **Limited Bucket Space**:
   ```java
   // Small table with many elements
   HashMap<String, String> map = new HashMap<>(4); // Only 4 buckets
   // Adding 10 elements will definitely cause collisions
   ```

3. **Similar Keys**:
   ```java
   // Similar strings often have similar hash codes
   map.put("user123", "John");
   map.put("user124", "Jane");  // Likely to collide
   ```

## Collision Resolution Evolution

### Java 7 and Earlier: Separate Chaining with Linked Lists

```java
// Simplified representation
class HashMap<K,V> {
    Entry<K,V>[] table;
    
    static class Entry<K,V> {
        K key;
        V value;
        Entry<K,V> next;  // Points to next entry in collision chain
        int hash;
    }
}
```

**Collision Resolution Process**:
1. Calculate bucket index: `index = hash & (table.length - 1)`
2. If bucket empty → insert new entry
3. If bucket occupied → traverse linked list
4. If key found → update value
5. If key not found → add to head of list (Java 7)

**Problems with Java 7 Approach**:
- **O(n) worst-case performance**: Long collision chains
- **Head insertion**: Caused infinite loops during resize in concurrent scenarios
- **Vulnerability to DoS attacks**: Malicious keys could create long chains

### Java 8+: Hybrid Approach (Lists + Red-Black Trees)

```java
// Node types in Java 8+
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
}

static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // Maintains insertion order
    boolean red;           // Red-black tree coloring
}
```

## Detailed Collision Resolution Process

### Step-by-Step Collision Handling in Java 8+

```java
public V put(K key, V value) {
    // 1. Calculate hash
    int hash = hash(key);
    
    // 2. Find bucket index
    int index = (table.length - 1) & hash;
    
    Node<K,V> first = table[index];
    
    if (first == null) {
        // No collision - direct insertion
        table[index] = new Node<>(hash, key, value, null);
    } else {
        // Collision detected - resolve it
        handleCollision(hash, key, value, first, index);
    }
}

private void handleCollision(int hash, K key, V value, Node<K,V> first, int index) {
    // Check if first node matches
    if (first.hash == hash && Objects.equals(first.key, key)) {
        // Key already exists - update value
        first.value = value;
        return;
    }
    
    if (first instanceof TreeNode) {
        // Collision chain is already a tree
        ((TreeNode<K,V>) first).putTreeVal(this, table, hash, key, value);
    } else {
        // Collision chain is still a linked list
        handleLinkedListCollision(hash, key, value, first, index);
    }
}

private void handleLinkedListCollision(int hash, K key, V value, Node<K,V> first, int index) {
    Node<K,V> current = first;
    int chainLength = 0;
    
    // Traverse the collision chain
    while (true) {
        chainLength++;
        
        if (current.next == null) {
            // End of chain - add new node (tail insertion in Java 8+)
            current.next = new Node<>(hash, key, value, null);
            
            // Check if we need to treeify
            if (chainLength >= TREEIFY_THRESHOLD) {
                treeifyBin(table, hash);
            }
            break;
        }
        
        current = current.next;
        
        // Check if key already exists
        if (current.hash == hash && Objects.equals(current.key, key)) {
            current.value = value;
            break;
        }
    }
}
```

### Tree-identification Process

**When does Tree-identification occur?**
1. Collision chain length ≥ 8 (`TREEIFY_THRESHOLD`)
2. Table capacity ≥ 64 (`MIN_TREEIFY_CAPACITY`)
3. If table < 64, resize instead of tree-ify

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n = tab.length;
    int index = (n - 1) & hash;
    
    if (tab == null || n < MIN_TREEIFY_CAPACITY) {
        resize(); // Resize instead of treeify
        return;
    }
    
    Node<K,V> first = tab[index];
    if (first != null) {
        // Convert linked list to red-black tree
        TreeNode<K,V> root = null;
        TreeNode<K,V> current = null;
        
        // Step 1: Convert all nodes to TreeNodes
        do {
            TreeNode<K,V> treeNode = replacementTreeNode(first, null);
            if (current == null) {
                root = treeNode;
            } else {
                current.next = treeNode;
                treeNode.prev = current;
            }
            current = treeNode;
        } while ((first = first.next) != null);
        
        // Step 2: Build balanced red-black tree
        tab[index] = root;
        root.treeify(tab);
    }
}
```

### UnTree-identification Process

**When does unTree-identification occur?**
- During resize when tree size ≤ 6 (`UNTREEIFY_THRESHOLD`)
- Prevents constant switching between tree and list

```java
final Node<K,V> untreeify(HashMap<K,V> map) {
    Node<K,V> head = null, tail = null;
    
    // Convert tree nodes back to regular nodes
    for (TreeNode<K,V> treeNode = this; treeNode != null; treeNode = treeNode.next) {
        Node<K,V> node = map.replacementNode(treeNode, null);
        if (tail == null) {
            head = node;
        } else {
            tail.next = node;
        }
        tail = node;
    }
    return head;
}
```

## Collision Performance Analysis

### Performance Comparison

| Scenario | Java 7 | Java 8+ (List) | Java 8+ (Tree) |
|----------|---------|----------------|----------------|
| No Collision | O(1) | O(1) | O(1) |
| Few Collisions (≤8) | O(n) | O(n) | O(n) |
| Many Collisions (>8) | O(n) | O(n) | O(log n) |

### Real-World Impact

```java
// Worst-case scenario in Java 7
Map<String, String> map = new HashMap<>();

// These strings all have same hash code
map.put("Aa", "value1");     // hashCode = 2112
map.put("BB", "value2");     // hashCode = 2112
map.put("C#", "value3");     // hashCode = 2112
// ... adding more similar keys creates O(n) chain

// In Java 7: Each get/put becomes O(n) operation
// In Java 8+: After 8 collisions, becomes O(log n) operation
```

## Hash Function Improvements

### Java 8+ Hash Perturbation

```java
// Java 7 and earlier
static int hash(int h) {
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

// Java 8+
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**Why the change?**
- **Simpler and faster**: Single XOR operation vs multiple operations
- **Better distribution**: Mixes high and low bits effectively
- **Reduced collisions**: Especially for poor hash functions

### Hash Distribution Example

```java
// Without perturbation
String key1 = "ABC";  // hashCode = 64578
String key2 = "DEF";  // hashCode = 68165
// Bucket indices might be similar

// With perturbation (h ^ (h >>> 16))
int hash1 = 64578 ^ (64578 >>> 16) = 64577;
int hash2 = 68165 ^ (68165 >>> 16) = 68164;
// Better distribution across buckets
```

## Collision Attack Prevention

### Hash-Flooding DoS Attack

**The Problem** (Pre-Java 8):
```java
// Attacker sends specially crafted keys
// All keys hash to same bucket
POST /api/data
{
  "Aa": "value1",
  "BB": "value2",
  "C#": "value3",
  // ... 1000 more colliding keys
}

// Server's HashMap becomes O(n²) for operations
// Server becomes unresponsive
```

**The Solution** (Java 8+):
- Tree-identification limits worst-case to O(log n)
- Attack impact reduced from O(n²) to O(n log n)
- Hash perturbation makes collision prediction harder

### Alternative Hash in Extreme Cases

```java
// Java 8 can use alternative hash functions
// when string-based collision attacks are detected
final int hash32() {
    // Alternative hash calculation
    // Used in extreme collision scenarios
}
```

## Collision Metrics and Monitoring

### Measuring Collision Rate

```java
public class HashMapCollisionAnalyzer<K,V> {
    private HashMap<K,V> map;
    
    public CollisionStats analyzeCollisions() {
        // Access internal table (via reflection in real scenario)
        Object[] table = getInternalTable();
        
        int totalBuckets = table.length;
        int occupiedBuckets = 0;
        int totalChainLength = 0;
        int maxChainLength = 0;
        int treeifiedBuckets = 0;
        
        for (Object bucket : table) {
            if (bucket != null) {
                occupiedBuckets++;
                int chainLength = calculateChainLength(bucket);
                totalChainLength += chainLength;
                maxChainLength = Math.max(maxChainLength, chainLength);
                
                if (isTreeNode(bucket)) {
                    treeifiedBuckets++;
                }
            }
        }
        
        return new CollisionStats(
            totalBuckets,
            occupiedBuckets,
            (double) totalChainLength / occupiedBuckets, // Average chain length
            maxChainLength,
            treeifiedBuckets
        );
    }
}
```

### Ideal Collision Metrics

- **Load Factor**: ~0.75 (default)
- **Average Chain Length**: ~1.0 (no collisions)
- **Max Chain Length**: <8 (avoid Tree-identification)
- **Bucket Utilization**: ~75% of buckets occupied

## Best Practices for Collision Avoidance

### 1. Implement Good Hash Functions

```java
// Good hash function example
public class Person implements Equals, Hashable {
    private String name;
    private int age;
    private String email;
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age, email); // Uses multiple fields
    }
    
    // Bad hash function
    public int badHashCode() {
        return name.length(); // Too many collisions
    }
}
```

### 2. Choose Appropriate Initial Capacity

```java
// If you know you'll have ~1000 elements
// Set initial capacity to avoid resize overhead
HashMap<String, String> map = new HashMap<>(1000 / 0.75); // ~1333

// Avoid too small initial capacity
HashMap<String, String> bad = new HashMap<>(1); // Will resize many times
```

### 3. Consider Load Factor Trade-offs

```java
// Lower load factor = fewer collisions, more memory
HashMap<String, String> sparse = new HashMap<>(16, 0.5f);

// Higher load factor = more collisions, less memory
HashMap<String, String> dense = new HashMap<>(16, 0.9f);

// Default 0.75 is usually optimal
HashMap<String, String> optimal = new HashMap<>();
```

### 4. Monitor and Profile

```java
// Use profiling tools to detect collision hotspots
// JProfiler, YourKit, or built-in JVM tools
// Look for:
// - High HashMap operation times
// - Uneven bucket distribution
// - Frequent Tree-identification
```

## Collision Resolution Summary

**Java 8+ collision handling is a masterpiece of engineering**:

1. **Backwards Compatible**: Existing code works unchanged
2. **Performance Resilient**: O(log n) worst-case prevents DoS attacks
3. **Memory Efficient**: Only tree-ifies when necessary
4. **Self-Optimizing**: Automatically adapts to collision patterns

The evolution from simple linked lists to red-black trees represents one of the most significant improvements in Java Collections Framework history, making HashMap both faster and more secure.