# Fluent Python Post-reading Notes

## Surprised Me (cross-chapter intuition log)

> Things that violated my intuition. These are the highest-value review items — they often reveal themes that recur across chapters (e.g. reference-vs-value: Ch2 tuple unpacking → Ch4 `memoryview`). Add one line whenever something surprises you; note the chapter so you can trace the thread.

- _(Ch2)_ example: unpacking a tuple copies **references**, so mutating an unpacked mutable also mutates the tuple's element.

---

## Chapter 1 The Python Data Model
1. TODO
## Chapter 2 An Array of Sequences
1. Sequence: container vs flat

    - Container sequence, such as a list and a tuple, stores references (pointers) of objects it contains, which can be of **different** types and are often scattered in memory.

        - This also means that when unpacking a container sequence e.g. a tuple, it simply copies the reference to variables so mutating those varabiles (unless immutable) will also update the container sequence. This is why generally storing mutable objects in a tuple (which is immutable) is not a good idear!
    
    - Flat sequence stores the **value** of its content in its own memory, not as separate Python objects. It can only hold primitive values of **one** type. Examples are `str`, `bytes` and `array.array`.

    So the real distinction is what's stored in memory! Flat sequence memory is much more compact as all its contents are stored continuously.

    To check memory size, use `sys.getsizeof()` but when used on a `list` it will only check the memory of object references stored in the list not the actual size of objects pointed by the object references. Use `sum(sys.getsizeof(f) for f in l)`

1. Loop variables in listcomp leaked into the enclosing scope (called varaible leakage clobbering) whereas in Python 3 listcomp has their own scope. Genexps always had their own scope in both Python 2 and 3.

1. How slicing `s[start:stop:step]` works:

    The slice selects indices starting at start, advancing by step, stopping before stop. The start index is always included (it's where you begin), and the stop index is always excluded (it's where you stop). The step just controls direction and stride.

    Negative stepper means starting backward. `s[::-2]` where `len(s) = 10` means starting from index 10, then 8, 6, 4, 2, 0.

    And `s[2:9:3]` means starting index 2, then 5, and finally 8 because 11 > 9!.

1. Bisect

    1. bisect.bisect(haystack, needle) (aka bisect_right) — returns the index where needle would be inserted to keep the sequence sorted. Useful for **lookups** (e.g., mapping a score to a grade).
    
    1. bisect.insort(seq, item) — actually inserts the item in sorted position, keeping the list ordered without a full re-sort.

1. When list is **not** an answer

    1. array.array — when you have millions of numbers of a single type; far more memory-efficient than a list of floats/ints (this is the flat sequence from point 1).

    1. collections.deque — for fast appends/pops at both ends (a list is O(n) for insert(0, x) / pop(0)).
    
    1. memoryview (share memory between array-like structures without copying) and numpy.ndarray (numeric arrays).

1. In sorting, make the key func return a tuple so that it can break a tie when the sorting based on the first element(s)! For instance, sorting a list of fruits based on its length and break the tie alphalically: `sorted(fruits, key=lambda fruit: (len(fruit), fruit))`

## Chapter 3 Dictionaries and Sets
- Unpacking Mappings
    - In a dict literal it's ok to unpack multiple dicts with the same key and the latter one wins e.g. `a={'x':1, 'y'=2} b={'x': 3} c = {**a, **b}`; and c is now `{'x':3, 'y':2}`. But in a function call all unpacked keys must be **string and unique** as Python doesn't allow duplicate keyword arguments and Python can't bind non-string arguemnt as a keyword name in a function call! E.g. this raises `TypeError: keywords must be strings`: `c = {1: 'a', 2:'b'}, dump(**c)`
- Merging Mappings with `|`
    - The type of `d1 | d2` will follow the type of `d1`, this is a CPython memory optimization to avoid shifting hash table buckets unnecessarily; this matters when subclassing `d1`.
    - Python will recognize keys with the same hash as duplicates e.g. int 1 and float 1.0.

- Pattern Matching with Mapping
    - `match/case` can match mapping subjects. A few critical rules from the chapter:

        **1. Partial matches succeed.**
        A case like `{'type': 'book'}` will match even if the subject has extra keys like `'title'` or `'pages'`. This is the opposite of sequence patterns, which require the **length** to match. This also means that the order of case statements matters as it can determine which target can be matched first!

        **2. Key order in the pattern is irrelevant.**
        `{'api': 2, 'type': 'book'}` matches the same subjects as `{'type': 'book', 'api': 2}`.

        **3. Capture remaining keys with `**rest`.**
        You can add one `**variable` at the end to catch all unmatched keys as a dict. `**_` is explicitly forbidden (it would be redundant).

        **4. `__missing__` is NOT triggered.**
        Pattern matching uses `d.get(key, sentinel)` internally — not `d[key]` — so `defaultdict` auto-creation never fires during a match.

        **5. Patterns match any `collections.abc.Mapping` subclass**, not just `dict` — so `OrderedDict` works too.

