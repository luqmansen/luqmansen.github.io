---
title: Database Pattern in Rust
date: 2026-08-30
---

I'm not sure what to title I should put there. Point is, I want to highlight few coding pattern that you can use in Rust when building database, or any stateful application in which the state logic is self-contained in Rust, without any external dependencies (eg: external database). 

Why? Because once, before I learn rust properly, during reading the [[Database design and implementation]] book, I attempted to follow along and write the code in Rust (original code example is in Java). I was thinking to myself "aight, Rust should be similar to Go, with slightly stricter compiler, right?". Oh boy.... I was wrong. I hitting the compiler more than I thought it would be. I was too naive and jump head-first without equipping myself with the proper tools. At that time, I already speed-ran Rustbook + Rustling exercise, but it's not nearly enough. 

Hence, here, I want to synthesis what I have learn after that, during following this [[LSM in 3 weeks]] course. Most of these patterns are something that observed I know I should've used during my first attempt of writing database in Rust.


1. Serialized Write + RW-Lock
	This is one of the most important pattern when working with stateful app in rust, in particular, working with the borrowing rules. This part was all over the place during the early week 1 of [[LSM in 3 weeks]].  Now, I really want to formalize my understanding. 
	
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
	
		Which the `write/put` surprisingly  only takes reference to the self,  thanks to the lockless SkipList. Let's consider we're only going to use lockless (hence `mut`-less data structure)
	
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
pub fn force_freeze_memtable(&self) -> Result<()> {
	let mut guard = self.state.write();
	let mut state = guard.as_ref().clone();

	state.imm_memtables.insert(0, Arc::clone(&state.memtable));
	let new_memtable = MemTable::create(self.next_sst_id());
	state.memtable = Arc::new(new_memtable);

	*guard = Arc::new(state);

	Ok(())
}
		```

		What's wrong with above code? Essentially, during the second read, there could be 2 compactions that race trying to install the new latest state. There could be a genuine race condition there (lost update)
		
		This is a good timeline illustration for 
```
time   writer A (L0→L1 compaction)          writer B (L1→L2 compaction)
 |
t1     read()  → snapshot S0
       S0.levels[0] = [1,2,3]
 |
t2                                          read()  → snapshot S0   (same bytes!)
                                            S0.levels[0] = [1,2,3]
 |
t3     ...build sst 9 on disk...            ...build sst 10 on disk...
 |
t4     write(); *guard = S0 + levels[0]=[9]
       ┌──────────────────────────┐
       │ state now: levels[0]=[9] │
       └──────────────────────────┘
 |
t5                                          write(); *guard = S0 + levels[0]=[] , levels[1]=[10]
                                            ┌───────────────────────────────────┐
                                            │ state now: levels[0]=[]           │
                                            │ sst 9 is GONE from the tree       │
                                            └───────────────────────────────────┘

```


Why don't just hold the `state.write()` the entire time then? Because that way, you would block other threads that only want to read. 

This is where you need the `state_lock` / `Mutex` which essentially, to have 2 independent locks that we can call separately. That way, you can check the state before swapping the internal state.

Technically you can do this:
	
```rust
pub fn force_full_compaction(&self) -> Result<()> {

	let state_lock = self.state_lock.lock();
	
	// this might be long compaction
	let result = self.compact(&task) 
	
	// below is microsecond pointer swap
	let mut guard = self.state.write();
	let latest_date = guard.as_ref().clone()
	
	apply_result_to_state(latest_date, result);
	*guard = Arc::new(latest_date);

	Ok(())
}
```


I want to 