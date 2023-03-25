+++
title = "An elegant actor library in just 48 lines of Rust"
template = "page.html"
date = 2023-03-25T15:00:00Z
[extra]
summary = "For the Rust Vienna meetup in February, I hold a short talk about interior vs. exterior mutability in Rust. As part of that talk, I also showed how to avoid interior mutability by using actors to encapsulate state."
+++

For the [Rust Vienna](https://www.meetup.com/rust-vienna/) meetup in March, I hold a [short talk](https://github.com/rust-vienna/meetups#2023-02-23) about interior vs. exterior mutability in Rust.
As part of that talk, I also showed how to avoid interior mutability by using actors to encapsulate state.

While preparing the talk, I looked at actor libraries like [actix](https://docs.rs/actix/latest/actix/index.html) and several others, but they were either quite verbose or very macro-heavy, or both.
I wanted to keep it as simple as possible to avoid too much distraction from what I intended to show.
The result was a self-written actor library in just 48 lines of Rust that I'm now showing you in this blog post,
and thereby explain it in a bit more details.

It's also the first post on my new blog, and I'm happy about any feedback you might have.
Feel free to contact me by mail or leave a comment at the HN post. [TODO: link to hackernews post]

## A simple counter

Let's start directly with a motivating example on how to use that actor library.

First, we define the state the actor will act on. In this example, we just implement a simple counter.

```rust
struct Counter {
    value: u64,
}

impl Counter {
    fn new() -> Self {
        Counter { value: 0 }
    }
}
```

Next we define two messages, one for reading the current value, and one to increment the value.
Starting with the first one, this message doesn't have any parameters, so we just define an empty struct for it.

```rust
struct Get;
```

Now we define how the actor handles such a message. Therefor we implement the `Handler<M>` trait for our `Counter` where the type parameter `M` is our `Get` message.

```rust
use actor::Handler;

impl Handler<Get> for Counter {
    type Result = u64;

    fn handle(&mut self, _m: Get) -> Self::Result {
        self.value
    }
}
```

The type of the response is defined as an associated `Result` type. As the message has no parameters, the second argument of our `handle` function will be unused, and we just mark it as such (`_m`).

Now we do the same for the second message.

```rust
struct Inc(u64);

impl Handler<Inc> for Counter {
    type Result = u64;

    fn handle(&mut self, Inc(amount): Inc) -> Self::Result {
        self.value = self.value.saturating_add(amount);
        self.value
    }
}
```

This time, we take the amount of the increment as a parameter. We directly deconstruct the `Inc` message struct in the parameter list of our `handle` function.
To avoid overflows of the counter, we use a [`saturating_add`](https://doc.rust-lang.org/std/primitive.u64.html#method.saturating_add).

That's all we need to do, and we are ready to spawn and use the actor.

```rust
#[tokio::main]
async fn main() {
    let counter = actor::spawn(Counter::new(), 20);

    println!("Get: {}", counter.send(Get).await);       // Get: 0
    println!("Inc(1): {}", counter.send(Inc(1)).await); // Inc(1): 1
    println!("Inc(4): {}", counter.send(Inc(4)).await); // Inc(4): 5
}
```

This spawns our actor as a new tokio Task. The second parameter to `spawn` is just the size of the actor's mailbox (i.e. the number of concurrent messages it can hold before senders block).
We can now send messages to it and await the results. There's a `send` method for each message we defined `Handler<M>` for.

The state is nicely encapsulated in our actor. It can be freely manipulated by the `handle` methods of our actor.
All we need to interact with it is an immutable address which we get when spawning the actor.

Now it's time to look behind the scenes and have a look at the actor library that powers this all.


## The actor library

We already saw the `Handler<M>` trait and its definition is quite straight-forward:

```rust
pub trait Handler<M: Send + 'static> {
    type Result: Send + 'static;

    fn handle(&mut self, message: M) -> Self::Result;
}
```

The message and result both need to be sendable, so we require the [`Send`](https://doc.rust-lang.org/nomicon/send-and-sync.html) marker trait.
And they must have a `'static` lifetime.

Now to one of the more interesting parts. Every actor has a mailbox with which it is able to receive messages.
In our case this will be a [mpsc](https://docs.rs/tokio/latest/tokio/sync/mpsc/index.html) channel.

Let's define the mail format and the address.

```rust
use tokio::sync::mpsc::Sender;

type Mail<T> = Box<dyn FnOnce(&mut T) + Send>;

#[derive(Clone)]
pub struct Address<T> {
    sender: Sender<Mail<T>>,
}
```

The address just contains the sending end of the mpsc channel.
You'll probably be quite curious about this `Mail<T>` type.
It basically needs to represent the message we received and a way to send back the response (this will be an [oneshot](https://docs.rs/tokio/latest/tokio/sync/oneshot/index.html) channel).
The tricky part will be how to dispatch the messages to the corresponding handlers. Let's directly get to it by defining the `send` method for the address, then the `Mail<T>` type will also make more sense.

```rust
impl<T> Address<T> {
    pub async fn send<M: Send + 'static>(&self, message: M) -> <T as Handler<M>>::Result
    where
        T: Handler<M>,
    {
        let (response_sender, response_receiver) = oneshot::channel();

        let mail: Mail<T> = Box::new(move |t| {
            let result = T::handle(t, message);
            response_sender.send(result).unwrap_or_default() // just ignore when the caller lost interest in the result
        });

        self.sender.send(mail).await
            .or(Err("the actor panicked")).unwrap();

        response_receiver.await.expect("the actor panicked")
    }
}
```

There's a lot going on here. The send method is generic over the message type.
All we need is an implementation of a handler for that message type for our actor.
We can specify this as a trait bound (`T: Handler<M>`).
The message itself just needs to satisfy the same bounds as above when we defined the `Handler<T>` trait.

Now let's get to the body of that method.

First, we create our response channel. Then, this is where we create the mail we send to the actor.
In the send method, we know the handler the message should be dispatched to because we defined the existence of such a handler as a trait bound.
To utilize this information, we define the `Mail<T>` type as a boxed closure that will act on the state of our actor.
That closure captures exactly everything we need: The message, the sending end of our response channel, and the concrete handler we want to dispatch to.

This is what we send to the mailbox of our actor. After that we just await the response from the actor and return it to the caller.

The caller might lose interest at the result (i.e. it stops polling the future returned by `send` and drops it).
In this case the receiving end of the response channel will also be dropped and the send at the sending end will fail. We can safely ignore the error when sending in that case.

The actor itself should stay alive as long as there is even a single sender. So it should never drop neither the sending end of the response channel nor the receiving end of the mailbox.
The only way those would be dropped is when the actor panics. In this case we can just cascade this panic to the caller.
There would be an opportunity here to handle this differently and let the sender know that the actor failed, similar to a [poisoned mutex](https://doc.rust-lang.org/std/sync/struct.Mutex.html#poisoning),
but for the demo of my talk, I wanted to keep it as simple as possible.

Now that we went through the whole `send` method, all that's left is the message loop of the actor itself.
For that let's look at the final part, the `spawn` function.

```rust
pub fn spawn<T: Send + 'static>(mut t: T, buffer: usize) -> Address<T> {
    let (sender, mut receiver) = mpsc::channel(buffer);

    let addr = Address { sender };

    tokio::spawn(async move {
        while let Some(f) = receiver.recv().await {
            f(&mut t);
        }
    });

    addr
}
```

This function is comparatively simple. We create the channel for the mailbox (this time a mpsc channel),
move the sending end to the address and the receiving end as well as the mutable state `t` to the tokio task we spawn.
To be able to move the state to the new task, it must also have a `'static` lifetime and the `Send` marker trait.

The task just continuously awaits new messages and calls the boxed closure (our mail type) with a mutable reference of the state.

One elegant aspect of this message loop is how it utilizes the semantics of the mpsc channel to manage the lifetime of the actor.
As long as there exists any address to the actor, it will wait for new messages.
But after the last address was dropped, all senders to the mpsc channel got dropped too and `recv()` will return `None`,
therefor also nicely terminating our message loop.

And that's it, those are all the 48 lines of Rust for this actor library.

I hope you enjoyed it. You can find the whole source code at the [github repo](https://github.com/tdanecker/rust-mutability/blob/main/src/actor.rs) I published after my talk.

If you like to leave a comment, check out the corresponding HN post. [TODO: link to hackernews post]
