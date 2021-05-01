---
layout: post
title:  "Implementing a simple threadpool in rust"
---

A threadpool is a pool of threads that are waiting to process tasks. A very simple threadpool can look like this:
&nbsp;
                     bq     threadpool
 task1, task2, ... ======= |T|T|T|T|...|T|
&nbsp;
Various tasks (task1, task2, ...) are sent over a blocking queue (bq) and are picked by threads (T) waiting in the threadpool. A blocking-queue is a mechanism using which produced items can be enqueued by producers and dequeued by consumers waiting on those items. Of course, we don't want consumers to continuously poll the queue to check if items are available or not. Instead, we want producers to signal consumers whenever items are available for consumption. Until then, the consumers can 'sleep'.
&nbsp;
This is where a condvar (condition variable) can come handy for signalling. The producers and consumers can rely on this condvar to notify and be notified of items in the queue.
&nbsp;
```
use std::collections::VecDeque;
use std::sync::{Arc, Condvar, Mutex};

pub struct BlockingQueue<T> {
    queue: Mutex<VecDeque<T>>,
    condvar: Condvar,
}

impl<T> BlockingQueue<T> {
    pub fn new() -> Self {
        Self {
            queue: Mutex::new(VecDeque::new()),
            condvar: Condvar::new(),
        }
    }

    pub fn enq(&self, item: T) {
        let mut vec = self.queue.lock().unwrap();
        vec.push_back(item);
        self.condvar.notify_one();
    }

    pub fn deq(&self) -> Option<T> {
        let mut vec = self
            .condvar
            .wait_while(self.queue.lock().unwrap(), |vec| {
                vec.is_empty()
            })
            .unwrap();

        vec.pop_front()
    }
}
```
&nbsp;

The `BlockingQueue` above holds a queue to store items. Since this queue is going to be 'shared' between producers and consumers, it should be protected. This is done by wrapping it in a `Mutex`. Producers and consumers, thus, have to lock the queue before enqueing/dequeing items to/from it. In the enqueue operation, we first lock the queue making sure no one else accesses the queue while we insert an item into it. Once locked, we can safely insert an item and then notify a waiting consumer about the item we just inserted. After the `enq` method returns, the lock is automatically released.
&nbsp;
Similarly, in the dequeue operation, consumers wait on the condvar to be notified of any items enqueued but meanwhile, they can go to sleep in order to save precious CPU cycles. We add a waiting condition that basically says - if there's a notification to wake-up, check if the queue is empty; if the queue is empty, then go back to sleep. After waking up, if there's an item to be consumed, we can 'break' from the waiting condition and confidently dequeue (pop an item from the front of the queue using `pop_front`). 
&nbsp;
We are now ready to use the above `BlockingQueue` as a mechanism to consume tasks (or computations) produced by the client of our threadpool.
&nbsp;
Our `Threadpool` is pretty simple, in that, it needs something to store a bunch of threads and something to receive tasks/computations to process. We can use a simple `Vec` to store our threads and a `BlockingQueue` to receive tasks:
&nbsp;
```
use blocking_queue::BlockingQueue;
use std::sync::Arc;
use std::thread::JoinHandle;

pub struct ThreadPool {
    bq: Arc<BlockingQueue<Box<dyn FnOnce() + Send + 'static>>>,
    handles: Vec<JoinHandle<()>>,
    should_quit: Arc<AtomicBool>,
}
```
&nbsp;
We start off with spawning threads and storing their handles in the 'ctor':
&nbsp;
```
    pub fn new() -> Self {
        let bq: Arc<BlockingQueue<Box<dyn FnOnce() + Send + 'static>>> =
            Arc::new(BlockingQueue::new());
        let should_quit = Arc::new(AtomicBool::new(false));

        let handles = (1..=5)
            .map(|_| {
                let bq = Arc::clone(&bq);
                let should_quit = Arc::clone(&should_quit);
                std::thread::spawn(move || {

                    while !should_quit.load(Ordering::Relaxed) {
                        match bq.deq() {
                            Some(t) => t(),
                            _ => return,
                        }
                    }
                })
            })
            .collect();

        Self {
            bq,
            handles,
            should_quit,
        }
    }
```
&nbsp;
We want to enqueue a task. Something that will execute only once. We can therefore represent a task as an `FnOnce` instance. However, we can't just store multiple `FnOnce` instances. This is because, no two `FnOnce` instances, even though similar, have the same type. Since our queue underneath is a `VecDeque` and expects all the items in it to be homogenous (same type), we can't just store multiple `FnOnce` instances in it, unless we wrap it in a `Box` type. This way we not only get ownership of the tasks but also, the compiler is now happy because all the items are now trait objects which are pointers to different types implementing the same trait - `FnOnce`. So instead of storing concrete types that implement `FnOnce`, we are storing trait objects which is acceptable.
&nbsp;
However, there's now a new problem. We are enqueing tasks, or in order words, sending them as trait objects from one thread (producer) to another (consumer). But trait objects by themselves are not safe to be sent across threads. We can get around the problem by telling the compiler that they are indeed safe to be sent across threads by using the `Send` marker trait. Also, whatever we are passing to a thread needs to live at least as long as the thread's lifetime, which could be as long as the main thread's lifetime. That's why, we also need to indicate this to the compiler through the `'static` lifetime bound:
&nbsp;