- Standard API of Mapping Types

    - The `collections.abc` module defines two ABCs that describe the mapping interface:

    - **`Mapping`** — read-only interface (`__getitem__`, `__len__`, `__iter__`, plus derived methods like `get`, `keys`, `values`, `items`)
    - **`MutableMapping`** — adds write operations (`__setitem__`, `__delitem__`, plus `pop`, `update`, `setdefault`, etc.)

        **Why this matters in practice:**
        The chapter says to prefer `isinstance(x, abc.Mapping)` over `isinstance(x, dict)` when writing functions that accept any mapping. Checking for `dict` rejects valid alternatives like `OrderedDict`, `defaultdict`, or user-defined mappings.

        **Building custom mappings:**
        The chapter advises extending `collections.UserDict` (not subclassing `dict` or the ABCs directly). `dict` is a C built-in with implementation shortcuts that can bypass your overridden Python methods in subtle ways. `UserDict` stores items in an internal `dict` called `.data`, which avoids tricky recursion problems when overriding `__setitem__` and friends.

- What is Hashable

    - An object is **hashable** if:
    
        1. It has a `__hash__()` method that returns a hash code that **never changes** during its lifetime, AND
        2. It has an `__eq__()` method to compare with other objects.

    - Additionally: **hashable objects that compare equal must have the same hash code.**

        **Rules for built-in types:**
        - Numeric types, `str`, `bytes` — always hashable (flat and immutable)
        - `frozenset` — always hashable (every element it contains must be hashable by definition)
        - `tuple` — hashable **only if all its items are hashable**
        - `list`, `set`, `dict` — never hashable (mutable)

        **User-defined types** are hashable by default: their hash code is `id(self)`, and the inherited `__eq__` compares object identities. The moment you define a custom `__eq__` that looks at internal state, you must also define a consistent `__hash__` — otherwise Python sets `__hash__` to `None` and the object becomes unhashable.

        **Note:** hash codes may differ across Python processes due to a security salt — they are only guaranteed constant within one process.

        - Note that flat sequences like `str`, `array.array` and `bytes` store their data inline - no references. Their hash is computed from their own bytes. Container sequences like tuple and list store references to other objects so a tuple's hashability is recursive - it must ask each referenced object for its hash. That's why a tuple is **not unconditionally** hashable: it delegates to its contents!

            - flat + immutable  →  always hashable     (`str`, `bytes`, `int`)
            - flat + mutable    →  never hashable      (`bytearray`, `array`)
            - container         →  conditionally hashable, depends on contents  (`tuple`)
            - container + mutable → never hashable    (`list`)

- Inserting or Updating Mutable Values?

    For a dict with mutable values e.g. a list, the native **`if/in`** approach takes 2 or 3 lookups depending on whether the key exists or is missing. `setdefault` is better here as `d.setdefault(key, default)` does two things atomically in one lookup:

        1. If the key doesn't exist, it will insert the default and return the default
        2. If the key exists, it will return the current value e.g. default is ignored and dict is unchanged.
    
    Because it returns the value, a method can be chained to the `setdefault` call directly e.g. `d.setdefault(k, []).append(x)` and no lookup needed.

    **NOTE:** `d.get(key, default)` looks similar but not equivalent; it returns the fallback (`default`) **without** inserting it to the dict when key is missing; so normally another lookup is needed afterwards. It's more useful when you only need a value back but do NOT want to insert the fallback into dict.

    Also `setdefault` is preferred when there is an **existing** dict that can't be swap out for a `defaultdict` e.g. it was passed in from outside your code, or you need a one-off default for a **single** key rather than all missing keys.

    When you need read, mutate in-place with any default, use `deafultdict`.
    
