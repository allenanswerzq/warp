# SumTree — The Core Data Structure

> Source: `crates/sum_tree/src/`
> A B-tree with cached summaries. O(log n) insert, delete, and aggregate queries.
> Used for: editor buffer, block list, terminal scrollback.

---

## What Problem It Solves

```
Plain Vec for a text buffer:
  Insert at line 500 of 10,000 lines → move 9,500 items → O(n)
  "How many bytes before line 500?" → sum 500 items    → O(n)

SumTree:
  Insert at line 500 → update ~4 nodes  → O(log n)
  "Bytes before line 500?" → walk 4 nodes → O(log n)
  Total line count → read root summary   → O(1)
```

---

## Structure: B-Tree + Cached Summaries

```
                  [summary: 1000 bytes, 50 lines]        ← root
                 /                                \
    [400 bytes, 20 lines]           [600 bytes, 30 lines]  ← internal
      /      |       \                /      |       \
  [items] [items] [items]       [items] [items]  [items]    ← leaves

Each leaf: up to 6 items (TREE_BASE = 6)
Each node: caches summary of ALL items below it
Summary = accumulated via += (bytes + bytes, lines + lines)
```

---

## Three Key Traits

```rust
// 1. Item: what you store
trait Item {
    type Summary: AddAssign + Default;
    fn summary(&self) -> Summary;
}

// 2. Summary: aggregated metadata (must support +=)
struct TextSummary { bytes: usize, lines: usize, chars: usize }

// 3. Dimension: one axis you can seek by
trait Dimension<Summary> {
    fn add_summary(&mut self, summary: &Summary);
}
// ByteOffset, LineCount, CharOffset are all Dimensions
```

---

## Cursor: Seek by Any Dimension

```
"Find line 500":

Cursor at root: total = 50 lines
  left child: 20 lines → not enough, count and skip
  right child: 30 lines → line 500 is here
    left grandchild: 10 lines → skip
    middle: 15 lines → found it!

Visited ~4 nodes instead of scanning 500 items.
```

```rust
let mut cursor = tree.cursor::<LineCount>();
cursor.seek(&LineCount(500), SeekBias::Left);
// now points to the item at line 500
// also knows: bytes_before, chars_before (from summaries)
```

---

## Where It's Used in Warp

```
SumTree<TextChunk>    → editor buffer (text with byte/line/char summaries)
SumTree<BlockHeight>  → block list (scroll position by pixel height)
SumTree<GridRow>      → terminal scrollback storage
```

One data structure powers the editor, terminal, AND block list.

---

## Performance

```
Operation               Vec/String    SumTree
───────────             ──────────    ───────
Insert at position      O(n)          O(log n)
Delete range            O(n)          O(log n)
"Bytes before line N"   O(n)          O(log n)
"Which line at byte N"  O(n)          O(log n)
Total line count        O(n)          O(1) ← root summary

100K line file:
  Vec: ~100,000 ops per edit
  SumTree: ~7 ops per edit (log₆ 100K ≈ 6.4)
```

---

## Copy-on-Write (Arc)

```
SumTree(Arc<Node<T>>)   ← nodes are reference-counted

Editing only creates new nodes on the path from root to leaf.
All other nodes are shared (Arc pointer copy).
→ cheap snapshots, concurrent reads
```

---

## vs Rope (Zed)

```
                    SumTree (Warp)         Rope (Zed)
Structure           B-tree (wide)          Binary tree (narrow)
Branching           6 children/node        2 children/node
Depth (100K items)  ~7 levels              ~17 levels
Leaf content        ANY type (generic)     strings only
Used for            text + blocks + grid   text only

Rope = specialized for text → simpler, slightly faster for strings
SumTree = generic → one structure for text, blocks, terminal
```