```
pub struct ThreadPool {
    bq: Arc<BlockingQueue<Box<dyn FnOnce() + Send + 'static>>>,
    ...
}

impl ThreadPool {
    ...
    pub fn execute<F>(&self, f: F)
    where
        F: Send + 'static + FnOnce() -> (),
    {
        self.bq.enq(Box::new(f));
    }
    ...
}
```
&nbsp;
The last bit there's left to be done is adding a way to shut the pool when we are done using it. We can use an `AtomicBool` to perform this flagging:
&nbsp;
```
    while !should_quit.load(Ordering::Relaxed) {
        match bq.deq() {
            Some(t) => t(),
            _ => return,
        }
    }
```
&nbsp;
It translates to - while we are not asked to stop, pull a task out of the queue and process it; if there is nothing to process, then quit. We can set this flag in a `shutdown` method:
&nbsp;
```
    pub fn shutdown(mut self) {
        self.should_quit.store(true, Ordering::Relaxed);
        let _: Vec<()> = self
            .handles
            .drain(..)
            .map(|handle| {
                let _ = handle.join();
            })
            .collect();
    }
```
&nbsp;
There's one problem though. Dequeing puts a thread to sleep until the producers feeds it a task. If while doing this we try to shut the pool down, no threads will see this flag raised since they are all sleeping waiting for tasks to be enqueued. As a result, we also need to shut our `BlockingQueue` so that alls the sleeping threads are awakened and see that they need to quit.
&nbsp;
We can add a `quit` method to `BlockingQueue` that will simply notify all the threads to wake up using the `notify_all` method on condvar. However, they also need to escape the condition they are waiting on after they wake up. We can use the same trick - use an `AtomicBool` flag indicating a shutdown:
&nbsp;
```
    pub fn deq(&self) -> Option<T> {
        let mut vec = self
            .condvar
            .wait_while(self.queue.lock().unwrap(), |vec| {
                // wait while the queue is empty && we aren't are not told to quit
                vec.is_empty() && !self.should_quit.load(Ordering::Relaxed) 
            })
            .unwrap();

        if self.should_quit.load(Ordering::Relaxed) {
            return None;
        }

        vec.pop_front()
    }

    pub fn quit(&self) {
        self.should_quit.store(true, Ordering::Relaxed);
        self.condvar.notify_all();
    }
```
&nbsp;
and call it from our `ThreadPool::shutdown` method:
&nbsp;
```
    pub fn shutdown(mut self) {
        self.bq.quit(); // wake up all the consumer threads
        self.should_quit.store(true, Ordering::Relaxed);
        let _: Vec<()> = self
            .handles
            .drain(..)
            .map(|handle| {
                let _ = handle.join();
            })
            .collect();
    }
```
In the `Threadpool::shutdown` method, we consume all the handles by draining the pool and giving all the threads a chance to finish up before exiting by calling `join` on the handles. 
&nbsp;
We can now use our `ThreadPool`:
&nbsp;
```
use std::thread;
use std::time::Duration;
use threadpool::ThreadPool;

fn main() {
    let tp = ThreadPool::new();
    tp.execute(|| {
        println!("{:?}: task 1: Hello!", std::thread::current().id());
    });
    tp.execute(|| {
        println!("{:?}: task 2: Hello!", std::thread::current().id());
    });

    thread::sleep(Duration::from_secs(2));

    tp.shutdown();
}
```
&nbsp;