- `__missing__` and key lookup chain
    - The four ways to access a dict
        - `d[key]`          # __getitem__ → the ONLY one that triggers __missing__
        - `d.get(key)`      # direct C lookup → never triggers __missing__
        - pattern matching  # it uses `d.get(key, fallback)` so doesn't trigger __missing__
        - `key in d`        # __contains__  → never triggers __missing__
        - `d.setdefault(k)` # direct C lookup → never triggers __missing__
    - `d[key]` will first call `type(d).__getitem__(d, key)`, for a plain `dict`, that runs the C-level hash table lookup and if it's not found it will trigger `__missing__(self, key)` if it's defined before raising `KeyError`.
    - `__missing__` is ONLY defined on dict subclasses so plain dict has no `__missing__` and step 2 never happens for a plain dict!
    - The Python docs are explicit: "No other operations or methods invoke `__missing__()`." `d.get(key)`, `key in d`, and `d.setdefault(k)` are all implemented in C and do their own internal lookups, never calling `__getitem__` or `__missing__`. Only `d[key]` does, and only when `__getitem__` fails to find the key.
    - But `defaultdict` exploits `__missing__` to have a side effect (`__getitem__` side effect: inserts a default) when you do `d[new_key]` but **not** when doing `d.get('new_key')`.
         ```
        d = defaultdict(list)
        d['x'] # now d['x'] = []
        d.get('y') # return None and d is unchanged
        ```

- Merging dicts

    -  `d |= some_iterables` requires an iterable of **key-value pairs** e.g. `d |= [('a', 1), ('b', 2)]` works but not `d |= [1, 2]` which raises `TypeError`. This mirrors `dict()` behavior e.g. `dict([('a', 1), ('b', 1)])` works but not `dict([1, 2])`.

- ChainMap

    - Writes on a `ChainMap` always goes to the **first** map in the chain. This is the key design decision: reads search all maps left to right, but writes only affects the first.
    - `ChainMap` holds a reference to the actual dict objects, not copies hence any changes to the underlying dict will be reflected in `ChainMap`.

- Dict View

    - Views are readonly and they are lightweight objects that hold a **reference to the dict itself** - no data is copied! What this means is `d.keys()` returns an object that points back to `d`, and every access goes through the live dict. That's why views refelct the dict mutations as there is no separate data to go stale. 

    - `d.values()` view doesn't support set operations because values are not guaranteed to be **unique** and may not be **hashable**, whereas keys are guaranteed both which are required by the set operations.

- Set operations
    - The set difference operator is `-` not `not`!
    - A single check against a set is `O(1)` whereas for a list it's `O(n)` so to check `m` items e.g. `m` lookups against a collection of size `n`: set is `O(m)`; list is `O(m*n)`, set wins in the per-lookup cost, not the total!

- The Hash Contract
    - Python 3 auto sets `__hash__ = None` on any class that defines `__eq__`  but not `__hash__`, this makes instances explicitly unhashable rather than using the default identity-based hash e.g. `id(self)`.
    - Two hashable objects that are equal must have the same hash.

- Pattern Matching with Map
    - All mapping patterns in `match/case` allow extra keys by default. And `**rest` just captures those extra keys.
    - The reason for this design: mappings are open schemas — you often only care about a subset of fields (like reading one field from a JSON API response). Sequences are closed — **position and length** are part of the structure, so matching [a, b] against a 3-element list would be ambiguous.

- `UserDict` vs Subclassing dict
    - `dict` is implemented in C for performance, and its built-in methods (`update, setdefault, __init__`) call the C-level storage directly, bypassing Python's `__setitem__`. So if you subclass dict and override `__setitem__`, then do `d.update(other)`, your override is **never** called — the C code writes directly to the underlying hash table.
    - `UserDict.data` is a plain dict instance attribute; all of `UserDict`'s methods are pure Python and properly call `self[key] = value`, so your overrides are always respected.

- `OrderedDict` vs `dict`
    - What `OrderedDict` has that `dict` doesn't?
        - `move_to_end(key, last=True)` — repositions a key to the beginning or end
        - **Order-sensitive equality** — `od1 == od2` is False if order differs
        - `popitem(last=True/False)` — explicit LIFO or FIFO removal
    - Regular `dict` preserves order as an implementation details but equality ignores order.

## Chapter 4 Unicode Text vs Bytes

- Handling Text Files
    - The best practice for handling text I/O is the "Unicode Sandwich", 
        - meaning that `bytes` should be decoded to `str` as early as possible on input (e.g. when opening a file for reading). 
        - The "filling" of the sandwich is the business logic of a program, where text handling is done exclusively on `str` objects. Never do encoding or decoding in the middle of other processing. 
        - On output, the `str` are encoded to `bytes` as late as possible. So the two slices of bread are bytes (I/O boundaries) and filling is str (processing logic).
    - Python 3 makes it easier to follow this as the built-in `open` does necessary decoding when reading and encoding when writing files in text mode. But the best practice to make encoding explicit rather than relying on the default which may change from machine to machine or even change on the same machine.