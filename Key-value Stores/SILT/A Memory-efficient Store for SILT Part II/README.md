# A Memory-efficient Store for SILT: Part II
## Compact recursive representation
Trees are generally implemented using pointers. Pointers are required in nodes to store the address of their child nodes. Based on the size and architecture of the underlying hardware, these pointers may be up to eight bytes long. Every internal node will need at least one pointer—this means it isn’t memory-efficient.

We will represent our tries without using pointers. Our representation will essentially be a string containing our compact representation of the prefix tree. This is a recursive representation, meaning we can capture it using a recursive function. The below representation is for our memory-efficient store's in-memory index prefix tree T with left subtrie L and right subtrie R:

  Rep(T)=∣L∣Rep(L)Rep(R)
where ∣L∣ is the number of leaf nodes in L. Rep(T) is a special character to indicate an empty or leaf node T.

The rationale behind using only ∣L∣ is that during a key lookup, if we have to traverse to the right subtree, we will skip all the leaf nodes in the other subtree—the left subtree. Remember that our key-value entries in storage are sorted and stored sequentially, and we use the prefix tree to search our entries in storage. The prefix tree gives us an index. This index is the number of fixed-length entries we will skip to reach our key-value entry, represented by a leaf node in the tree. We will navigate our representation of a tree by counting the number of leaf nodes we have skipped to reach a leaf node. When we reach a leaf node in our representation, our count will give us the index—the number of entries we need to skip in storage to reach our key-value entry. We need to create a compact representation before searching for a key.

Here's our algorithm to create a representation of a trie in Python from an array of sorted keys.

```python
# K is an array of sorted keys from our illustration from the previous lesson
K = ["00001100",
     "00010010",
     "00100110",
     "00111110",
     "01000110",
     "10000001",
     "10101000",
     "11001101",
     "11101000",
     "11111001"]
# We may add our own array of keys by uncommenting the line below
# and adding strings of keys. Keys must be of fixed length and sorted
# K = ["00000000", ...]

# construct(K) gives compact representation of trie with keys from K
def construct(K):
    if len(K) == 0 or len(K) == 1: return("!")
    else:
        # If a key's MSB is 0 then it belongs in the left subtrie L
        L = [key[1:] for key in K if key[0] == "0"]
        # If a key's MSB is 1 then it belongs in the right subtrie R
        R = [key[1:] for key in K if key[0] == "1"]
        
        return str(len(L)) + construct(L) + construct(R)

print(construct(K))
```

```
The algorithm above constructs a compact recursive representation for an array of sorted keys. We've used Python strings and not bit-strings for simplicity. Clicking the "Run" button will generate a compact representation of the keys from our example in the illustration from our last lesson—also shown in the slide deck below. Source: "SILT: a memory-efficient, high-performance key-value store", 2011, In Proceedings of the Twenty-Third ACM Symposium on Operating Systems Principles (SOSP '11).
```

```
Note: For simplicity, we have used Python strings. This trie implementation does not allow for indexes greater than nine since it would require two characters. Our implementation should have an array of integers and special characters (or bits) for the leaf node indicator. The latter is to save memory—we only need to indicate a leaf node that does not require 32 bits.

The method to indicate an empty subtree or leaf node depends on the implementation. In our implementation, we have used Python strings for simplicity, and our indicator is the exclamation mark (!).
```

Running the code above without entering a different K (lines 2–11) yields the result 5421!!1!!!21!!1!1!!. The slide deck below shows how the algorithm provides this value.

[Algorithm]

5421!!1!!!21!!1!1!! is the compact recursive representation of the of the tree for keys from our example presented in the previous lesson. Let's look at its interpretation given below:

- 5: This is the value of ∣L∣ for the root node since five keys start with 0, represented by the five leaf nodes in its left subtrie.

- 4: This is the value of ∣L∣ for root's left child. The four leaf nodes in its left subtrie represent the four keys with 0 in their second position in this subtrie.

- 2: This is the value of ∣L∣ we get when we go into the left subtrie. There are two keys with 0 in their third position in this subtrie, represented by the two leaf nodes in the subtrie's descendants on the left side.

- 1!!: The next subtrie in our iteration is the bottom left internal node. The two exclamation marks represent its children nodes, the leaf nodes. We have now finished the left subtries in the root node's left subtrie—one key with 0 in its fourth position in this subtrie's left-side descendants.

- 1!!: The right subtrie where ∣L∣ equals 2—also the internal node with leaf nodes representing index 2 and 3 as child nodes (denoted by the exclamation marks at the end). One key with 0 in its fourth position in this subtrie's left-side descendants.

