# Threading & shared data reference

## Thread creation

```cpp
void create_thread(int tps, thread_callback@ cb); // ticking thread, tps 1–128, ONLY from main()
void on_render(thread_callback@ cb);              // runs on the render path
// tps 0 = one-shot (run callback once and exit) — creatable from ANY context; use for file I/O, image loading
// Ticking threads are drift-free: tick time accumulates each interval.
// funcdef: void thread_callback() {}
```

## Mutexes

```cpp
mutex@ mtx = create_mutex();
mtx.lock();           // exclusive, blocks
mtx.unlock();
bool ok = mtx.try_lock();   // non-blocking
mtx.lock_shared();    // shared (many readers)
mtx.unlock_shared();
```

## Atomics (lock-free, acquire/release ordering)

```cpp
atomic_bool@   atomic_bool(bool v);
atomic_int32@  atomic_int32(int v);   atomic_uint32@ atomic_uint32(uint v);
atomic_int64@  atomic_int64(int64 v); atomic_uint64@ atomic_uint64(uint64 v);
atomic_float@  atomic_float(float v); atomic_double@ atomic_double(double v);

T .load();  void .store(T);  T .exchange(T);          // returns previous
T .compare_exchange(expected, desired); // returns observed (== expected on success) — needed for read-modify-write
```

## Why sharing is dangerous

- **Unsynchronized global access** between threads can race on the AngelScript engine's internal state and crash the whole process.
- **Torn reads**: a reader sees a half-updated object (new frame number + old positions).

**Safe pattern — publish/acquire snapshots.** Handles are reference-counted: copying a handle just bumps the count, so a reader holding a handle keeps that snapshot alive and can read it lock-free after the swap:

```cpp
interface snapshot_t {}
class shared_snapshot {
    private snapshot_t@ current;
    private mutex@ mtx;
    shared_snapshot(snapshot_t@ initial) { @current = initial; @mtx = create_mutex(); }
    void publish(snapshot_t@ next) { mtx.lock(); @current = next; mtx.unlock(); } // build fully BEFORE publishing
    snapshot_t@ acquire() { mtx.lock_shared(); snapshot_t@ s = current; mtx.unlock_shared(); return s; }
}
// writer: build fresh snapshot each tick → g_state.publish(next);
// reader: game_snapshot@ snap = cast<game_snapshot>(g_state.acquire()); read freely, no lock held
```

Rule of thumb: single-writer/many-reader → snapshot swap; a counter → atomic with `compare_exchange`; anything else → mutex (prefer `try_lock` in tick loops to avoid stalls, `lock_shared` for read-only paths).

## Hash set — O(1) unique values

```cpp
hash_set_float@ hash_set_float(); hash_set_double@ hash_set_double();
hash_set_int32@ hash_set_int32(); hash_set_uint32@ hash_set_uint32();
hash_set_int64@ hash_set_int64(); hash_set_uint64@ hash_set_uint64();

bool .insert(v);   // true if inserted, false if already present
bool .erase(v);    // true if removed
bool .contains(v);
uint .size(); bool .empty(); void .clear();
```

## Hash map — uint64 keys, dynamically typed values

```cpp
enum hmap_type { unset, boolean, i32, u32, i64, u64, f32, f64, string };
hash_map@ map = create_hash_map();

void .set_bool/uint32/int32/int64/uint64/float/double/string(uint64 key, T value); // overwrite
bool .get_bool/...(uint64 key, T &out);   // true only if key exists AND type matches
bool .contains(uint64 key);               // any type
bool .erase(uint64 key);
hmap_type .get_type(uint64 key);
uint .size(); bool .empty(); void .clear();
```
