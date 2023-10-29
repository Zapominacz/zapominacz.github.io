---
title: "Injectable Task"
date: 2023-08-01T22:07:20+01:00
authors: ["Kinga Wilczek"]
draft: true
---
# [Async/Await] Injectable Task 

Async/Await is the new bright tool given by Apple in order to make our asynchronous code easier to write and understand. Many of us wanted to integrate it into the project but mostly we are doing it in the already existing projects with so called legacy code. To introduce new technology we usually need to bridge all code with refactored or new places. In case of Async/Await at some point we need to use Task initializator `Task { }` in order to connect synchronous code with asynchronous one. The best is to picture this in terms of example. So let’s introduce an environment. 

```Swift
typealias Items = [Item]

struct Item: Equatable {}

protocol ItemProviding {
    func items() async -> Items
}

struct ItemService: ItemProviding {
    func items() async -> Items {
        return Array(repeating: Item(), count: 10)
    }
}

final class StoreState {
    private(set) var items: Items = []

    private let itemService: ItemProviding

    init(itemService: ItemProviding) {
        self.itemService = itemService
    }

    // Cannot be made async because that part of the app is not ready yet
    func update() {
        // causes error
        items = itemService.items()
    }
}
```

We have some newly created feature which is about to provide us some items, we decided to make it using Async/Await and use along with legacy code (`StoreState`). But the code listed above doesn't compile because we cannot call directly `async` code from non-async env. So the first idea that comes to mind is to use that mentioned `Task` initializer. Let's see that in code. 

```Swift
final class StoreState {
    private(set) var items: Items = []

    private let itemService: ItemProviding

    init(itemService: ItemProviding) {
        self.itemService = itemService
    }

    func update() {
        Task {
            self.items = await itemService.items()
        }
    }
}
```
Let's make a small stop here because some `uważne oko po angielsku do sprawdzenia` may spot that there is one hidden thing which is strong self caputre. And what is the thing with this `self`? 🤔 Does it leak or not? Because we're usually using `weak self` in closures it would be considered dangerous. To be honest it doesn't leak because it has to finish at some point even with error **BUT** do we want to keep it till finished if `StoreState` is not needed anymore? it's up to us when you load something what can affect data in other shared structures be aware that it will perform and after that releases the instance so if you don't know probably it's safer to put here `[weak self]` - so just choose wisely.

But despite of that consideration you may as well ask why it doesn't inform us about self usage and in order to answer that question we need to delve into the Swift code, specifically this [file](
https://github.com/apple/swift/blob/0fe2d7b003d3c7405ba011d3a3f3ad05eaf8071a/stdlib/public/BackDeployConcurrency/Task.swift#L479). The pice of code we're looking for and which is responsible for this behaviour is `@_implicitSelfCapture`. As [the documentation]((https://github.com/apple/swift/blob/main/docs/ReferenceGuides/UnderscoredAttributes.md#_implicitselfcapture)) says: 
> **allows access to `self` inside a closure without explicitly capturing it, even when `Self` is a reference type** 

In other words it captures `self`, doesn't inform us about it even if it's reference type (`class` - TBD: are there any other reference types?????). 

What's even more interesting you may use it in your code to test this options and play with them but the documentation says that:
>**WARNING: This information is provided primarily for compiler and standard library developers. Usage of these attributes outside of the Swift monorepo is STRONGLY DISCOURAGED.**

Enough is enough so let's back on track, we did it, perfectl, works, of course now we want to ensure that nobody breaks our logic with tests. But before we do this let's prepare mocks.

```Swift
final class ItemServiceMock: ItemProviding {
    var fakeItems: Items = []
    private(set) var itemsCalled = false
    
    func items() async -> Items {
        itemsCalled = true
        return fakeItems
    }
}
```
Quite easy part of code, just mocking the protocol (Name it properly as not sure whether it is a mock or not, fake, spy, etc.) And now we're ready to do some really simple tests.

```Swift
final class StoreStateTests: XCTestCase {
    private var itemService: ItemServiceMock!
    private var systemUnderTest: StoreState!
    
    override func setUp() {
        super.setUp()
        
        itemService = ItemServiceMock()
        systemUnderTest = StoreState(itemService: itemService)
    }
    
    func testItemsOnUpdate() {
        XCTAssertEqual(systemUnderTest.items, [])
        let expectedItems = Array(repeating: Item(), count: 3)
        itemService.fakeItems = expectedItems
        
        systemUnderTest.update()
        
        XCTAssertEqual(systemUnderTest.items, expectedItems)
    }
}
```

But what if we run them many times, do they always pass? It turns out that not always, so what can we do? We can add an additional `Yield` and see if this helps.

```Swift
final class StoreStateTests: XCTestCase {
    private var itemService: ItemServiceMock!
    private var systemUnderTest: StoreState!
    
    override func setUp() {
        super.setUp()
        
        itemService = ItemServiceMock()
        systemUnderTest = StoreState(itemService: itemService)
    }
    
    func testItemsOnUpdate() async {
        XCTAssertEqual(systemUnderTest.items, [])
        let expectedItems = Array(repeating: Item(), count: 3)
        itemService.fakeItems = expectedItems
        
        systemUnderTest.update()
        await Task.yield()
        
        XCTAssertEqual(systemUnderTest.items, expectedItems)
    }
}
```
Ok weird but works, but what happen if we put there more tests 🤔

