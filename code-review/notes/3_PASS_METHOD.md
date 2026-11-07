### 🧭 How to Review Faster (The 3-Pass Method)

  Instead of reading top-to-bottom and hoping bugs jump out at you, use three quick passes
  (~3–5 minutes each):

  #### Pass 1: State & Data Structures (3 mins)

  Look only at the fields at the top of the class first for members.
  Ask:

  1. Is this class shared across threads?
  2. Are nested collections or keys mutable? (Map<Set<String>, List<String>> -> Wait: Set is
  the key, can it be mutated? List is the value, what type of List is inside the thread-safe
  map?)
  1. Can these collections grow forever? (recentEvents -> Where are elements removed? What
  happens if purgeStaleEvents isn't called?)

  #### Pass 2: Method-by-Method Contracts & Mutation (5 mins)

  For each method, trace inputs, mutations, and outputs:

  1. Inputs: Are we storing a caller's mutable object (Set, List, Date, array) directly
  without copying it?
  2. Loops & Iteration: Are we modifying a collection (add/remove) while looping over it?
  Are we iterating over a synchronizedList without a synchronized block?
  3. Compound Checks: Do we check .size() or .contains() on one line and then act on the
  next line (Time-Of-Check to Time-Of-Use / TOCTOU race)?
  4. Outputs: Are we returning our internal mutable List/Set/Map directly to the caller so
  they can accidentally modify our internal state?

  #### Pass 3: Write the Comment Using "What + Why + How" (5 mins)

  When you leave a comment, structure it in 3 short sentences:

  1. What is broken: "Calling .add() on the ArrayList returned by computeIfAbsent is not
  thread-safe."
  2. Why it matters in production: "Concurrent threads registering the same tag set will
  race on the ArrayList, causing lost product IDs or ArrayIndexOutOfBoundsException."
  3. How to fix it: "Use ConcurrentHashMap.newKeySet() instead of ArrayList for the map
  values."


### 💡 Staff Reviewer Example Comment

> **Blocker (Lines 12–13, 21–25):** Thread-unsafe `ArrayList` mutation and mutable `Set` keys in `ConcurrentHashMap`
> Two critical concurrency/collections issues here:
> 1. `computeIfAbsent(...).add(productId)` is not thread-safe. Once `computeIfAbsent` returns the `ArrayList`, `.add(productId)` runs outside `ConcurrentHashMap`'s atomic lock. Concurrent registrations for the same tag set will race on the `ArrayList`, corrupting its internal array or losing elements.
> 2. `Set<String> tags` is used directly as a map key and later mutated in `appendTagToGroup` (`existingTags.add(newTag)`). Mutating a `Set` after inserting it as a `Map` key changes its `hashCode()`, stranding the entry in the wrong hash bucket so subsequent `get()`/`remove()` calls silently fail.
> 
> 
> **Suggested fix:**
> Defensively freeze the key with `Set.copyOf(tags)` and use a thread-safe value collection (or atomic compute with immutable sets):
> ```java
> private final Map<Set<String>, Set<String>> tagIndex = new ConcurrentHashMap<>();
> 
> public void registerProduct(String productId, Set<String> tags) {
>     Set<String> immutableKey = Set.copyOf(tags);
>     tagIndex.computeIfAbsent(immutableKey, k -> ConcurrentHashMap.newKeySet()).add(productId);
>     recordEvent("REGISTERED:" + productId);
> }
> 
> ```
> 
> 
> To re-tag a group in `appendTagToGroup`, atomically remove the old key (`tagIndex.remove(existingTags)`) and merge its products under the new immutable key rather than mutating the key in-place.