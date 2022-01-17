
# High Performance systems in Swift

This contains:

Class Vs Struct tradeoffs

Copy on Write

Concurrency and Locking





## Class Vs Struct tradeoff


For a better tradeoff, lets first consider the differences between classes and struct:

Classes have reference semantics and structs have value semantics. We use structs as values and
classes as objects with identity. We use structs when comparing instance data with == is required
and classes when we compare instance identity with ===. Classes can hold values with same
reference in the memory and can undergo mutation while structs dont.  

Structs use stack memory allocation while classes rely on heap memory allocation. Stack allocation
like insertion, deletion etc are faster than heap that stores different reference types and
manages reference count of that object. Hence, structs have faster memory allocation than classes. 
But gradually, as the number of instances increases in the stack, it becomes difficult to copy
large value types and structs becomes slower. To reduce the complexity of copying large value types, 
copy on write(CoW) is used from the swift standard library.
With copy on write behaviour, we dont need to create copies of the variable when its passed, 
rather only creates the copy when the objects changes its value. Eg- we have an array of 10 
million data that is being passed but a copy will only be produced if the array changes.

Thus, the CoW is performant with struct here.
Consider example:



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
public 	enum HTTPMethod: String{
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
```swift 
public func transform(_ httpRequest: HTTPRequest)->HTTPRequest{
	return httpRequest
} 
_ = transform(httpRequest)

```





