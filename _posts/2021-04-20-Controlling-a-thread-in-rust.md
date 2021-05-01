---
layout: post
title:  "Controlling a thread in Rust"
---

Recently at work, I wanted to test some changes made on a server 'hassle-free'. Major motivations being:
- no actual app should be needed (my phone has only 64GB storage space :()
- no other device (like a phone) other than my dev machine
- no third party apps (e.g., postman) should be required
- wanted to write some code
- wanted to use rust
- dust off that HTTP basic knowledge

&nbsp;
And so here are some of the observations made during the development of this app.
&nbsp;
This mocking app/client-simulation app does nothing but calls a bunch of REST APIs. To do this, the rust ecosystem provides a bunch of libraries and I used reqwest for no special reason.
But the important part of the app is to invoke a particular REST API in a loop after a certain time interval. Moreover, there are two goals: 
- a JSON payload should be sent in this API request only if 'told to do so'
- there should be an ability to stop and restart this loop at any point in time.

&nbsp;
We obviously need to offload the looping on another thread and then somehow 'control' it from the main thread.

&nbsp;
Rust's ecosystem provides the tools to achieve our goals:
- communicating using a shared state
- communicating using channels

&nbsp;
For the first goal, we can use a mutex that'll be shared between the two threads and for the second, we can use a channel to send a 'stop' message to the 'worker' thread.
Our client simulator itself is a CLI app with a REPL where commands to invoke the REST APIs will be entered and executed. For simplicity, this is a largely single-threaded app with only one additional worker thread. Therefore, the REST API calls made will also be blocking and not async.

&nbsp;
A `CommandHandler` struct maintains the entire state like different HTTP clients (there are multiple backend servers), configuration settings, worker thread's handle and the shared state, sender part of the channel.

&nbsp;
```Rust
use std::sync::{mpsc, Arc, Mutex};
use std::thread;
use std::thread::JoinHandle;
...

struct CommandHandler {
    ep1_api_client: EP1Client,
    ep2_api_client: Arc<EP2Client>,
    tx: Option<Sender<()>>,
    worker_thread_handle: Option<JoinHandle<()>>
    is_payload_needed: Arc<Mutex<bool>> 
    settings: ConfigSettings
}
```

&nbsp;
Since we are dealing with handling commands here, the API invocation loop can be spawned like:

&nbsp;
```Rust
// Somewhere in a method ...
...
let client = self.ep2_api_client.clone();
let (tx, rx) = mpsc::channel();
self.tx = Some(tx);
let sleep_duration = self.settings.sleep_duration;
let is_payload_needed = self.is_payload_needed.clone();

let handle = thread::spawn(move || loop {
    {
        let val = is_payload_needed.lock().unwrap(); // Check if payload needed
        client.pulse(PulseRequestContext { // Invoke API
            is_payload_needed: *val,
        });
    }

    thread::sleep(Duration::from_millis(sleep_duration * 1000));

    match rx.try_recv() { // Is it time to quit?
        Ok(_) => break,
        Err(e) => match e {
            mpsc::TryRecvError::Empty => continue,
            mpsc::TryRecvError::Disconnected => break,
        },
    }
});

self.worker_thread_handle = Some(handle);
...
```

&nbsp;
So we've managed to spawn off our API invocation loop and have gotten a handle to it. Note that the `try_recv()` method above is non-blocking because we shouldn't be blocking indefinitely for a messaage to be received. 

&nbsp;
We can signal our worker thread to send a payload via commands that do this:
```Rust
...
let mut guard = self.is_payload_needed.lock().unwrap();
*guard = false; // or true if payload needed
...
```

&nbsp;
We can signal our worker thread to quit by:
```Rust
...
if self.worker_thread_handle.is_some() {
    self.tx.as_ref().unwrap().send(()); // send 'stop' message over the channel
    // consume the handle replacing it with None:
    if let Some(handle) = self.worker_thread_handle.take() { 
        handle.join(); // wait for the worker to do some cleanup perhaps
    }
    self.tx = None;
}
...
```

&nbsp;
There's a small problem with using a channel to stop the worker thread though. The problem is that if the thread is sleeping, it will be able to receive the stop messasge only after it wakes up. We can do better by using a condition-variable instead for signaling:

&nbsp;
```Rust
struct CommandHandler {
    ...
    pair: Arc<(Mutex<bool>, Condvar)>,
}

// Somewhere in a method ...
...
let pair = self.pair.clone();
let handle = thread::spawn(move || loop {
    ...
    let (should_quit, cvar) = &*pair;
    let result = cvar
        .wait_timeout(
            should_quit.lock().unwrap(),
            Duration::from_millis(sleep_duration * 1000),
        )
        .unwrap();
    let should_quit = *result.0;
    if should_quit {
        break;
    }
    ...
}
```

&nbsp;
We can now signal our worker thread to quit by a notification via the condition-variable:
&nbsp;
```Rust
...
let (should_quit, cvar) = &*self.pair;
{
    let mut should_quit = should_quit.lock().unwrap();
    *should_quit = true;
    cvar.notify_one();
}
...
```
&nbsp;
This will immediately quit the worker thread through the condition-variable since it waits for either the time-out to elapse or for the condition to change. However, the problem with using a condition-variable is:
a. There can be a lost wakeup - main thread sends a stop message to the worker even before the worker has started waiting on the condition.
b. There can be spurious wakeups where the thread will wakeup even before the time-out has elapsed or the condition has been satisfied due to different reasons.

&nbsp;
We can use `wait_timeout_while()` instead to avoid lost wakeups by passing a predicate function which checks the boolean for truthness:
&nbsp;
```Rust
...
let result = cvar
    .wait_timeout_while(
        should_quit.lock().unwrap(),
        Duration::from_millis(sleep_duration * 1000),
        |pending| !*pending,
    )
    .unwrap();
...
```

&nbsp;
Spurious wakeups can be avoided by using the `parking_lot` crate that guarantees that waiting will not return unless there was a notification.
