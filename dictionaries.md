# Dictionaries in software development — small notes

A **dictionary** (also: map, hash map, associative array, object) stores **values looked up by keys**, not by position.

```
key  ──►  value
"alice" ► 42
"bob"   ► 7
```

That is the same idea as “a key/value store in memory is a database.” For a lot of programs it **is** the right one.

---

## What you actually get

| Idea | Meaning |
|------|--------|
| **Key** | Unique label you search with (string, int, id…) |
| **Value** | Anything: number, object, list, even another dict |
| **Lookup** | “Give me the value for this key” — usually **O(1) average** with a hash map |
| **Insert / update** | Same slot: first time insert, later overwrite |
| **Delete** | Remove the pair |
| **Missing key** | Error, `null`/`None`, or a default — **you must decide** |

Typical names: Python `dict`, JS `Map` / `{}`, Java `HashMap`, C# `Dictionary`, Go `map`, Rust `HashMap`.

---

## Mental model (hash map)

```
key  →  hash(key)  →  bucket  →  (key, value)
```

- Same key always hashes the same way → fast find.  
- Different keys can collide → bucket holds a short list/tree.  
- **Keys must be hashable/immutable** (don’t use a list you then mutate as a key).

Order: **do not assume insertion order** unless the language documents it (Python 3.7+ dicts do; many hash maps do not).

---

## When to use one

- Count things: `word → count`  
- Index by id: `user_id → User`  
- Config / env: `name → setting`  
- Cache: `key → computed result`  
- Group: `category → list of items`  
- Graph-ish: `node → neighbors`

**Not** for: “item 0, 1, 2…” (use a list/array), or “keep sorted by key” unless you pick a **tree map** / sorted dict.

---

## Gotchas

1. **Missing keys** — crash vs silent default.  
2. **Mutable keys** — hash changes, entry vanishes.  
3. **Nested mutation** — two variables point at the **same** dict.  
4. **JSON objects** are dictionaries; keys are strings.  
5. **Iteration while updating** — often undefined / error.  
6. **Memory** — millions of keys is still RAM; that’s when a real DB starts.

Persist: dump to JSON/file on exit, load on start — still a dictionary, now durable. It stops being enough when data > memory, many writers, or crash mid-write (the three DB constraints).

---

# Questions + memos

**Q1.** Array vs dictionary: you need “user 1842’s email.” Which, and why?  
**Memo:** Dictionary: key is the id. An array would force scan or a huge holey list.

**Q2.** `counts["a"] += 1` when `"a"` was never inserted. What should you do?  
**Memo:** Missing-key policy. Use `get("a", 0)`, `defaultdict`, or `putIfAbsent` — don’t assume the key exists.

**Q3.** Why can’t a typical hash map use a mutable list as a key?  
**Memo:** Hash is computed from key contents. Mutate the list → lookup goes to the wrong bucket.

**Q4.** Two threads `d[k] = d[k] + 1` with no lock. What’s wrong?  
**Memo:** Read-modify-write race. Dicts are usually **not** a concurrency story; you need a lock, atomic map, or a DB with transactions.

**Q5.** You `json.dump` a dict of 10M users on every request. Why is that a bad “database”?  
**Memo:** Fine as a cache/config. Pain when bigger than RAM, concurrent writes, or power fails mid-write — then you want pages, WAL, MVCC, not a naked dict.

**Q6.** `for k in d: del d[k]` — safe?  
**Memo:** Usually **no**. Iterate a **copy** of keys, or collect then delete.

**Q7.** When would you pick a **tree map** (sorted keys) over a hash map?  
**Memo:** You need ranges (“all keys from M to P”) or ordered iteration. Hash maps are for **point** lookup.

**Q8.** Is `{"id": 1, "id": 2}` one pair or two?  
**Memo:** Keys are unique. Duplicate key → **one** entry; last write wins (JSON parsers may reject or keep one).