```Swift
final class StoreStateTests: XCTestCase {
    private var itemService: ItemServiceMock!
    private var systemUnderTest: StoreState!
    
    override func setUp() {
        super.setUp()
        
        itemService = ItemServiceMock()
        systemUnderTest = StoreState(itemService: itemService)
    }
    
    func testItemsOnUpdate() async {
        XCTAssertEqual(systemUnderTest.items, [])
        let expectedItems = Array(repeating: Item(), count: 3)
        itemService.fakeItems = expectedItems
        
        await Task.yield()
        
        systemUnderTest.update()
        
        XCTAssertEqual(systemUnderTest.items, expectedItems)
    }
    
    func testItemsCountOnUpdate() async {
        XCTAssertEqual(systemUnderTest.items, [])
        let expectedCount = 3
        let expectedItems = Array(repeating: Item(), count: expectedCount)
        itemService.fakeItems = expectedItems
        
        await Task.yield()
        
        systemUnderTest.update()
        
        XCTAssertEqual(systemUnderTest.items.count, expectedCount)
    }
    
    func testItemsUpdateShouldCall() async {
        XCTAssertEqual(systemUnderTest.items, [])
        let expectedItems = Array(repeating: Item(), count: 3)
        itemService.fakeItems = expectedItems
        
        await Task.yield()
        
        systemUnderTest.update()
        
        XCTAssertTrue(itemService.itemsCalled)
    }
}
```

We've added ... TBD

Whooops it failes, what's more each time we run it the number of tests which are passing is different so we ended up with flaky tests, what could we potentially do? 🤔 Of course we could proceed with refactoring and refactor fully app to have it everywhere but not always it's possible especially during migration.
Ok let's start with asking the question is there any way to force result from Task {} and therefore be able to await it? 🤔 Let's check documentation if there is anything what could help us.
Indeed it has value(https://developer.apple.com/documentation/swift/task/value-40dtq) or result(https://developer.apple.com/documentation/swift/task/result)
Both of them are async so we can await them but one returns only success wherheas the latter return info no matter whether error or success. We know that our case doesn't need error so we could potentially use just value. But in the end we have two options.
Ok so far so good, we know what we can await till the whole task finishes so now the thing we could do is to pass it as an argument to the constructor in order to be able to call either result or value and await it.
before we jump right into code let's see what Task constructor hides under the hood
`@discardableResult init(priority: TaskPriority? = nil, operation: @escaping () async -> Success)`
in addition to that we need to know the full type of Task which is
`@frozen struct Task<Success, Failure> where Success : Sendable, Failure : Error`

```Swift
typealias DefaultPriorityTaskFactory<Success, Error: Swift.Error> = (@Sendable @escaping () async -> Success) -> Task<Success, Error>

enum Step5 {
    final class StoreState {
        private(set) var items: Items = []
        
        private let itemService: ItemProviding
        private let taskFactory: DefaultPriorityTaskFactory<Void, Never>
        
        init(itemService: ItemProviding,
             taskFactory: @escaping DefaultPriorityTaskFactory<Void, Never> = { .init(operation: $0) }) {
            self.itemService = itemService
            self.taskFactory = taskFactory
        }
        
        func update() {
            // How to deal with `_ =`
            _ = taskFactory { [weak self] in
                self?.items = await self?.itemService.items() ?? []
            }
        }
    }
}
```

And now we're ready to improve our tests. 

```Swift
final class StoreStateTests: XCTestCase {
    private var itemService: ItemServiceMock!
    private var systemUnderTest: StoreState!
    private var task: Task<Void, Never>!
    
    override func setUp() {
        super.setUp()
        
        itemService = ItemServiceMock()
        systemUnderTest = StoreState(itemService: itemService,
                                        taskFactory: { [weak self] in
            let task = Task(operation: $0)
            self?.task = task
            
            return task
        })
    }
    
    func testItemsOnUpdate() async {
        XCTAssertEqual(systemUnderTest.items, [])
        let expectedItems = Array(repeating: Item(), count: 3)
        itemService.fakeItems = expectedItems
        
        systemUnderTest.update()
        
        await task.value
        
        XCTAssertEqual(systemUnderTest.items, expectedItems)
    }
    
    func testItemsCountOnUpdate() async {
        XCTAssertEqual(systemUnderTest.items, [])
        let expectedCount = 3
        let expectedItems = Array(repeating: Item(), count: expectedCount)
        itemService.fakeItems = expectedItems
                    
        systemUnderTest.update()
        
        await task.value
        
        XCTAssertEqual(systemUnderTest.items.count, expectedCount)
    }
    
    func testItemsUpdateShouldCall() async {
        XCTAssertEqual(systemUnderTest.items, [])
        let expectedItems = Array(repeating: Item(), count: 3)
        itemService.fakeItems = expectedItems
                    
        systemUnderTest.update()
        
        await task.value
        
        XCTAssertTrue(itemService.itemsCalled)
    }
}
```

And now our tests are 100% stable!!!
