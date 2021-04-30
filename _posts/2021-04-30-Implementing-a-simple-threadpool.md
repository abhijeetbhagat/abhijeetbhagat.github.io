---
layout: post
title:  "Implementing a simple threadpool in rust"
date:   2021-04-30 16:32:41 +0530
categories: rust threadpool blocking-queue condvar 
---

A threadpool is a pool of threads that are waiting to process tasks. A very simple threadpool can look like this:
&nbsp;

                     bq      threadpool
 task1, task2, ... ======= |T|T|T|T|...|T|

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

The `BlockingQueue` above holds a queue to store items. Since this queue is going to be 'shared' between producers and consumers, it should be protected. This is done by wrapping it in a `Mutex`. Producers and consumers, thus, have to lock the queue before enqueing/dequeing items to/from it. In the enqueue operation, we first lock the queue making sure no one else accesses the queue which we insert an item into it. Once locked, we can safely insert an item and then notify a waiting consumer about the item we just inserted. After the `enq` method returns, the lock is automatically released.
&nbsp;
Similarly, in the dequeue operation, consumers wait on the condvar to be notified of any items enqueued but meanwhile, they can go to sleep in order to save precious CPU cycles. We add a waiting condition that basically says - if there's a notification to wake-up, check if the queue is empty; if the queue is empty, then go back to sleep. After waking up, if there's an item to be consumed, we can 'break' from the waiting condition and confidently dequeue (pop an item from the front of the queue using `pop_front`). 
&nbsp;
We are now ready to use the above `BlockingQueue` as a mechanism to consume tasks (or computations) produced by the client of our threadpool.
&nbsp;

