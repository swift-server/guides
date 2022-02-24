# Index For talks about Performance in Swift

## Overview: 
This guide contains a list of talks and the key takeaways from it. It explains the published wwdc talks
but with more examples and some other scenarios to better understand it. 

### Contents:

- [Class Vs Struct tradeoffs and CoW performance](#class-vs-struct-tradeoff-and-cow-performance)
    - [Difference btw structs and classes](#differences)
    - [Copy-on-Write](#copy-on-write)
    - [Benchmarking Struct with Class](#benchmarking-structs-with-classes)
    - [Implementing Struct backed by Class](#implementing-structs-backed-by-classes)
    - [Implementing Struct backed by Class with Copy on Write](#struct-backed-by-class-with-copy-on-write)
    - [Drawbacks of CoW](#drawbacks-of-copy-on-write)

- [Concurrency and Locking](#concurrency-and-locking-performance)
    - [Race Condition with Semaphore](#race-conditions-with-semaphores)
    - [Priority inversion with Semaphore](#priority-inversion-with-semaphores)
    - [Thread explosion](#thread-explosion)
    - [Runtime contract and rewriting code with async/await semantics for better code performance.](#runtime-contract)
    - [Synchronization offering:](#synchronization)
        - [Mutual Exclusion](#mutual-exclusion)
            - This covers:
                - [Actors with data races when we are accessing the mutable states within the actors](#accessing-mutable-state-within-the-actor)
                - [Actors with data races when we are accessing the mutable states outside the actors and fixing it with sendable protocols](#accessing-mutable-state-outside-the-actor)
        - [Reentrancy and Prioritization](#reentrancy-and-prioritization)
            - This covers actors with Priority Inversion
        - [Main Actor](#main-actor)
            -Batching up code to improve time complexity and performance
    - [Potential bug around await semantics](#⚠️-potential-bug)
    - [Lock Contention Pattern](#lock-contention-pattern)







## [Class Vs Struct tradeoff and CoW performance](https://www.youtube.com/watch?v=iLDldae64xE)

### Differences:

For a better tradeoff, lets first consider the differences between classes and struct:

Classes have reference semantics while structs have value semantics. Structs are used as 
values whereas classes are used as objects with identity. We use structs to check whether 
the values contained by both the objects are same or not with the use of == whereas we 
use classes to check whether the variables hold the same reference as their value or not 
with === .

Classes can hold values with same reference and hence if one value gets changed, another value 
will also be changed.
But if I copy a struct, values associated with the struct won’t be changed. So if I want to  change
 a copy of a struct, it won’t affect other copies.  
So, if I want safety in mutation, structs will be chosen but if I want flexibility of creating more
classes based on other classes, classes will be choosed.

Memory-wise, Structs use stack memory allocation while classes rely on heap memory allocation.
Stack allocation like insertion and deletion are faster than heap that manages reference count of
every object. Therefore, structs have faster memory allocation and are more performant in use than
classes. 

### Copy-on-Write
Value Semantics performs deep copy either when a new variable is introduced or an existing variable is mutated. 
If our structs contain reference types, the reference types won’t automatically get copied upon assigning the struct to
a new variable. Instead, the references themselves get copied. This is called a shallow copy.

But gradually, as the number of instances increases in the stack, it becomes difficult to copy
large value types. To reduce the complexity of copying large value types, copy on write(CoW) is
used.

With copy on write behaviour, we don’t need to create copies of the variable when its passed,
rather only creates the copy when the objects changes its value. Eg- we have an array of 5 million
data that is being passed but a copy will only be produced if the array changes.
Thus, CoW avoids memory overhead of creating large amount of copies. 

Consider an example:
## 

```swift
public struct HTTPRequest {
    public var method: HTTPMethod
    public var target: String
    public var version: HTTPVersion
    public var headers: [(String, String)]
    public var body: [UInt8]?
    public var trailers: [(String, String)]?
]
public enum HTTPMethod: String{
    case GET
    case POST
    case DELETE
    case PUT
    // …
}
public struct HTTPVersion{
    var major: Int
    var minor: Int
}
```
``` swift
let swift=HTTPRequest(method: .GET, target: “/performance/swift”,…)
var lang=swift
lang.target=“/performance/Nio”
XCTAssertEqual(swift.target, “/performance/swift”)   //results true
```
Here we are creating an HTTPRequest and created a copy of swift object into lang. Now, If I try to
change the target of lang, the swift target remains same as a result of structs based HTTPRequest.

```swift 
public func transform(_ httpRequest: HTTPRequest)->HTTPRequest{
    return httpRequest
} 
_ = transform(httpRequest)

```
### Benchmarking structs with classes
Here, we are creating a transform function that returns our httpRequest back. This method is mainly
created for the purpose of checking speed.
The transform function for a struct httpRequest takes 52 nano seconds to run.

However, if I use a class HTTPRequest, the transform method gives a result in 15 nano seconds which
seems quite performant ideally. But we loose upon the value semantics. As we change the target of
lang, the swift target will also be changed. This can cause memory leaks and threading problems
which ain’t performant.
In such cases we can use struct backed by class. This way we can gain the value semantics of the
struct and faster results from the classes.

### Implementing Structs backed by Classes
``` swift
public struct HTTPRequest{
 	private class _Storage{
 		var method: HTTPMethod
 		var target: String
 		var version: HTTPVersion
 		var headers: [(String, String)]
 		var body: [UInt8]?
 		var trailers: [(String, String)]?
 		
 		init(method: HTTPMethod = .GET, target: String, version: HTTPVersion=HTTPVersion(), headers: [(String, String)],body: [UInt8]? , trailers: [(String, String)]?) {
 			//[…]
 		}
 	}
 	private var _storage: _Storage
}
```
``` swift 
extension HTTPRequest{
    public var target: HTTPMethod {
        get{
              return self._storage.target
        }
        set{
             self._storage.target=newValue
        }
    }
}
```
``` swift
//setting a new target on the Skelton code written above.
var swift = HTTPRequest( … )
swift.target = “/hellowww”
let lang=swift
lang.target=“/NIO”

```

Here I am setting the target of swift object. Once I copied the swift object into lang and changed
the target of lang. This results in 2 structs pointing to the same backing storage and it can
really be problematic if we want to mutate one struct but the effects of mutation will be shown in
both the structs. This can be resolved with a copy on write approach by using a
 **isKnownUniquelyReferenced** function that returns true or false. If it’s false or there is only
one backing storage to a struct then we will set the new target.  In case it’s true or there exists
a struct who share the same backing storage with the other struct, we will create a new storage by
copying the backing storage such that each struct have their own separate backing storage.
The isKnownUniquelyReferenced function ensures thread safety as long as the caller does not 
create race condition on the same variable. The copies of the same variable can be safely 
acessed from multiple threads and mutated independently. However, if there is a race condition, then
we’ll get undefined behavior.


###  Struct backed by class with copy-on write
``` swift 
//Implementation:
//Struct backed by class with copy-on write
extension HTTPRequest{
    public var target: HTTPMethod {
        get{ return self._storage.target }
        set{
             if !isKnownUniquelyReferenced(&self._storage) {
                self._storage = self._stroage.copy( )
             }
             self._storage.target = newValue
        }
    }
}
extension HTTPRequest._Storage {
     func copy( ) -> HTTPRequest._Storage {
        return _Storage(method: self.method, target: self.target, version: self.version, 
                   headers: self.headers, body: self.body, trailers: self.trailers)
    }
}
```
This way we achieved the best of both the worlds and obtained results in just 15 nanoseconds. We
got the speed of the classes and the value semantics of structs with covering a corner case
scenario of 2 structs sharing same backing storage. This approach is only applicable for 2 structs
who may or not have same backing storage.
This approach shows how performant copy-on-write is in implementation. We can also cow box all of the
structs for some more performance like this but repeated copy-on-write in our code can give a time
complexity of O(N^2) performance which is actually slow. This is why it isn’t automated in our
swift compiler as well. 

### Drawbacks of Copy-on-Write
One downside of Copy-on-write is that value types like arrays, dictionary and sets maintain a 
reference count for all the number of copies created for that variable. So, whenever a new copy 
is created, the internal reference count is incremented. Updating the reference count is a slow 
operation because it needs to be thread safe and to achieve thread safety, locking mechanisms 
are implemented internally, which have their own performance costs.

Another limitation of copy-on-write can be that they can create accidental copies. Taking scenario of Indices
###Indices:
Indices are wrapper for base collection, start and end index. It keeps reference to the base collection to advance 
the indices. This extra reference can be a performance issue as it would result in unnecessary copies when a 
collection was mutated during iteration. 
However, if our index is an integer type, we can still optimize it using *CountableRange<Index>*

COW can create accidental copies here. For example:
``` swift
final class Empty { }
struct COWStruct {
    var ref = Empty()
    mutating func change() -> String {
        if !isKnownUniquelyReferenced(&ref) {
            return " Copy is made " }
        else{
            return " No Copy is made "
        }
    }
}
```
Now, if we create a container that stores a value, we can either modify the storage property directly, or we can
 access it indirectly through a subscript. When we directly access it, we get the copy-on-write 
 optimization, but when we access it indirectly through a subscript, a copy is made.
 ``` swift
 struct Container<A> { 
    var storage: A 
    subscript(s: String) -> A {
        get { return storage }
        set { storage = newValue }
    }
 }
 var d = Container(storage: COWStruct()) 
 d.storage.change() // No Copy is made - direct access
 d["test"].change() // Copy is made - indirect access through subscript
 
 ```

## [Concurrency and locking performance](https://developer.apple.com/videos/play/wwdc2021/10254/)

A concurrent code can be higher in performance if it doesn’t have common concurrency problems like 
deadlocks, priority inversion, race conditions, data races and contains lesser lock contention pattern. 

### Race Conditions with Semaphores
In a concurrent environment, race condition is quite common if the progrmamer is not careful.
For instance, if there are 2 threads who are trying to access a single resource at the same time, 
then there are chances of race condition. Race conditions can be avoided with the use of semaphores as
semaphores can control the access of the shared resource. If a thread has occupied some resource, then 
semaphore makes sure that the second thread waits until the first thread releases the shared resource 
by using signal() function.

``` swift
protocol Banking{
    func paymentAmount(amount: Double) throws;
}
enum PaymentError: Error {
    case inSufficientAccountBalance
}
var accountBalance=30000.00
struct PayPal: Banking {
    func paymentAmount(amount: Double) throws {
        debugPrint("welcome to paypal payment gateway")
        guard accountBalance > amount else{ throw
            PaymentError.inSufficientAccountBalance
        }
        Thread.sleep(forTimeInterval: Double.random(in: 1...3))
        accountBalance -= amount
    }
    func printMessage(){
        debugPrint("Payment successful through paypal, new account balance=\(accountBalance)")
    }
}
struct Stripe: Banking {
    func paymentAmount(amount: Double) throws {
        debugPrint("welcome to Stripe payment gateway")
        guard accountBalance > amount else{ throw
            PaymentError.inSufficientAccountBalance
        }
        Thread.sleep(forTimeInterval: Double.random(in: 1...3))
        accountBalance -= amount
    }
    func printMessage(){
        debugPrint("Payment successful through Stripe, new account balance=\(accountBalance)")
    }
}
let queue=DispatchQueue(label: "just-a-queue", qos: .utility, attributes: .concurrent)
let semaphore=DispatchSemaphore(value: 1)

queue.async{
    do{
        let paypal=PayPal()
        try paypal.paymentAmount(amount: 10000)
        paypal.printMessage()
    }catch PaymentError.inSufficientAccountBalance {
        debugPrint("Payment failure due to insufficient Account Balance, paypal transaction cancelled")
    }catch{
        debugPrint("Error")
    }
}
queue.async{
    do{
        let stripe=Stripe()
        try stripe.paymentAmount(amount: 25000)
        stripe.printMessage()
    }catch PaymentError.inSufficientAccountBalance {
        debugPrint("Payment failure due to insufficient Account Balance, stripe transaction cancelled")
    }catch{
        debugPrint("Error")
    }
}
```
Here in the above code, we are creating 2 structs who are accessing account balance(shared resource)
asynchronously. And as a result we recieve negative balance as both the structs changed the account 
balance leading to race condition.
```
"welcome to Stripe payment gateway"
"welcome to paypal payment gateway"
"Payment successful through paypal, new account balance=20000.0"
"Payment successful through Stripe, new account balance=-5000.0"
```
By use of semaphores, race condition can be fixed as follows by adding wait() and signal() function in 
the same code:
```
let queue=DispatchQueue(label: "just-a-queue", qos: .utility, attributes: .concurrent)
let semaphore=DispatchSemaphore(value: 1)

queue.async{
    do{
        semaphore.wait()
        let paypal=PayPal()
        try paypal.paymentAmount(amount: 10000)
        paypal.printMessage()
        semaphore.signal()
    }catch PaymentError.inSufficientAccountBalance {
        semaphore.signal()
        debugPrint("Payment failure due to insufficient Account Balance, paypal transaction cancelled")
    }catch{
        semaphore.signal()
        debugPrint("Error")
    }
}
queue.async{
    do{
        semaphore.wait()
        let stripe=Stripe()
        try stripe.paymentAmount(amount: 25000)
        stripe.printMessage()
        semaphore.signal()
        
    }catch PaymentError.inSufficientAccountBalance {
        semaphore.signal()
        debugPrint("Payment failure due to insufficient Account Balance, stripe transaction cancelled")
        
    }catch{
        semaphore.signal()
        debugPrint("Error")
    }
}
```
As a result we get output as follows. Once we have accessed account balance from paypal struct, stripe struct 
fails the guard condtion and prints insufficent account balance.  
```
"welcome to paypal payment gateway"
"Payment successful through paypal, new account balance=20000.0"
"welcome to Stripe payment gateway"
"Payment failure due to insufficient Account Balance, stripe transaction cancelled"
```
### Priority Inversion with Semaphores
Semaphores can avoid race conditions but sometimes it can introduce priority inversion and deadlocks as a consequence.
for example in the following code, high priority tasks runs first and then other tasks are executed.
```
let highPriority=DispatchQueue.global(qos: .userInitiated)
let lowPriority=DispatchQueue.global(qos: .utility)
let defaultPriority=DispatchQueue.global(qos: .default)

let semaphore=DispatchSemaphore(value: 1)

func printEmoji(queue: DispatchQueue, emoji: String){
    queue.async {
        debugPrint("\(emoji) waiting")
        semaphore.wait()
        for i in 0...5{
            debugPrint("\(emoji) \(i)")
        }
        debugPrint("\(emoji) signal")
        semaphore.signal()
    }
}
printEmoji(queue: highPriority, emoji: " 🆘 ")
printEmoji(queue: lowPriority, emoji: " 🟢 ")
printEmoji(queue: defaultPriority, emoji: " 🏁 ")
```
the output is 
```
" 🆘  waiting"
" 🏁  waiting"
" 🟢  waiting"
" 🆘  0"
" 🆘  1"
" 🆘  2"
" 🆘  3"
" 🆘  4"
" 🆘  5"
" 🆘  signal"
" 🏁  0"
" 🏁  1"
" 🏁  2"
" 🏁  3"
" 🏁  4"
" 🏁  5"
" 🏁  signal"
" 🟢  0"
" 🟢  1"
" 🟢  2"
" 🟢  3"
" 🟢  4"
" 🟢  5"
" 🟢  signal"
```
But sometimes it can give unexpected results as in an asynchronous environment, order of execution cant be predicted.
Here in the following code, high priority tasks are executed after default priority. So there is no surety 
that high priority tasks will always be executed first if we use semaphores. 
```
" 🆘  waiting"
" 🟢  waiting"
" 🏁  waiting"
" 🏁  0"
" 🏁  1"
" 🏁  2"
" 🏁  3"
" 🏁  4"
" 🏁  5"
" 🏁  signal"
" 🆘  0"
" 🆘  1"
" 🆘  2"
" 🆘  3"
" 🆘  4"
" 🆘  5"
" 🆘  signal"
" 🟢  0"
" 🟢  1"
" 🟢  2"
" 🟢  3"
" 🟢  4"
" 🟢  5"
" 🟢  signal"
```
The problem with this code is that we have used semaphore in one function and that one function is being
accessed by multiple threads with different quality of services. As a result this can cause priority inversion.
In the worst case scenario, we can encounter deadlocks by using semaphores. Deadlocks can also occur due to 
bad code design or bad implementation in a multithreaded environment. 

### Thread Explosion 
When we submit lots of tasks say 1000 tasks to a concurrent queue, then the code becomes very slow and 
saturates all the threads until all cpu cores are saturated.

Now if a thread gets blocked and still there are more tasks left to execute, the system provides more threads 
to maintain a concurrency level and help release the resources held by the first thread. In complex applications, 
if there are many thread that gets blocked for executing so many tasks then it will result in more number of 
system generated threads than there are cpu cores available. This can hinder performance of our system. 

Each of the blocked thread holds valuable memory and resources with an associated kernel 
data structure to track the thread. This resource overhead can lead to deadlocks and cause 
greater scheduling overhead.   
As more threads are brought up the cpu needs to perform full context switch in order to switch 
form old to new thread and as a result the blocked threads starts executing while the 
scheduler timeshares the thread on the cpu to make forward progress. But time sharing hundreds of threads 
can lead to excessive context switching and lowers the performance with greater overhead.  

for example in the following code, I am deserailzing articles from the data and updating articles into 
my database. If the no of articles are large, then it can take time to update the database and UI 
respectively. This can hinder performance of our app and can really affect the user experience. 
``` swift
func deserializeArticles(from data: Data) throws -> [Article] { … }
func updateDatabase(with articles: [Article], for feed: Feed) { … }
let urlSession = URLSession(configuration: .default, delegate: self, delegateQueue: concurrentQueue)
for feed in feedsToUpdate {
    let dataTask = urlSession.dataTask(with: feed.url) { data, response, error in
        guard let data = data else { return }
        do {
             let articles = try deserializeArticles(from: data)
             databaseQueue.sync {
                 updateDatabase(with: articles, for: feed)
             }
        } catch {
            print("Error during article deserialization: \(error.localizedDescription)")
        }
    }
    dataTask.resume( )
}
```
### Runtime Contract
For good performance, swift concurrency provides ability to switch between continuations 
instead of full context switches. This behaviour is implemented by making a runtime contract 
that threads are always able to make forward progress by use of language features like :

1) await semantics - This provides an asynchronous wait that doesn’t block the current thread 
while waiting from the async functions instead the function maybe suspended and the thread 
will be freed upto execute other tasks. 
2) Runtime tracking of dependancies between tasks - Every continuation is dependent on the runtime of
the async function. Similarly within a task group, a parent task may create several child tasks 
and each of those child tasks needs to complete before a parent task can proceed. This runtime 
is explicitly known to the swift compiler and runtime who helps preserve the contract by help 
of primitives like await, actors and task groups.
This runtime contract helps building a new thread pool that will spawn only as many threads
as there are cpu cores, thereby making sure not to overcommit and hinder performance. This 
way we were able to reduce context switches, eliminate deadlocks and boosted performance. 

Rewriting code with async/await semantics to acheive better performance of our app.
``` swift
func deserializeArticles(from data: Data) throws -> [Article] { … }
func updateDatabase(with articles: [Article], for feed: Feed) async  { … }
await withThrowingTaskGroup(of: [Article].self) { group in
    for feed in feedsToUpdate {
        group.async {
            let (data, response) = try await URLSession.shared.data(from: feed.url)
            let articles = try deserializeArticles(from: data)
            await updateDatabase(with: articles, for: feed)
            return articles
        }
    }
}
```

When we use concurrent queues , we can run into thread explosion often. Hence, in a WWDC talk
about concurrency with Grand Central Dispatch, it was advised to structure applications into
distinct subsystems and maintain one serial dispatch queue per subsystem to control the 
concurrency of the application. As we know, serial queues are concurrent with other queues so 
there is still a performance benefit of offloading work to a queue, even if it’s concurrent.
But it was difficult to get concurrency greater than one within a subsystem without running the
risk of thread explosion. But now, with the new semantics that’s leveraged by the runtime, we can
obtain a safe and controlled concurrency. 
Therefore, it is advised to use concurrent queue, only if there is measurable performance benefit. 

Even multiple concurrent threads where each thread is running sequentially and as long as they
don’t communicate via shared memory, they can obtain desired results but once different 
concurrent threads share memory, the program crashes. To avoid this, we use thread 
synchronisation. 
### Synchronization
It offers 
1) Mutual Exclusion 
2) Reentrancy and Prioritisation
3) Main Actor

### [Mutual Exclusion](https://developer.apple.com/videos/play/wwdc2021/10133/)
Consider an example of updating database with some article by syncing it to a serial queue. 
If the queue is not running, the calling thread is reused to execute the work item on the queue
without any context switch. If the serial queue is running ,the calling thread is blocked. 
This blocking behaviour triggers thread explosion. 

``` swift
databaseQueue.sync { updateDatabase(with: articles, for: feed) }
```
Hence it is advised to use dispatch async that is non blocking, so even under contention it 
will not lead to thread explosion. When there is no contention, dispatch needs to 
request a new thread to do async work while the calling thread continues to do other task. 
But, frequent use of dispatch async can lead to excess thread wake ups and context switches
which can significantly lower the performance. 

``` swift
databaseQueue.async { /*…background work…. */}
```
This brings us to actor who is efficient in contention and non contention cases. Actors 
guarantee *mutual exclusion* such that one work item may be active at a given time which means 
that the actor state is not accessed concurrently preventing data races. Actors are more performant
than semaphores/dispatch barrier solution to avoid data races as actors can eleminate the need
of lock based synchronization. Actors can have generics,methods and properties of a class. 
The only difference between classes and actors is of inheritance. Classes can inherit from 
other classes while actors dont. 
Actors can conform to protocols and be augmented with extensions.
Like classes, they are reference types; because the purpose of actors is to express shared
mutable state.
Actor types isolates their instance data from the rest of the program and ensure synchronized 
access to that data. Structs and actors can eleminate data races but classes dont.  
Data races can occur when two separate threads concurrently access the same data and at 
least one of those accesses is a write. Data races are trivial to construct but are 
notoriously hard to debug.
Data races can be detected through enabling Thread Sanitizer by navigating to 
Product > Scheme > Edit Scheme. After that, in the edit scheme dialog choose 
Run > Diagnostic > and select the Thread Sanitizer checkbox. This setting can improve our
application in avoiding data races but can increase the build time for our application.
### Accessing mutable state within the actor
Consider example:
Here in this code, we are incrementing the count till 1000 times and once dispatch group completes
execution. Count value is displayed on the label. 
```
class Counter {
    var count = 0
    func addCount() {
        count += 1
    }
}
let Count = 1000
let counter = Counter()
let group = DispatchGroup()

// Call addCount() asynchronously 1000 times
for _ in 0..<Count {
    
    DispatchQueue.global().async(group: group) {
        counter.addCount()
    }
}
group.notify(queue: .main) {
    self.Label.text = "\(counter.count)"
}
```
But this yields inconsistent results caused by data races and there is no guarantee that each thread will 
update the count one after another.
As a solution, we have 2 fixes in this code:
1) We can create a parent task that will spawn a group of child tasks executing the addCount() function
 asynchronously and then create a task group within the *withTaskGroup(of:body:)* method. This method will
 suspend tasks until all the child tasks are completed. Doing this we will be able to get consistent
 results.
 ```
 let Count = 1000
 let counter = Counter()

 // Create a parent task
 Task {
    // Create a task group
    await withTaskGroup(of: Void.self, body: { taskGroup in
        for _ in 0..<Count {
            // Create child task
            taskGroup.addTask {
                counter.addCount()
            }
        }
    })
    statusLabel.text = "\(counter.count)"
 }
 ```
2) For avoiding data races, we can use actors to protect its mutable state from both read and write access.
```
let Count = 1000
let counter = Counter()
// Create a parent task
Task {
    // Create a task group
    await withTaskGroup(of: Void.self, body: { taskGroup in
        for _ in 0..<Count {
            // Create child task
            taskGroup.addTask {
                // Marked with await
                await counter.addCount()
            }
        }
    })
    // Marked with await
    statusLabel.text = "\(await counter.count)"
}
```
### Accessing mutable state outside the actor
Actors can help us in preventing data races by ensuring mutual exclusion to its mutable states
as long as we are accessing the mutable states within the actors. But If the mutable states are 
accessible outside of the actors, a data race can still occur!
Consider example:
Here in this code we are incrementing the likeCount inside the actor and decrementing it outside the 
actor. So if we try to run the actor’s like(_:) and the dislike(_:) function concurrently, a 
data race will occur!
```
final class Post {
    let title: String
    var likeCount = 0
    init(title: String) {
        self.title = title
    }
}
actor Person {
    private let posts = [
        Post(title: "Post 1"),
        Post(title: "Post 2"),
        Post(title: "Post 3")
    ]
    /// Increase like count by 1
    func like(_ postTitle: String) {
        guard let post = getPost(with: postTitle) else {
            return
        }
        post.likeCount += 1
    }
    /// Get posts based on post title
    func getPost(with postTitle: String) -> Post? {
        return posts.filter({ $0.title == postTitle }).first
    }
}
let person = Person()
```
Here in the below code, we are creating 2000 child tasks to like an article and 1000 child tasks
to dislike the same article concurrently. In the end, we will get an output 
of “👍🏻 Like count: 2000“
```
let postTitle = "post 1"
// Create a parent task
Task {

    // Create a task group
    await withTaskGroup(of: Void.self, body: { taskGroup in

        // Create 2000 child tasks to like
        for _ in 0..<2000 {
            taskGroup.addTask {
                await self.person.like(postTitle)
            }
        }

        // Create 1000 child tasks to dislike
        for _ in 0..<1000 {
            taskGroup.addTask {
                await self.dislike(postTitle)
            }
        }
    })
    print("👍🏻 Like count: \(await person.getPost(with: postTitle)!.likeCount)")
}
/// Access posts outside of the actor and reduces its like count by 1
func dislike(_ postTitle: String) async {
    guard let post = await person.getPost(with: postTitle) else {
        return
    }
    // Reduce like count
    post.likeCount -= 1
}
``` 
But this code can still show threading issues. The problem with this code is that we have used classes and
because of this we get references of post class into the mutable state of the actor which have been shared 
outside of the class. This way we've created the potential for data races.

As a solution, we can use Sendable protocol to avoid data races. Sendable type is one whose values can be 
shared across different actors. If we copy a value from one place to another, and both places can safely
modify their own copies of that value without interfering with each other, then that type is considered as 
a Sendable type. 
Different kinds of sendable types are:
1) Value types as each copy is independent.
2) Actor types because they synchronize access to their mutable state.
3) Immutable classes Or if the class that internally performs synchronization, for example with a lock, 
to ensure safe concurrent access then it can be a sendable type.

4) @Sendable function types

Rewriting code with structs conforming to Sendable protocol.
```
struct Article: Sendable {

    let title: String
    var likeCount = 0

    init(title: String) {
        self.title = title
    }
}
actor Person {
    private let posts = [
        Post(title: "Post 1"),
        Post(title: "Post 2"),
        Post(title: "Post 3")
    ]

    /// Increase like count by 1
    func like(_ postTitle: String) {

        guard var post = getPost(with: postTitle) else {
            return
        }

        post.likeCount += 1
    }
    /// Get posts based on posts title
    func getPost(with postTitle: String) -> Post? {
        return posts.filter({ $0.title == postTitle }).first
    }
}
let person = Person()

let postTitle = "post 1"

// Create a parent task
Task {

    // Create a task group
    await withTaskGroup(of: Void.self, body: { taskGroup in

        // Create 2000 child tasks to like
        for _ in 0..<2000 {
            taskGroup.addTask {
                await self.person.like(postTitle)
            }
        }

        // Create 1000 child tasks to dislike
        for _ in 0..<1000 {
            taskGroup.addTask {
                await self.dislike(postTitle)
            }
        }
    })

    print("👍🏻 Like count: \(await person.getPost(with: postTitle)!.likeCount)")
}
// Access posts outside of the actor and reduces its like count by 1
func dislike(_ postTitle: String) async {
    guard var post = await person.getPost(with: postTitle) else {
        return
    }

    // Reduce like count
    post.likeCount -= 1
}
```

Data races are similar to race conditions in the sense of accessing the shared resource by multiple
threads in a concurrent environment. The difference between them is that there are no checks or locks
before the resource is accessed in Data races while Race condition follows check and update mechanism
such that it checks for a condition or validation before the resource is updated.
 
### [Reentrancy and Prioritization](https://developer.apple.com/videos/play/wwdc2021/10254/)
Semaphores provides no surety of always executing high priority tasks first. As a solution to this, 
actors can be used.
If we call a method on an actor that is not running, the calling thread can be reused to 
execute the method call. If the called actor is already running, the calling thread 
can suspend the executing function and pick up other work. 

In case there are lot of asynchronous work and specially a lot of contention the system needs to
make trade of based on the task priority. A high priority work such as user interaction would take
precedence over background work of saving backups. Actors are designed to allow the system to
prioritise work well due to notion of reentrancy. 
 

![alt text](https://github.com/shilpeegupta14/images/blob/main/Screenshot%202022-01-17%20at%201.25.47%20AM.png?raw=true)
                
                      Quick recap of contention and non contention cases.
                 
Since actors are designed for reentrancy the runtime may choose to move the higher priority item to
the front of the queue ahead of the lower priority items. This way higher priority work could be
executed first with lower priority work following later. This directly addresses the problem of
priority inversion allowing for more effective scheduling and resource utilisation. 

### [Main Actor](https://developer.apple.com/videos/play/wwdc2021/10133/)
Consider the following code where we have a function update Articles on MainActor, which loads
articles out of the database and updates the UI for each article. Each iteration of the loop
requires at least two context switches: one to hop from the main actor to the database actor and
another to hop back.

``` swift
// on database actor
func loadArticle(with id: ID) async throws -> Article { …}
@MainActor func updateUI(for article: Article) async { .. }
@MainActor func updateArticles(for ids: [ID]) async throws {
    for id in ids{
        let article = try await database.loadArticle(with: id)
        await updateUI(for: article)
    }
}
```

Since each loop iteration requires two context switches, there is a repeating pattern where two
threads run one after another for a short span of time. If the number of loop iterations is low,
and substantial work is being done in each iteration, that is probably all right. However, if
execution hops on and off the main actor frequently, the overhead of switching threads can start
to add up. 

Restructuring the code so that work for the main actor is batched up. We can batch up work by
pushing the loop into the loadArticles and updateUI method calls, making sure they process arrays
instead of one value at a time. Batching up work reduces the number of context switches and help
increase performance by reducing the time complexity of the program.

``` swift
//on database actor
func loadArticles(with ids: [ID]) async throws -> [Article]
@MainActor func updateUI(for articles: [Article]) async 
@MainActor func updateArticles(for ids: [ID]) async throws {
    let articles=try await database.loadArticles(with: ids)
    await updateUI(for: articles)
}
```
Note that swift makes no guaranty that the thread that executed the code before the await is the
same thread which will pick up the continuation as well.
Consider example:
Here in this code we are downloading images in a cache to avoid downlaoding the same image multiple 
times. The code checks the cache, download the image and then record the image in the cache before
returning. Since the code implements actor, this code is free from low-level data races; any number 
of images can be downloaded concurrently.
The actor's synchronization mechanisms guarantee that only one task can execute code that accesses
the cache instance property at a time, so there is no way that the cache can be corrupted.
Whenever await occurs, the function can be suspended at this point. It gives up its CPU so other 
code in the program can execute, which affects the overall program state.
```
actor ImageDownloader {
    private var cache: [URL: Image] = [:]

    func image(from url: URL) async throws -> Image? {
        if let cached = cache[url] {
            return cached
        }

        let image = try await downloadImage(from: url)

        // Potential bug: `cache` may have changed.
        cache[url] = image
        return image
    }
}
```
Consider a scenario where we have two different concurrent tasks trying to fetch the same image
at the same time. The first sees that there is no cache entry, proceeds to start downloading the
image from the server, and then gets suspended because the download will take a while.
While the first task is downloading the image, a new image might be deployed to the server under
the same URL.
Now, a second concurrent task tries to fetch the image under that URL.
It also sees no cache entry because the first download has not finished yet, then starts a second
download of the image.
It also gets suspended while its download completes.
After a while, one of the downloads -- let's assume it's the first -- will complete and its task
will resume execution on the actor. It populates the cache and returns the resulting image of a cat.
Now the second task has its download complete, so it wakes up and overwrites the same entry with the
another image for the same URL.
We don't have any low-level data races, but because we carried assumptions about state across an 
await, we ended up with a potential bug.
The solution for this problem is to check our assumptions after the await. 
Solution 1: If there's already an entry in the cache when we resume, we keep that original version 
and throw away the new one.
```
actor ImageDownloader {
    private var cache: [URL: Image] = [:]

    func image(from url: URL) async throws -> Image? {
        if let cached = cache[url] {
            return cached
        }

        let image = try await downloadImage(from: url)

        // Replace the image only if it is still missing from the cache.
        cache[url] = cache[url, default: image]
        return cache[url]
    }
}
``` 
This can also be solved in a better way through our next approach.
Solution 2: A better solution would be to avoid redundant downloads entirely.
```
actor ImageDownloader {
    private enum CacheEntry {
        case inProgress(Task<Image, Error>)
        case ready(Image)
    }
    private var cache: [URL: CacheEntry] = [:]

    func image(from url: URL) async throws -> Image? {
        if let cached = cache[url] {
            switch cached {
            case .ready(let image):
                return image
            case .inProgress(let task):
                return try await task.value
            }
        }
        let task = Task {
            try await downloadImage(from: url)
        }
        cache[url] = .inProgress(task)
        do {
            let image = try await task.value
            cache[url] = .ready(image)
            return image
        } catch {
            cache[url] = nil
            throw error
        }
    }
}
```
Actor reentrancy prevents deadlocks and guarantees forward progress, but it requires you to check 
your assumptions across each await. To design well for reentrancy, we can perform mutation of
actor state within synchronous code. Ideally, we can do it within a synchronous function so all 
state changes are well-encapsulated. State changes can involve temporarily putting our actor into
an inconsistent state and restoring consistency before an await. Any assumptions programmer've made
about global state, clocks, timers, or an actor will need to be checked after the await.
If the code gets suspended, the program will move on before our code gets resumed.

### ⚠️ Potential Bug
At the await point in our code, also known as suspension point, several tasks are descheduled
which can hold lock across await. Though locks ensure code safety and ensure forward progress 
but persistent lock contention limits performance. Primitives like os_unfair_locks and NSLocks
are safe but compiler doesn’t provide support in correct usage of locks and the programmer 
needs to handle it correctly.
Primitives like semaphores and condition variables are unsafe to use with swift 
concurrency as they hide dependency information from the swift runtime and introduces a 
dependency in code execution.

### Lock Contention Pattern
Lock contention depends upon how often lock is requested and how long it is held once acquired.
If its higher, the processor can sit idle despite of plenty of task provided. Hence it is advised
to use concurrency primitives like await, actors, and task groups, that are made known at compile
time in order to preserve the runtime contract and achieve higher performance.

