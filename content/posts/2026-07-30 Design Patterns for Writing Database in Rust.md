---
title: Design Patterns for Writing Database in Rust
date: 2026-07-30
---
I want to highlight few coding patterns that you can use in Rust when building a database, or any stateful application in which the state logic is self-contained in Rust, without any external dependencies (eg: external database). 

Why? Because once, before I learned rust properly, during reading the [[Database Design and Implementation]] book, I attempted to follow along and write the code in Rust (original code example is in Java). I was thinking to myself "Rust should be similar to Go, with slightly stricter compiler, right?". Oh boy.... I was wrong. I was hitting the compiler errors more than I thought it would be. I was too naive and jump head-first without equipping myself with the proper knowledge. At that time, I was already speed-ran Rustbook + Rustling exercise, but it's not nearly enough. 

Hence, I want to synthesis what I have learned after that, during following this [[LSM in 3 weeks]] course. Most of these patterns are something that observed and I should've known during my first attempt of writing database in Rust.

## Serialized Write + RW-Lock

This is one of the most important pattern when working with stateful app in rust, in particular, working with the borrowing rules. This pattern was all over the place during the early week 1 of [[LSM in 3 weeks]].  

Without this, my first attempt was to sprinkle `Arc<Mutex<T>>`  all over my code. Technically speaking, it will work, but it's unidiomatic and most importantly will grind your database to a halt, because read and write require synchronization.

Suppose that these are the core structs of the storage engine you're trying to build:
```rust
// rest of the properties are omitted for clarity

/// Represents the state of the storage engine.
pub struct LsmStorageState {
	/// The current memtable.
	pub memtable: Arc<MemTable>,
	/// Immutable memtables, from latest to earliest.
	pub imm_memtables: Vec<Arc<MemTable>>,
	
	pub sstables: HashMap<usize, Arc<SsTable>>,
	
	// .... omitted 
}

pub(crate) struct LsmStorageInner {
	pub(crate) state: Arc<RwLock<Arc<LsmStorageState>>>,
	pub(crate) state_lock: Mutex<()>,
	
	// .... omitted 
}



/// A thin wrapper for `LsmStorageInner` and the user interface for MiniLSM.
/// which We don't use this at least until week 3
pub struct MiniLsm {
	pub(crate) inner: Arc<LsmStorageInner>,
	
}
```

There are 2 main access patterns:

1. `LsmStorageState.memtable`'s `read` and `write` 

	Which the `write/put` surprisingly  only takes reference to the self,  thanks to the lockless SkipList. Let's consider we're only going to use lockless (hence `mut`-less data structure) for now. 

2. `LsmStorageState` mutation

   This is where for me as a person who used to with the convenience of runtime-managed language tripped the most.
	  
   There are few operations where you need to have mutable access to the inner state:
   - flushing memtable immutable `imm_memtables`
   - modifying the `sstables` , `levels`
   - etc
   
	The point is, during this mutation, you need full exclusive lock over the `state`.  
	Ok, then why do i have both mutex and rw-lock 
	```rust
	pub(crate) state: Arc<RwLock<Arc<LsmStorageState>>>,
	pub(crate) state_lock: Mutex<()>,
	```
	?
	
	This is the first pattern: State swapping under guarded Mutex.
	

	Suppose this code
	
	```rust
	pub fn put(&self, _key: &[u8], _value: &[u8]) -> Result<()> {
		// first block
		{
			let state = self.state.read();
			if state.memtable.approximate_size() < self.target_size { 
				state.memtable.put(_key, _value);
				return Ok(())
		}
		
		// second block
		let mut guard = self.state.write();
		let mut state = guard.as_ref().clone();
		
		state.imm_memtables.insert(0, Arc::clone(&state.memtable));
		let new_memtable = MemTable::create(self.next_sst_id());
		state.memtable = Arc::new(new_memtable);
		
		state.memtable.put(_key, _value); 
		
		*guard = Arc::new(state);
		
		Ok(())
	}
	```

	What's wrong with above code? Essentially, during the second block, there could be 2 threads that race trying to install freeze the memtable and replace it with a new one. There could be a genuine race condition there (lost update of the prev newly installed memtable here.
	
	This is a good timeline illustration for this. Suppose there're 2 threads, A and B
```
		  state.memtable
t0            ┌────────┐
		  │  M0    │   ← puts land here
		  └────────┘

t1  A snapshots: sees M0. creates M1.
t2  B snapshots: sees M0. creates M2.

t3  A installs   memtable = M1,  imm = [M0]
		  ┌────────┐
		  │  M1    │   ← puts land here now
		  └────────┘        put("x","1") → goes into M1

t4  B installs   memtable = M2,  imm = [M0]
		  ┌────────┐
		  │  M2    │
		  └────────┘

M1  ──▶  unreachable. refcount hits 0. freed.
		 "x" was never flushed, never in imm_memtables,
		 never on disk. The put returned Ok(()).


```


