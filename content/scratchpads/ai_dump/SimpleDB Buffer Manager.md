ai design dump


- **Use this minimal structure:** Share the manager through `Arc<BufferManager>`, keep the current snapshot in `RwLock<Arc<State>>`, and use each buffer lock as the slot lock.

  ```text
  BufferManager
  ├── current: RwLock<Arc<State>>
  └── buffers: Vec<Arc<RwLock<Buffer>>>

  State
  └── block_to_buffer: HashMap<Block, Arc<RwLock<Buffer>>>

  Buffer
  ├── block: Option<Block>
  ├── pins: usize
  ├── page contents
  └── dirty and log metadata
  ```

  - The inner `Arc<State>` keeps an old snapshot alive after the state read lock is released.
  - The outer `Arc` around the `RwLock` is unnecessary when the whole `BufferManager` is already shared through an `Arc`.
  - The buffer vector never changes after construction, so it needs no lock.
  - The pin count should exist in exactly one place.

- **Treat the state as an immutable map snapshot:** Readers clone the current `Arc<State>` while briefly holding `State R`, and writers clone and replace it while holding `State W`.

  ```text
  current ─▶ State v1: A -> S1, C -> S2

  reader clones State v1
  writer installs State v2: B -> S1, C -> S2

  reader still owns State v1
  ```

  - This is an RCU-like copy-and-swap design, but readers still briefly acquire an `RwLock`.
  - The snapshot freezes the map, but it does not freeze the mutable buffer behind an `Arc`.
  - `Arc` keeps the buffer allocation alive, but it does not prevent that buffer from being reassigned.
  - Writers must serialize the clone-and-swap operation under `State W`, or concurrent writers could overwrite each other’s updates.

- **Use this fast path:** Release `State R` immediately after cloning the snapshot, then lock and validate the selected buffer.

  ```text
  fast_pin(wanted):
      loop:
          snapshot = clone(*current.read())
          // State R is released here.

          buffer = clone(snapshot.block_to_buffer[wanted])

          if buffer does not exist:
              return slow_pin(wanted)

          buffer_guard = buffer.write()
          // This may wait for an in-progress load,
          // but no State lock is held while waiting.

          if buffer_guard.block != wanted:
              // The snapshot was old and this buffer was reassigned.
              drop(buffer_guard)
              continue

          buffer_guard.pin()
          drop(buffer_guard)
          return buffer
  ```

  - The block comparison and pin increment must occur under the same buffer write lock.
  - If the old reader locks first, its pin prevents reassignment.
  - If the loader locks first, the old reader later observes the changed block and retries.

- **Use this slow path:** Hold `State W` only long enough to recheck, claim a buffer, and publish the new snapshot.

  ```text
  slow_pin(wanted):
      loop:
          state_guard = current.write()

          if state_guard already contains wanted:
              drop(state_guard)
              return fast_pin(wanted)

          (buffer, buffer_guard) =
              first buffer whose write lock is immediately available
              and whose pins == 0

          if no buffer exists:
              drop(state_guard)
              return NO_AVAILABLE_BUFFER

          old_block = buffer_guard.block

          next = clone(**state_guard)

          if old_block exists:
              next.block_to_buffer.remove(old_block)

          next.block_to_buffer.insert(wanted, clone(buffer))
          *state_guard = Arc(next)

          drop(state_guard)
          // The global state pause ends here.

          // Buffer W remains held, so callers for wanted wait here.
          buffer_guard.assign_to_block(wanted)
          buffer_guard.pin()

          drop(buffer_guard)
          return buffer
  ```

  - Slot selection must skip a currently locked buffer rather than wait for it while holding `State W`.
  - Publishing before loading prevents a second thread from selecting another buffer for the same block.
  - The buffer write lock prevents anyone from observing the newly published mapping before its page contents are ready.
  - Different missing blocks can load concurrently when they claim different buffers.

- **Keep one universal lock order:** The slow path acquires `State W` and then attempts `Buffer W`, while the fast path releases `State R` before waiting for `Buffer W`.

  ```text
  slow pin:   State W -> Buffer W -> release State -> disk I/O -> release Buffer
  fast pin:   State R -> clone snapshot -> release State -> Buffer W
  unpin:      Buffer W only
  flush_all:  Buffer W for one buffer at a time
  ```

  - No operation should hold a buffer lock and then request the state lock.
  - `unpin` should decrement the pin count but leave the mapping installed when it reaches zero.
  - A zero pin count means the cached buffer is eligible for replacement, not that its mapping must immediately disappear.
  - `flush_all` does not need the state lock because the buffer vector is stable.

- **Use the buffer lock itself as the simplest loading wait:** A thread requesting the loading block finds the published buffer and blocks on its write lock, while `parking_lot` parks the thread instead of continuously spinning.

  ```text
  loader:  Buffer W [---------- disk I/O ----------] release
  waiter:            tries Buffer W ... sleeps ... acquires
  ```

  - A separate `Loading` state is unnecessary when the loader retains the buffer write lock throughout I/O.
  - A CAS loop is unsuitable for waiting because disk I/O can take long enough to waste millions of CPU iterations.
  - CAS could claim `Empty -> Loading`, but it would still need a parking mechanism for waiters.

- **Use an explicit loading state only if you need failure, timeout, or cancellation handling:** This version separates short metadata locking from page-content locking.

  ```text
  Slot
  ├── meta: Mutex<Meta>
  ├── ready: Condvar
  └── buffer: RwLock<Buffer>

  Meta
  ├── status: Empty | Loading(Block) | Ready(Block) | Failed
  └── pins: usize
  ```

  ```text
  waiter:
      meta = slot.meta.lock()

      while meta.status == Loading(wanted):
          slot.ready.wait(meta)
          // wait releases the mutex, parks, and reacquires it.

      if meta.status == Ready(wanted):
          meta.pins += 1
          return slot

      retry from a fresh state snapshot
  ```

  ```text
  loader:
      State W + Meta lock:
          mark Loading(wanted)
          reserve one pin
          install wanted -> slot
      release State W and Meta lock

      Buffer W:
          flush old page
          read wanted page
      release Buffer W

      Meta lock:
          mark Ready(wanted)
          notify all waiters
      release Meta lock
  ```

- **Pin placement matters only when it changes the lock boundary:** Keeping pins inside `Buffer` or immediately beside it in `Slot` is mechanically identical when both are protected by the same `RwLock`.

  - Reading the pin count requires a read lock, while changing it requires a write lock.
  - Keeping pins in separate `Meta` is beneficial only when pinning should proceed independently of page-content readers.
  - Keeping pins inside snapshot `State` is undesirable because every pin and unpin would require a state write and a new map snapshot.
  - For the minimal buffer-lock design, keep pins inside `Buffer`; for the condition-variable design, keep pins beside `Loading` and `Ready` inside `Meta`.