 The transform function takes 52 nano seconds to run.

However, if I use a class HTTPRequest instead of struct HTTPRequest. It gives a result in 
15 nano seconds which seems quite performant ideally. But we loose upon the value semantics.
As we change the target of lang, the swift target will also be changed. This can lead to 
threading problems and cause memory leaks which ain’t performant.
In such cases we can use struct backed by class. This way we can gain the value semantics 
of the struct and faster results from the classes.



## 
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
var a = HTTPRequest( … )
 a.target = “/helloowww”
```


This can set the results and obtain a one-to-one relationship between a single struct and
a storage. But this isn’t the only way to set a new target. One can also do like this:

## 
``` swift
var a = HTTPRequest( … )
let b = a
a.target = “bye”
```

This can result in 2 structs pointing to the same backing 
storage and it can really be problematic if we want to mutate 
one struct but the effects of mutation will be shown in both 
the structs. This can be resolved with a copy on write approach
by using a **isKnownUniquelyReferenced** function that returns 
true or false. It returns true if the object have a strong 
reference, otherwise false. If its true or there is only one 
backing storage to a struct then we will set the new target.  
In case its false or there exists a struct who share the same
backing storage with the other struct, we will create a new 
storage by copying the backing storage such that each struct will
have their own separate backing storage.


*However, this approach is only applicable for 2 structs sharing the same backing storage.*


## 
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
This way we achieved the results in just 15 nanoseconds by 
implementing the speed of the classes and the value semantics 
of structs with covering an edge case scenario of 2 structs 
sharing same backing storage.
Struct backed by class with CoW shows how performant copy-on-write is. We can also CoW 
box all of the structs for some more performance like this but 
repeated copy-on-write can gives a time complexity of O(N^2) 
performance which is actually slower. This is why it isn’t 
automated in our swift compiler as well. 

## Concurrency and locking performance
A concurrency model is high in performance if it have fewer 
context switches or doesn’t have concurrency problems like 
deadlocks, priority inversion, race conditions or contains 
lesser lock contention pattern. 


In GCD, when we submit tasks to a concurrent queue, the system 
brings up threads until all cpu cores are saturated. Once a 
thread gets blocked and still there are more tasks left to 
execute, the system provides more threads to maintain a 
concurrency level and help release the resources held by the 
first thread. In complex applications, if there are many thread 
that gets blocked for tasks like accessing database queue or 
other. It will result in more number of system generated threads 
than there are cpu cores available. This is called thread 
explosion. 
Each of the blocked thread holds valuable memory and resources with an associated kernel 
data structure to track the thread. This resource overhead can lead to deadlocks and cause 
greater scheduling overhead. Thread explosion comes with memory and scheduling overhead.  
As more threads are brought up the cpu needs to perform full context switch in order to switch 
form old to new thread and as a result the blocked threads starts executing while the 
scheduler timeshares the thread on the cpu to make forward progress. But as a result of thread 
explosion, time sharing hundreds of threads leads to excessive context switching and 
lowers the performance with greater overhead.  

## 
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
		} catch { /* … */}
	}
	dataTask.resume( )
}
```
For good performance, swift concurrency provides ability to switch between continuations 
instead of full context switches. This behaviour is implemented by making a runtime contract 
that threads are always able to make forward progress by use of language features like :

await semantics - This provides an asynchronous wait that doesn’t block the current thread 
while waiting from the async functions instead the function maybe suspended and the thread 
will be freed upto execute other tasks. 
Runtime tracking of dependancies between tasks - Every continuation is dependent on the runtime of
the async function. Similarly within a task group, a parent task may create several child tasks 
and each of those child tasks needs to complete before a parent task can proceed. This runtime 
is explicitly known to the swift compiler and runtime who helps preserve the contract by help 
of primitives like await, actors and task groups. 

``` swift
//Swift concurrency using task group
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
This runtime contract helps building a new thread pool that will spawn only as many threads
as there are cpu cores, thereby making sure not to overcommit and hinder performance. This 
way we were able to reduce context switches, eliminate deadlocks and boosted performance. 

When we use concurrent queues , we might run into thread explosion often. Hence, in a WWDC talk
about concurrency with Grand Central Dispatch, it was advised to structure applications into
distinct subsystems and maintain one serial dispatch queue per subsystem to control the 
concurrency of the application. As we know, serial queues are concurrent with other queues so 
there is still a performance benefit of offloading work to a queue, even if it’s concurrent.
But it was difficult to get concurrency greater than one within a subsystem without running the
risk of thread explosion. Now, with the new semantics that’s leveraged by the runtime, we can
obtain a safe and controlled concurrency. 
Therefore, it is advised to use concurrent queue, only if there is measurable performance benefit. 

Even multiple concurrent threads where each thread is running sequentially and as long as they
don’t communicate via shared memory, they can obtain desired results but once different 
concurrent threads share memory, the program crashes. To avoid this, we use thread 
synchronisation. Synchronization offers 
1) Mutual Exclusion 
2) Reentrancy and Prioritisation
3) Main Actor

Consider an example of updating database with some article by syncing it to a serial queue. 
If the queue is not running, the calling thread is reused to execute the work item on the queue
without any context switch. If the serial queue is running ,the calling thread is blocked. 
This blocking behaviour triggers thread explosion. 

``` swift
databaseQueue.sync { updateDatabase(with: articles, for: feed) }
```
Hence it is advised to use dispatch async that is non blocking, so even under contention it 
will not lead to thread explosion. However when there is no contention, dispatch needs to 
request a new thread to do async work while the calling thread continues to do other task. 
However, frequent use of dispatch async can lead to excess thread wake ups and context switches
which can significantly lower the performance. 

``` swift
databaseQueue.async { /*…background work…. */}
```
This brings us to actor who is efficient in contention and non contention cases. Actors 
guarantee mutual exclusion such that one work item may be active at a given time which means 
that the actor state is not accessed concurrently preventing data races. Actors can be 
considered as a class with some private state and an associated serial dispatch queue. 
When we call a method on an actor that is not running, the calling thread can be reused to 
execute the method call. In case where the called actor is already running, the calling thread 
can suspend the executing function and pick up other work. 

In case there are lot of asynchronous work and specially a lot of contention the system needs to
make trade of based on the task priority. A high priority work such as user interaction would take
precedence over background work of saving backups. Actors are designed to allow the system to
prioritise work well due to notion of reentrancy. 


## 

![alt text](https://github.com/shilpeegupta14/images/blob/main/Screenshot%202022-01-17%20at%201.25.47%20AM.png?raw=true)
	            
				      Quick recap of contention and non contention cases.
 

Consider  a news application with serial database queue. Suppose the database receive a high
priority work of fetching latest data to update the ui and low priority work of backing up the
database to iCloud.

``` swift
databaseQueue.async{ fetchLatestForDisplay() }.     //user initiated
```
``` swift
databaseQueue.async { backUpToiCloud() }             //background
```
The database queue looks like this:

![alt text](https://github.com/shilpeegupta14/images/blob/main/Screenshot%202022-01-17%20at%201.35.59%20AM.png?raw=true)

As the code runs new work items are created in interleaved order. Dispatch queue execute the 
items received in a strict first in first out order. Unfortunately this means that after item A 
has executed 5 lower priority items, it can then execute the next high priority item.  This is
called a priority inversion. Serial queue work around priority inversion by boosting the priority
of all the work in the queue that is ahead of the high priority work In practice this means that
the work in the queue will be done sooner. 

Solving this issue require changing the semantic model away from first in first out which brings
actor reentrancy. Actor Reentrancy means new work items on a actor can make progress while one or
more older work items on it are suspended.

``` swift
await database.fetchLatestForDisplay()         //user initiated
```
``` swift
await database.backUpToiCloud()                  //background
```
![alt text](https://github.com/shilpeegupta14/images/blob/main/Screenshot%202022-01-17%20at%201.40.02%20AM.png?raw=true)
               
			        Fig-Task A has high priority so it is executed first.
![alt text](https://github.com/shilpeegupta14/images/blob/main/Screenshot%202022-01-17%20at%201.41.32%20AM.png?raw=true)

 				Fig-After task B is executed, all the lower priority items are executed.
                 
Since actors are designed for reentrancy the runtime may choose to move the higher priority item to
the front of the queue ahead of the lower priority items. This way higher priority work could be
executed first with lower priority work following later. This directly addresses the problem of
priority inversion allowing for more effective scheduling and resource utilisation.

Consider the following code where we have a function update Articles on MainActor, which loads
articles out of the database and updates the UI for each article. Each iteration of the loop
requires at least two context switches: one to hop from the main actor to the database actor and
one to hop back.

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
same thread which will pick up the continuation as well. At the await point in our code, several
tasks are descheduled which can hold lock across await. Though locks ensure code safety and ensure
forward progress but persistent lock contention limits performance. Primitives like os_unfair_locks
and NSLocks are safe but compiler doesn’t provide support in correct usage of locks and the
programmer needs to handle it correctly. Primitives like semaphores and condition variables are
unsafe to use with swift concurrency as they hide dependency information from the swift runtime but
introduce a dependency in code execution.

Lock contention depends upon how often lock is requested and how long it is held once acquired.
If its higher, the processor can sit idle despite of plenty of task provided. Hence it is advised
to use concurrency primitives like await, actors, and task groups, that are made known at compile
time in order to preserve the runtime contract and achieve higher performance.

