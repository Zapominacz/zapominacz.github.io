---
title: "Dealing with Memory Leaks in JavaScript: Closures"
author: "Mikołaj Wilczek"
date: 2024-08-15T21:00:00+02:00
---

Recently, I stumbled upon a couple of insightful blog posts that delve into a subtle yet significant issue in JavaScript: memory leaks caused by nested functions and closures. This topic is crucial, if you're a React developer as it explains the drawbacks in `useCallback` hooks.

## Original example

I was reading [Jake Archibald's post](https://jakearchibald.com/2024/garbage-collection-and-closures/) where he discusses how nested functions can inadvertently retain references, leading to memory leaks. The following code snippet illustrates this scenario:

```javascript
function demo() {
  const bigArrayBuffer = new ArrayBuffer(100_000_000);
  const id = setTimeout(() => {
    console.log(bigArrayBuffer.byteLength);
  }, 1000);

  return () => clearTimeout(id);
}

globalThis.cancelDemo = demo();
```

In this example, `bigArrayBuffer` is captured, because it is referenced in the closure inside the function. [Kevin Schiener](https://schiener.io/2024-03-03/react-closures) have a litte more practical example in React:

```javascript
import { useState, useCallback } from "react";

class BigObject {
  data = new Uint8Array(1024 * 1024 * 10);
}

export const App = () => {
  const [countA, setCountA] = useState(0);
  const [countB, setCountB] = useState(0);
  const bigData = new BigObject(); // 10MB of data

  const handleClickA = useCallback(() => {
    setCountA(countA + 1);
  }, [countA]);

  const handleClickB = useCallback(() => {
    setCountB(countB + 1);
  }, [countB]);

  // This only exists to demonstrate the problem
  const handleClickBoth = () => {
    handleClickA();
    handleClickB();
    console.log(bigData.data.length);
  };

  return (
    <div>
      <button onClick={handleClickA}>Increment A</button>
      <button onClick={handleClickB}>Increment B</button>
      <button onClick={handleClickBoth}>Increment Both</button>
      <p>
        A: {countA}, B: {countB}
      </p>
    </div>
  );
};
```

Using the Memory tab and taking the heap snapshots, we can conclude that:
1. An object is retained if at least one closure references to it.
2. For every rerender, there will be more left BigObjects
3. Fortunatelly in this case, Chrome keeps them at an acceptable level (I cannot generate more than 62 MB of heap size), but I wouldn't take it for granted as there is a lot of unpredactibility in it.
 
## How Can We Deal with Memory Leaks? Approach #1 - WeakMap

This got me thinking about how to manage these scenarios, particularly when large objects need to be recreated on every re-render. In cases where inner functions are necessary for code readability or architecture, it's vital to prevent unnecessary memory retention.

I experimented with using `WeakMap` to manage these references, hoping it would help with garbage collection:

```javascript
const Home = () => {
  const [countA, setCountA] = useState(0);
  const [countB, setCountB] = useState(0);
  const weakMap = new WeakMap();
  let key = {}; // unique key
  useEffect(() => {
    return () => {
      console.log("cleanup");
      key = null;
      weakMap.delete(key);
    };
  }, [])
  weakMap.set(key, new BigObject())

  const handleClickA = useCallback(() => {
    setCountA(countA + 1);
  }, [countA]);

  const handleClickB = useCallback(() => {
    setCountB(countB + 1);
  }, [countB]);

  // This only exists to demonstrate the problem
  const handleClickBoth = () => {
    handleClickA();
    handleClickB();
    console.log(weakMap.get(key).data.length);
  };

  return (
    <div>
      <button onClick={handleClickA}>Increment A</button>
      <button onClick={handleClickB}>Increment B</button>
      <button onClick={handleClickBoth}>Increment Both</button>
      <p>
        A: {countA}, B: {countB}
      </p>
    </div>
  );
};
```

However, I found that even with `WeakMap`, the keys were still retained due to the closure. This meant the potential memory leak persisted.

## Approach #2: WeakRef

Another approach is using `WeakRef`, which allows for weak references that don’t prevent garbage collection:

```javascript
import { useState, useCallback } from "react";

class BigObject {
  data = new Uint8Array(1024 * 1024 * 10);
}
const Home = () => {
  const [countA, setCountA] = useState(0);
  const [countB, setCountB] = useState(0);
  const bigObject = new BigObject();
  const weakRef = new WeakRef(bigObject);
  const handleClickA = useCallback(() => {
    setCountA(countA + 1);
  }, [countA]);

  const handleClickB = useCallback(() => {
    setCountB(countB + 1);
  }, [countB]);

  // This only exists to demonstrate the problem
  const handleClickBoth = () => {
    handleClickA();
    handleClickB();
    console.log(weakRef.deref()?.data.length);
  };

  return (
    <div>
      <span>{bigObject.data.length}</span>
      <button onClick={handleClickA}>Increment A</button>
      <button onClick={handleClickB}>Increment B</button>
      <button onClick={handleClickBoth}>Increment Both</button>
      <p>
        A: {countA}, B: {countB}
      </p>
    </div>
  );
};
```

In this case, after clicking, it could display `undefined` or `10 485 760`. If you click after rerender and before GC will free the memory, you will get `10 485 760`. As we see, there is no place where the strong reference is used, so it will disappear after the render cycle.

## Approach #3: Manual Cleanup with useEffect

The most effective solution I discovered was to move references higher up in the component hierarchy or even to a global scope. By doing this, I could manually control cleanup using React’s `useEffect` hook, ensuring that large objects are properly dereferenced. We track object allocations in a dictionary and manually nullify the reference during cleanup. This approach ensures that each instance is correctly dereferenced, preventing memory leaks. This case assumes there is more than one reference to this component in the code, and every one of them needs a separate, individual `BigObject`. The biggest downside of this solution is the need for a unique key for every component reference.

```javascript
class BigObject {
  data = new Uint8Array(1024 * 1024 * 10);
}

// out of the scope
const bigObjects = {};
const uniqueKey = "unique"; // can be index, context, etc.

const Home = () => {
  const [countA, setCountA] = useState(0);
  const [countB, setCountB] = useState(0);
  useEffect(() => {
    return () => {
      bigObjects[uniqueKey] = null;
    };
  }, [])
  bigObjects[uniqueKey] = new BigObject();

  const handleClickA = useCallback(() => {
    setCountA(countA + 1);
  }, [countA]);

  const handleClickB = useCallback(() => {
    setCountB(countB + 1);
  }, [countB]);

  // This only exists to demonstrate the problem
  const handleClickBoth = () => {
    handleClickA();
    handleClickB();
    console.log(bigObjects[uniqueKey].data.length);
  };

  return (
    <div>
      <button onClick={handleClickA}>Increment A</button>
      <button onClick={handleClickB}>Increment B</button>
      <button onClick={handleClickBoth}>Increment Both</button>
      <p>
        A: {countA}, B: {countB}
      </p>
    </div>
  );
};
```

## Conclusion

Dealing with memory leaks in JavaScript, can require a nuanced approach. Sometimes, we might need to manage memory manually by nullifying variables. The key is finding a balance that ensures performance without sacrificing code readability.

I've created an [example repository](https://github.com/Zapominacz/js-gc-examples) with working version of this solution. Feel free to explore the code, and if you have thoughts or questions, I encourage you to use the issues section.

If you have any comments regarding this post (or previous), feel free to [submit an issue](https://github.com/Zapominacz/zapominacz.github.io/issues/new).

### Further Reading:
- [Garbage Collection and Closures by Jake Archibald](https://jakearchibald.com/2024/garbage-collection-and-closures/)
- [React Closures by Philipp Schiener](https://schiener.io/2024-03-03/react-closures)
