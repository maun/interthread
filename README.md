# interthread

> "The basic idea behind an actor is to spawn a 
self-contained task that performs some job independently
of other parts of the program. Typically these actors
communicate with the rest of the program through 
the use of message passing channels. Since each actor 
runs independently, programs designed using them are 
naturally parallel."
> - Alice Ryhl 


For a comprehensive understanding of the underlying
concepts and implementation details of the Actor Model,  
it's recommended to read the article  [Actors with Tokio](https:/ryhl.io/blog/actors-with-tokio/)
 by Alice Ryhl ( also known as _Darksonn_ ). 
This article not only inspired the development of the 
`interthread` crate but also serves as foundation 
for the Actor Model implementation logic in it. 


## What is the problem ?

To achieve parallel execution of individual objects 
within the same program, it is challenging due 
to the need for various types that are capable of 
working across threads. The main difficulty 
lies in the fact that as you introduce thread-related types,
you can quickly lose sight of the main program 
idea as the focus shifts to managing thread-related 
concerns.
It involves using constructs like threads, locks, channels,
and other synchronization primitives. These additional 
types and mechanisms introduce complexity and can obscure 
the core logic of the program.


Moreover, existing libraries like [`actix`](https://docs.rs/actixlatest/actix/), [`axiom`](https://docs.rs/axiom/latest/axiom/), 
designed to simplify working within the Actor Model,
often employ specific concepts, vocabulary, traits and types that may
be unfamiliar to users who are less experienced with 
asynchronous programming and futures. 

## Solution 
 
The [`actor`](https://docs.rs/interthread/latest/interthread/attr.actor.html) macro -  when applied to the 
implementation block of a given "MyActor" object,
generates additional types and functions 
that enable communication between threads.

A notable outcome of applying this macro is the 
creation of the `MyActorLive` struct ("ActorName" + "Live"),
which acts as an interface/handle to the `MyActor` object.
`MyActorLive` retains the exact same public method signatures
as `MyActor`, allowing users to interact with the actor as if 
they were directly working with the original object.

### Examples


Filename: Cargo.toml

```text
[dependencies]
interthread = "1.0.2"
oneshot     = "0.1.5" 
```

Filename: main.rs
```rust

pub struct MyActor {
    value: i8,
}

#[interthread::actor(channel=2)] // <-  this is it 
impl MyActor {

    pub fn new( v: i8 ) -> Self {
       Self { value: v } 
    }
    pub fn increment(&mut self) {
        self.value += 1;
    }
    pub fn add_number(&mut self, num: i8) -> i8 {
        self.value += num;
        self.value
    }
    pub fn get_value(&self) -> i8 {
        self.value
    }
}

// uncomment to see the generated code
//#[interthread::example(path="src/main.rs")] 
fn main() {

    let actor = MyActorLive::new(5);

    let mut actor_a = actor.clone();
    let mut actor_b = actor.clone();

    let handle_a = std::thread::spawn( move || { 
    actor_a.increment();
    });

    let handle_b = std::thread::spawn( move || {
    actor_b.add_number(5)
    });

    let _  = handle_a.join();
    let hb = handle_b.join().unwrap();

    // we never know which thread will
    // be first to call the actor so
    // hb = 10 or 11
    assert!(hb >= 10);

    assert_eq!(actor.get_value(), 11);
}

```
 
An essential point to highlight is that when invoking 
`MyActorLive::new`, not only does it return an instance 
of `MyActorLive`, but it also spawns a new thread that 
contains an instance of `MyActor` in it. 
This introduces parallelism to the program.

The code generated by [`actor`](https://docs.rs/interthread/latest/interthread/attr.actor.html) takes 
care of the underlying message routing and synchronization, 
allowing developers to rapidly prototype their application's
core functionality. This fast sketching capability is
particularly useful when exploring different design options, 
experimenting with concurrency models, or implementing 
proof-of-concept systems. Not to mention, the cases where 
the importance of the program lies in the result of its work 
rather than its execution.



The same example can be run in 
[tokio](https://crates.io/crates/tokio),
[async-std](https://crates.io/cratesasync-std), 
and [smol](https://crates.io/cratessmol), 
with the only difference being that the methods will 
be marked as `async` and need to be `await`ed for 
asynchronous execution.

### Examples
Filename: Cargo.toml

```text
[dependencies]
interthread = "1.0.2"
tokio = { version="1.28.2",features=["full"]}
```
Filename: main.rs

```rust

pub struct MyActor {
    value: i8,
}

#[interthread::actor(channel=2,lib="tokio")] // <-  one line )
impl MyActor {

    pub fn new( v: i8 ) -> Self {
       Self { value: v } 
    }
    // if the "lib" is defined
    // object methods can be "async" 
    pub async fn increment(&mut self) {
        self.value += 1;
    }
    pub fn add_number(&mut self, num: i8) -> i8 {
        self.value += num;
        self.value
    }
    pub fn get_value(&self) -> i8 {
        self.value
    }
}

#[tokio::main]
async fn main() {

    let actor = MyActorLive::new(5);

    let mut actor_a = actor.clone();
    let mut actor_b = actor.clone();

    let handle_a = tokio::spawn( async move { 
    actor_a.increment().await;
    });

    let handle_b = tokio::spawn( async move {
    actor_b.add_number(5).await
    });

    let _  = handle_a.await;
    let hb = handle_b.await.unwrap();

    // hb = 10 or 11
    assert!(hb >= 10);

    assert_eq!(actor.get_value().await, 11);
}
```


The [`actor`](https://docs.rs/interthread/latest/interthread/attr.actor.html) macro is applied to an impl block, allowing it to be used with both structs and enums to create actor implementations.

### Examples
Filename: Cargo.toml

```text
[dependencies]
interthread = "1.0.2"
oneshot     = "0.1.5" 
```

Filename: main.rs
```rust
#[derive(Debug)]
pub struct Dog(String);

impl Dog {
    fn say(&self) -> String {
        format!("{} says: Woof!", self.0)
    }
}

#[derive(Debug)]
pub struct Cat(String);

impl Cat {
    fn say(&self) -> String {
        format!("{} says: Meow!", self.0)
    }
}

#[derive(Debug)]
pub enum Pet {
    Dog(Dog),
    Cat(Cat),
}


#[interthread::actor(channel=2)]
impl Pet {
    // not in this case, but if 
    // the types used with `Pet` have different
    // parameters for the `new` method, 
    // simply pass a ready `Self` type
    // like this
    pub fn new( pet: Self) -> Self {
        pet
    }

    pub fn speak(&self) -> String {
        match self {
           Self::Dog(dog) => {
            format!("Dog {}",dog.say())
            },
           Self::Cat(cat) => {
            format!("Cat {}", cat.say())
            },
        }
    }
    pub fn swap(&mut self, pet: Self ) -> Self {
        std::mem::replace(self,pet)
    }
}


fn main() {

    let pet = PetLive::new( 
        Pet::Dog(Dog("Tango".to_string()))
    );

    let mut pet_a = pet.clone();
    let pet_b     = pet.clone();
    
    let handle_a = std::thread::spawn( move || {
        println!("Thread A - {}",pet_a.speak());
        // swap the the pet and return it  
        pet_a.swap(Pet::Cat(Cat("Kiki".to_string())))
    });

    let swapped_pet = handle_a.join().unwrap();

    let _handle_b = std::thread::spawn( move || {
        println!("Thread B - {}",pet_b.speak());
    }).join();

    //play with both pets now  
    println!("Thread MAIN - {}",pet.speak());
    println!("Thread MAIN - {}",swapped_pet.speak());

}
```
Outputs
```terminal
Thread A - Dog Tango says: Woof!
Thread B - Cat Kiki says: Meow!
Thread MAIN - Cat Kiki says: Meow!
Thread MAIN - Dog Tango says: Woof!
```


The crate also includes a powerful macro called [`example`](https://docs.rs/interthread/latest/interthread/attr.example.html) that can expand the [`actor`](https://docs.rs/interthread/latest/interthread/attr.actor.html) macro, ensuring that users always have the opportunity to visualize and interact with the generated code. Which makes [`actor`](https://docs.rs/interthread/latest/interthread/attr.actor.html)  100%  transparent macro . 

Using [`actor`](https://docs.rs/interthread/latest/interthread/attr.actor.html) in conjuction with [`example`](https://docs.rs/interthread/latest/interthread/attr.example.html) empowers users to 
explore more and more advanced techniques and unlock the full potential of 
parallel and concurrent programming, paving the way for 
improved performance and streamlined development processes.

### Examples
Filename: Cargo.toml

```text
[dependencies]
interthread = "1.0.2"
tokio = { version="1.28.2",features=["full"]}
```
Filename: main.rs
```rust
use tokio::time::{sleep,Duration};
pub struct MyActor;

#[interthread::actor(channel=2,lib="tokio")] 
impl MyActor {

    pub fn new() -> Self {Self}

    pub async fn sleep(&self, n:u8) {
        tokio::spawn(async move{
            // sleep one second
            sleep(Duration::from_secs(1)).await;
            println!("Task {} awake now!",n);
        });
    }
}

#[tokio::main]
async fn main(){

    let actor = MyActorLive::new();

    for i in 0..60 {

        let act_a = actor.clone();

        let _ = tokio::spawn(async move {
            act_a.sleep(i).await;
        });
    }
    // check how long
    // will take to sleep a minute 
    sleep(Duration::from_secs_f64(1.01)).await; 
}

```

Outputs (on my machine )

```terminal
Task 34 awake now!
Task 23 awake now!
Task 25 awake now!
Task 24 awake now!
Task 5 awake now!
        ...
Task 59 awake now!
Task 42 awake now!
Task 57 awake now!
Task 58 awake now!
Task 55 awake now!
```
60 in  total.

The above example demonstrates a more advanced usage of the [`actor`](https://docs.rs/interthread/latest/interthread/attr.actor.html) macro, showcasing its flexibility and capabilities. In this example, we explore non-blocking behavior that doesn't modify the state of the object or return any type.

To modify the state we'll need to use some additional types  "shared state types" or "thread-safe types".

### Examples

```rust

use tokio::time::{sleep,Duration};
use std::sync::{Arc,Mutex};

pub struct MyActor(Arc<Mutex<u8>>);

#[interthread::actor(channel=2,lib="tokio")] 
impl MyActor {

    pub fn new() -> Self {Self(Arc::new(Mutex::new(0)))}

    pub async fn sleep_increment(&self) {
        // clone the value 
        let value = Arc::clone(&self.0);
        tokio::spawn(async move{
            // sleep one second 
            sleep(Duration::from_secs(1)).await;
            // increment the value
            let mut guard = value.lock().unwrap();
            *guard += 1;
        });
    }
    pub fn get_value(&self) -> u8 {
        self.0.lock().unwrap().clone()
    }
}

#[tokio::main]
async fn main(){

    let actor = MyActorLive::new();

    for _ in 0..60 {
        let act_clone = actor.clone();

        let _ = tokio::spawn(async move {
            act_clone.sleep_increment().await;
        });
    }
    // play with Duration
    // set it to `1.00` and see how many tasks will
    // increment after sleep 
    sleep(Duration::from_secs_f64(1.01)).await;
    println!("Total tasks - {}", actor.get_value().await);
}

```
Outputs ( on my machine )

```terminal
Total tasks - 60
```

To return a type from the task, we will use a 'channel' 
from crate <a href="https://docs.rs/oneshot">oneshot</a>.
Tokio offers its own version of `oneshot`.

Filename: Cargo.toml

```text
[dependencies]
interthread = "1.0.2"
tokio = { version="1.28.2",features=["full"]}
```

### Examples

```rust

use tokio::time::{sleep,Duration};
use tokio::sync::oneshot::{self,Sender};
use std::sync::{Arc,Mutex};
pub struct MyActor(Arc<Mutex<u32>>);
// we use argument `id`
#[interthread::actor(channel=2,lib="tokio",id=true)] 
impl MyActor {

    pub fn new() -> Self {Self(Arc::new(Mutex::new(0)))}

    pub async fn init_actor_increment(&self,val:usize, sender: Sender<MyActorLive>){
        
        // clone the value of Actor 
        let value = Arc::clone(&self.0);
        tokio::spawn(async move {
            // I prefer to initialize them like this,
            // since they are competing with each other
            // to obtain the unique ID.
            // but if you commentout this "sleep" 
            // statement it will work anyway
            sleep(Duration::from_millis(val as u64)).await;

            //create actor
            let actor = MyActorLive::new();

            // send actor
            let _ = sender.send(actor);

            // increment the value
            let mut guard = value.lock().unwrap();
            *guard += 1;
        });
    }
    pub fn get_value(&self) -> u32 {
        self.0.lock().unwrap().clone()
    }
}

#[tokio::main]
async fn main(){
    let mut handles = Vec::new();
    let actor = MyActorLive::new();
    
  
    for i in 0..1000 {
        let act_clone = actor.clone();

        let handle = tokio::spawn(async move {

            let (send,recv) = oneshot::channel();
            
            // we want to receive an instance of 
            // new actor 
            // we send channel sender   
            act_clone.init_actor_increment(i, send).await;

            // awaiting for new actor 
            recv.await
        });
        handles.push(handle);
    }
    
    let mut actors = Vec::new(); 
    // receiving 
    for handle in handles {
        let act = 
        handle.await
              .expect("Task Fails")
              .expect("Receiver Fails");
        
        actors.push(act);
    }

    println!("Total tasks - {}", actor.get_value().await);
    println!("actors.len() -> {}", actors.len());
    

    // actors can be sorted by
    // the time they were invoked
    actors.sort();
    assert_eq!(actors[0] < actors[1],true); 
    assert_eq!(actors[121] < actors[122],true); 
    assert_eq!(actors[998] < actors[999],true); 


    // check if they have unique Ids 
    for i in (actors.len() - 1) ..0{
        let target = actors.remove(i);
        if actors.iter().any(move |x| *x == target){
            println!("ActorModel Ids are not unique")
        }
    }
    eprintln!(" * end of program * ");
}
```
The `id` argument is particularly useful when working with multiple instances of the same type, each/some serving different threads. It allows for distinct identification and differentiation between these instances, enabling more efficient and precise control over their behavior and interactions.

The following example serves as a demonstration of the  flexibility provided by the [`actor`](https://docs.rs/interthread/latest/interthread/attr.actor.html) macro. It showcases how 
easy is to customize and modify various aspects of the code generation process. 

### Examples

Filename: Cargo.toml

```text
[dependencies]
interthread = "1.0.2"
oneshot     = "0.1.5" 
```

Filename: main.rs
```rust
use std::sync::mpsc;
use interthread::actor;
 
pub struct MyActor {
    value: i8,
}

// this is initial macro 
// #[actor(channel=2,file="src/main.rs",edit(script(imp(play))))]
// will change to 
#[actor(channel=2, edit(script(imp(play))))]

impl MyActor {

    pub fn new( value: i8 ) -> Self {
        Self{value}
    }
    pub fn increment(&mut self) -> i8{
        self.value += 1;
        self.value
    }
    // it's safe to hack the macro in this way
    // while developing, along  with other
    // things will be created a new `Script` variant  
    // We'll catch it in `play` function
    pub fn play_get_counter(&self)-> Option<u32>{
        None
    }
}

// we have the code of `play` component
// using `edit` in conjuction with `file`
// Initiated By  : #[actor(channel=2,file="src/main.rs",edit(script(imp(play))))]  
impl MyActorScript {

    pub fn play( 
         receiver: mpsc::Receiver<MyActorScript>,
        mut actor: MyActor) {
        // set a custom variable 
        let mut call_counter = 0;
    
        while let Ok(msg) = receiver.recv() {
    
            // match incoming msgs
            // for `play_get_counter` variant
            match msg {
                // you don't have to remember the 
                // the name of the `Script` variant 
                // your text editor does it for you
                // so just choose the variant
                MyActorScript::PlayGetCounter { output  } =>
                { let _ = output.send(Some(call_counter));},
                
                // else as usual 
                _ => { msg.direct(&mut actor); }
            }
            call_counter += 1;
        }
        eprintln!("the end");
    }
}


fn main() {

    let my_act = MyActorLive::new(0);
    let mut act_a = my_act.clone();
    let mut act_b = my_act.clone();

    let handle_a = std::thread::spawn(move || {
        act_a.increment();
    });
    let handle_b = std::thread::spawn(move || {
        act_b.increment();
    });
    
    let _ = handle_a.join();
    let _ = handle_b.join();


    let handle_c = std::thread::spawn(move || {

        // as usual we invoke a method on `live` instance
        // which has the same name as on the Actor object
        // but 
        if let Some(counter) = my_act.play_get_counter(){

            println!("This call never riched the `Actor`, 
            it returns the value of total calls from the 
            `play` function ,call_counter = {:?}",counter);

            assert_eq!(counter, 2);
        }
    });
    let _ = handle_c.join();
}
```
The provided example serves as a glimpse into the capabilities of the actor macro, which significantly reduces the amount of boilerplate code required for interthread actors. While the example may not be immediately comprehensible, it demonstrates how the macro automates the generation of essential code, granting developers the freedom to modify and manipulate specific parts as needed.

For more details, read the
[![Docs.rs](https://docs.rs/interthread/badge.svg)](https://docs.rs/interthread#sdpl-framework)

If you like this project, please consider making a small  contribution. 

Your support helps ensure its continued development
<a href="https://www.buymeacoffee.com/6fm9wrhmk7V" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-blue.png" alt="Buy Me A Coffee" style="height: 30px !important;width: 108px !important;" ></a>

Join `interthread` on GitHub for discussions! [![GitHub](https://img.shields.io/badge/GitHub-%2312100E.svg?&style=plastic&logo=GitHub&logoColor=white)](https://github.com/NimonSour/interthread/discussions/1)

Please check regularly for new releases and upgrade to the latest version!

Happy coding! 
