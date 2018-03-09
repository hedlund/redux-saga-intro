class: center, middle

# An extremely short intro to Redux-Saga

---

# ES6 Generators

* Functions that can be paused & resumed
* Defined using `function*` (note the asterix)
* You `yield` values to the caller,
* Values can be passed back when resuming

---

# Generator example

The generator:

```js
function *foo() {
    yield 1
    yield 2
    yield 3
}
```

Usage:

```js
const it = foo();       // hasn't actually executed the generator
let next = it.next();   // { value: 1, done: false }
next = it.next();       // { value: 2, done: false }
next = it.next();       // { value: 3, done: false }
next = it.next();       // { value: undefined, done: true }
```

---

# Looping

Stupid example:

```js
const it = foo();
for (let next = it.next(); !next.done; next = it.next()) {
    console.log(next.value);
}
// 1 2 3
```

Better example, using as an iterator:

```js
for (let v of foo()) {
    console.log(v);
}
// 1 2 3
```

---

# Passing values

```js
function *foo(x) {
    const y = 2 * (yield (x + 1));
    const z = yield (y / 3);
    return (x + y + z);
}
```

*Actually you should avoid returning values, as they "disappear" when using iterators...*

```js
const it = foo(5);
it.next();          // { value: 6, done: false }
it.next(12);        // { value: 8, done: false }
it.next(13);        // { value: 42, done: true }
```

---

# So what is it good for?

Math stuff (Fibonacci series, etc)

Combine with *promises* and you can write async code really nicely (fake `async` / `await`): [tj/co](https://github.com/tj/co)

Used extensively by [Redux-Saga](https://github.com/redux-saga/redux-saga).


---

# Redux-Saga

*An alternative side effect model for Redux apps*

It's a Redux middleware that uses generators to handle asynchronous code, such as calling API's.

```js
// ...
import { createStore, applyMiddleware } from 'redux'
import createSagaMiddleware from 'redux-saga'

// ...
import { helloSaga } from './sagas'

const sagaMiddleware = createSagaMiddleware()
const store = createStore(
  reducer,
  applyMiddleware(sagaMiddleware)
)
sagaMiddleware.run(helloSaga)

const action = type => store.dispatch({type})

// rest unchanged
```

---

# Redux Counter example

```js
const Counter = ({ value, onIncrement, onDecrement, onIncrementAsync }) =>
  <div>
    <button onClick={onIncrementAsync}>
      Increment after 1 second
    </button>

    <button onClick={onIncrement}>
      Increment
    </button>

    <div>
      Clicked: {value} times
    </div>
  </div>
```

```js
function render() {
  ReactDOM.render(
    <Counter
      value={store.getState()}
      onIncrement={() => action('INCREMENT')}
      onIncrementAsync={() => action('INCREMENT_ASYNC')} />,
    document.getElementById('root')
  )
}
```

---

# Counter saga

```js
import { delay } from 'redux-saga'
import { put, takeEvery } from 'redux-saga/effects'

// Our worker Saga: will perform the async increment task
export function* incrementAsync() {
  yield delay(1000)
  yield put({ type: 'INCREMENT' })
}

// Our watcher Saga: spawn a new incrementAsync task on each INCREMENT_ASYNC
export function* watchIncrementAsync() {
  yield takeEvery('INCREMENT_ASYNC', incrementAsync)
}
```

---

# Saga Effects

Plain old JavaScript Objects that describe the logic of the Saga. Part of the Saga protocol that makes the generators work for us...

* `put` - Dispatch an action
* `delay` - Delay execution
* `takeEvery` - Take every action and do something
* `takeLatest` - Only take the latest action and do something
* `take` - Wait for an action
* `call` - Call a function
* `select` - Select data from the Redux store
* ...

---

# Calling an API

```js
import Api from './path/to/api'
import { call, put } from 'redux-saga/effects'

function* fetchProducts() {
  try {
    const products = yield call(Api.fetch, '/products')
    yield put({ type: 'PRODUCTS_RECEIVED', products })
  }
  catch(error) {
    yield put({ type: 'PRODUCTS_REQUEST_FAILED', error })
  }
}

export function* productSaga() {
  yield takeEvery('PRODUCTS_REQUEST', fetchProducts)
}
```

---

# Combining with a reducer...

```js
const actionsMap = {
  PRODUCTS_REQUEST: (state, action) => ({
      isLoading: true,
      ...state
  }),
  PRODUCTS_RECEIVED: (state, { products }) => ({
      isLoading: false,
      products
  }),
  PRODUCTS_REQUEST_FAILED: (state, { error }) => ({
      isLoading: false,
      error
  })
};
```

---

# ...and a React Component

```js
class Products extends Component {

    componentDidMount() {
        if (!this.props.products) {
            this.props.fetchProducts();
        }
    }

    render() {
        const { isLoading, products, error } = this.props;
        return (
            <IsLoading isLoading={ isLoading }>
                <IsError error={ error }>
                    <ul>
                        { products && products.map(p => <li>{p.name}</li>)}
                    </ul>
                </IsError>
            </IsLoading>
        )
    }
}
```

---

# Testing

```js
import { call, put } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// expects a call instruction
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products')
)

const products = [
    { name: 'Foo' },
    { name: 'Bar' },
];

// expects a products dispatch
assert.deepEqual(
    iterator.next(products).value,
    put({ type: 'PRODUCTS_RECEIVED', products })
)

// expects to be done
assert.deepEqual(
    iterator.next(),
    { value: undefined, done: true }
)
```

---

# Testing error scenarios

```js
import { call, put } from 'redux-saga/effects'
import Api from '...'

const iterator = fetchProducts()

// expects a call instruction
assert.deepEqual(
  iterator.next().value,
  call(Api.fetch, '/products')
)

const error = {}

// expects an error dispatch
assert.deepEqual(
  iterator.throw(error).value,
  put({ type: 'PRODUCTS_REQUEST_FAILED', error }),
)

// expects to be done
assert.deepEqual(
    iterator.next(),
    { value: undefined, done: true }
)
```

---

# Useful links

* [redux-saga.js.org](https://redux-saga.js.org)
* [github.com/redux-saga/redux-saga](https://github.com/redux-saga/redux-saga)
* [egghead.io/courses/async-react-with-redux-saga](https://egghead.io/courses/async-react-with-redux-saga)

Example code blatantly borrowed from:

* [davidwalsh.name/es6-generators](https://davidwalsh.name/es6-generators)
* [redux-saga.js.org/docs/introduction/](https://redux-saga.js.org/docs/introduction/)