Why don't just hold the `state.write()` the entire time then? Because that way, you would block other threads that only want to read. 

This is where you need the `state_lock` / `Mutex` which essentially, to have 2 independent locks that we can call separately. That way, you can check the state before swapping the internal state.
```rust
pub fn put(&self, _key: &[u8], _value: &[u8]) -> Result<()> {
// first block
{
	let state = self.state.read();
	if state.memtable.approximate_size() < self.target_size { 
		state.memtable.put(_key, _value);
		return Ok(())
}

// NEW!!!
let state_lock = self.state_lock.lock();
{
	// read again.there's a chance that other thread
	// has already froze the table
	let state = self.state.read();
	if state.memtable.approximate_size() < target_size {
		let v = state.memtable.put(_key, _value);
		return Ok(());
	} 
} // read lock released here! 

let mut guard = self.state.write();
let mut state = guard.as_ref().clone();

state.imm_memtables.insert(0, Arc::clone(&state.memtable));
let new_memtable = MemTable::create(self.next_sst_id());
state.memtable = Arc::new(new_memtable);

state.memtable.put(_key, _value); 

*guard = Arc::new(state);

Ok(())

// state_lock released here!
}
```



 **Questions!!!**

If the `memtable` is genuinely lockless data structure, why don't we just put it outside as standalone `Arc` without wrapping it with a RW-lock? The write and read doesn't require `mut` anyway

We used to do this in Golang!
```go
type Inner struct {                     
    mu    sync.Mutex // equal to guards state        
    state *State                        
}  
```

We should be able to create something equivalent in rust, like this?

```rust

struct LsmStorageInnerRwLockLess{
	state: Arc<LsmStorageState>
	mutex: Mutext()
}
```


And this this is how we should freeze!
```rust
impl LsmStorageInnerRwLockLess {
    fn freeze(&self) {
        let _g = self.state_lock.lock();
        
        let mut s = (*self.state).clone();
        
        s.imm_memtables.insert(0, Arc::clone(&self.state.memtable));
        
        self.state = Arc::new(s);
    }
}
```

Bing bong! Your code won't compile!

```rust
self.state = Arc::new(s);
	//  ^^^^^^^^^^ 
	// cannot assign to `self.state`, which is behind a `&` reference
	// `self` is a `&` reference, so it cannot be written to
	
```

Can we make the self `&mut`? 
```rust
impl LsmStorageInnerRwLockLess {
    fn freeze(&mut self) { // changed to &mut!!!!
        let _g = self.state_lock.lock();
        
        let mut s = (*self.state).clone();
        
        s.imm_memtables.insert(0, Arc::clone(&self.state.memtable));
        
        self.state = Arc::new(s);
    }
}
```

Yes but error will be on the call site!

(simplified code)
```rust
use std::sync::{Arc, Mutex};

#[derive(Clone)]
struct State { n: usize }

struct Inner {
    state: Arc<State>,
    state_lock: Mutex<()>,
}

impl Inner {
    fn freeze(&mut self) {
        let _g = self.state_lock.lock();
        let mut s = (*self.state).clone();
        s.n += 1;
        self.state = Arc::new(s);
    }
}

fn main() {
    let inner = Arc::new(
	    Inner { 
		    state: Arc::new(State { n: 0 }), 
		    state_lock: Mutex::new(()) 
	});
	
    let flush_thread_handle = Arc::clone(&inner);
    std::thread::spawn(move || {
        flush_thread_handle.freeze();
    });
    inner.freeze();
}
```

The error
```rust
error[E0596]: cannot borrow data in an `Arc` as mutable
  --> src/lib.rs:24:9
```

why?

So, this is the common newbie mistake when using  `&mut` thinking that it's a mutable and they can freely use it whenever they want. It's not. The `mut` keyword signifies that it's an exclusive access to this variable. That means, there's only one place that allowed to mutate this variable at any given time. That's not going to happen on this code, because when you're already using `Arc::clone` at least there are 3 references already during the start of the program. The only way to get it working is to use `Arc::get_mut` which will returns `Options<T>`  i.e. only work when the number of strong reference is equal to 1, otherwise it will always be `None`.

Back to the golang code which is super common pattern when you're using mutex. Btw, I will use Golang as an example a lot to bridge my understanding, as I'm very familiar with it.

```go
type Inner struct {                     
    mu    sync.Mutex
    state *State                        
}  
```

In this code, no one is stopping you from mutating the `state` even without holding the lock. It relies on programmer discipline to prevent race condition. Rust prevents this by enforcing you to wrap your state inside a lock primitive, either it is a `Mutex<T>` or `RwLock<T>` (well you can implement your own lock too, but that's for another day).

I still want to at least write about 
- Self-referencing data structure in Rust (for iterator pattern)
- Higher-Rank Trait Bound blackmagic fuckery
- Generative Associated Type

To be continued....
