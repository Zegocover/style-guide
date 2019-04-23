# useEffect vs onComplete callbacks

Let's say you have a hook which provides a way to call an async operation which takes some length of time

```javascript
const MyComponent = () => {
  const [runTask] = useMyAsyncTask();

  return <Button onClick={runTask}>Click Me</Button>;
};
```

Simple right?

Let's make it more powerful, so it can show the state of the task and a result if one exists

```javascript
const MyComponent = () => {
  const [runTask, { result, loading }] = useMyAsyncTask();

  return (
    <>
      {data && <p>Result: {result}</p>}
      <Button loading={loading} onClick={runTask}>
        Click Me
      </Button>
    </>
  );
};
```

Okay, now we want some side-effect to happen when the task completes, i.e. something other than changing the component's return value. There are two ways to do this, with a callback or with an effect. Let's look at the callback first:

```javascript
const MyComponent = () => {
  const [runTask, { result, loading }] = useMyAsyncTask();

  const myHandler = data => {
    // do something
  };

  return (
    <>
      {data && <p>Result: {result}</p>}
      <Button
        loading={loading}
        onClick={() => runTask({ onComplete: myHandler })}
      >
        Click Me
      </Button>
    </>
  );
};
```

Now the useEffect version:

```javascript
const MyComponent = () => {
  const [runTask, { result, loading }] = useMyAsyncTask();

  // If result has changed from the previous render AND it's not null/undefined, do the thing
  useEffect(() => {
    if (result) {
      // do something
    }
  }, [result]);

  return (
    <>
      {data && <p>Result: {result}</p>}
      <Button loading={loading} onClick={runTask}>
        Click Me
      </Button>
    </>
  );
};
```

So far, these approaches are relatively equivalent and you might use them interchangeably. Let's get a bit more specific.

## Use case 1: Navigation after success

Let's say that if operation was successful, you want to navigate to a new page. How would this look with callbacks?

```javascript
const MyComponent = () => {
  const navigation = useNavigation(); // hypothetical
  const [runTask, { loading }] = useMyAsyncTask();

  const myHandler = data => {
    navigation.push('./success/');
  };

  return (
    <Button
      loading={loading}
      onClick={() => runTask({ onComplete: myHandler })}
    >
      Click Me
    </Button>
  );
};
```

Straightforward enough, but there's potentially a bug here. It's likely (but by no means a rule) that if you navigate somewhere else whilst the task is in flight (perhaps you've gone to the homepage), you won't want the onComplete navigation to occur. With what we have above, the onComplete handler will be called regardless of whether `MyComponent` is still mounted, which isn't what we want.

```javascript
const MyComponent = () => {
  const navigation = useNavigation(); // hypothetical
  const [runTask, { result, loading }] = useMyAsyncTask();

  useEffect(() => {
    if (result) {
      navigation.push('./success/');
    }
  }, [result]);

  return (
    <Button loading={loading} onClick={runTask}>
      Click Me
    </Button>
  );
};
```

With this useEffect version, the effect won't be run if the component isn't still mounted, protecting us from this bug.

## Use case 2: Callback handlers in props

What if we want to trigger a callback handler that's been provided by a parent component (perhaps it updates the state of the parent).

```javascript
const MyComponent = ({ onSuccess }) => {
  const navigation = useNavigation(); // hypothetical
  const [runTask, { loading }] = useMyAsyncTask();

  // Here, when onSuccess is called, it'll be the version of the function
  // from when the onClick handler was created during render. Rather than
  // the version which is correct at the time the async task completes.
  return (
    <Button
      loading={loading}
      onClick={() => runTask({ onComplete: onSuccess })}
    >
      Click Me
    </Button>
  );
};
```

Here we have the same problem as before. If the parent has been unmounted (and by association, this component), the callback will still be triggered, but will most likely break because it's ultimately trying to perform a setState call in a parent component that's no longer mounted. useEffect protects us again.

```javascript
const MyComponent = ({onSuccess}) => {
  const navigation = useNavigation(); // hypothetical
  const [runTask, { result, loading }] = useMyAsyncTask();

  useEffect(() => {
    if (result) {
      // onSuccess is guaranteed to be fine, because it's the version
      of the function from the _current_ render.
      onSuccess(result);
    }
  }, [result]);

  return (
      <Button
        loading={loading}
        onClick={runTask)}
      >
        Click Me
      </Button>
  );
};
```

So is it always preferable to use the useEffect approach instead of an onComplete callback? Not necessarily, if you want something to happen that doesn't care about the state of the component tree at the moment it triggered, it's probably need to use a callback. An example of this might be analytics tracking or triggering some kind of global notification or state update) where you can be relatively confident it'll always work. But if you're calling a function provided as a prop, there's no way to guarantee that it's safe to call asynchronously, if you want to be certain that you're avoiding bugs, useEffect is the way to go.

The rule of thumb should be to handle the result of async tasks in `useEffect`, breaking the rule only in circumstances where you're sure it's okay to do so.