- 21!!1!1!!: This is the compact representation of the root's right subtrie.

While the actual degree of memory saved will vary from system to system, we can try to get an idea of how much memory was saved using this compact representation.

We have ten leaf nodes and eight internal nodes (the root is not an internal node, so we are not counting it). Uniquely identifying 18 nodes requires 5 bits—we can think of this as the memory used by a single pointer, and we need 18 of these. So the total number of bits consumed by pointers is 90 (18 nodes multiplied by 5 bits). Uniquely identifying ten indexes requires 4 bits, and we would need ten of these—these are our indexes stored in the leaf nodes—a total of 40 bits. In total, our in-memory index requires 130 bits without the compact representation.
```
Note: The reason we have not counted the prefix tree's root node when calculating the total number of bits required to represent a prefix tree is that even the compact representation would require at least one pointer to identify its address. For a prefix tree, this address is generally of the root node.
```
Our compact representation has required 19 pieces. There are 11 possible values that each piece can take, and it requires 4 bits. We need 19 of these. Hence, the total number of bits consumed is 76. We have saved 54 bits by using the compact representation!
```
Note: Pointers will require way more than 5 bits, and the pointer-based representation will require much more memory. However, using the compact representation will always save us a significant amount of memory.
```
Let's look at how we will look up keys in this compact representation.



## GET requests
We will process GET requests in the memory-efficient store to first search the key in the in-memory prefix tree for the key's index. We will then use the index to look up the key in storage. Here is our algorithm to search for a key K in the representation of a trie T.

```python
# K is a lookup key
# T is the compact representation of a trie

# lookup(K,T) gives the index of K in T
def lookup(K, T):
  (thead, ttail) = (T[0], T[1:])
  if thead == "!":
    return 0
  else:
    if K[0] == "0":
      # Recurse into the left subtrie
      return lookup(K[1:], ttail)
    else:
      # Skip the left subtrie
      ttail = discard_subtrie(ttail)
      # Recurse into the right subtrie
      return int(thead) + lookup(K[1:], ttail)

# discard 
def discard_subtrie(T):
  (thead, ttail) = (T[0], T[1:])
  if thead == "!":
    return ttail
  else:
    # Skip both subtries
    ttail = discard_subtrie(ttail)
    ttail = discard_subtrie(ttail)
    return ttail

print(lookup("01000110", "5421!!1!!!21!!1!1!!"))
```

```
Algorithm to search for a key in the compact representation of a prefix tree. Like in the previous algorithm, we have used Python strings for simplicity. Source: "SILT: a memory-efficient, high-performance key-value store", 2011, In Proceedings of the Twenty-Third ACM Symposium on Operating Systems Principles (SOSP '11).
```

The algorithm recursively navigates T and recurses into the left or right subtree based on K. We first break T into its head (T[0]) and tail (T[1:]). Here are our recursion cases:

1. The base case is simple. If the head is the special character to indicate a leaf node (exclamation mark in our case), the algorithm returns 0.

2. If the head does not indicate a leaf node, then we will proceed with the lookup in either of the subtries based on K:

    I. If the first bit of the K is 0, we will proceed with the lookup in the left subtrie with the remaining key K[1:]. The first character in the tail represents the left subtrie. We do not need to add anything to our return value since traversing left does not increase the index.

    II. If the first bit of K is 1, we will proceed with the lookup in the right subtrie with the remaining key K[1:]. We will need to drop the left subtrie to start lookup in the right subtrie—we will do this through our helper function (lines 20–28 in the code snippet above) that is explained below. When returning, we will add the head to our return value since we have skipped the left subtrie by recursing right. The head represents the number of leaf nodes in the left subtrie (∣L∣ from our representation in the previous section).

Our implementation has a helper function with an algorithm to drop the next left subtrie in our representation T. We first break T into its head (T[0]) and tail (T[1:]). Here are our recursion cases:

1. The base case is the same as the lookup algorithm. If the head is the special character to indicate a leaf node (an exclamation mark in our case), the algorithm returns 0.

2. If the head is not the special character, we call the recursive function twice to remove the entire left subtrie.

The slide deck below gives us a step-by-step understanding of how the above algorithm works.

[Algorithm]

We can see in the above slide deck (slide 3) that at the time of traversing to the right subtrie, we have added the value of ∣L∣ of the node at which we traverse right. The value of ∣L∣
 gives us the number of leaf nodes we skipped while traversing right—the number of nodes in the left subtrie.
 
