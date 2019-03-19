# Advanced Redux and Routing in React

## Learning Competencies
At the end of this topic you will be able to learn about

- Understand Asynchronous actions in redux
- Understand Asynchronous flow
- Understanding Middleware in redux
- Async middlewares like `redux-thunk` or `redux-promise`
- Usage with React Router
- build async application using React + Redux

## Overview

## Async Actions

Previously we built a React + Redux application which are fully synchronous.
Every time an action was dispatched, the state was updated immediately.

In this guide, we will build a different, asynchronous application. It will use the Reddit API to show the current headlines for a selected subreddit. How does asynchronicity fit into Redux flow?

Before creating async action creators, look at [this](https://redux.js.org/advanced/async-actions) to create sync action creators

## Async Action creators

Finally, how do we use the synchronous action creators we defined earlier together with network requests? The standard way to do it with Redux is to use the Redux Thunk middleware. It comes in a separate package called `redux-thunk`

Learn about Middleware in detail [here](https://redux.js.org/advanced/middleware)

By using this specific middleware, an action creator can return a function instead of an action object. This way, the action creator becomes a thunk.

When an action creator returns a function, that function will get executed by the Redux Thunk middleware. This function doesn't need to be pure; it is thus allowed to have side effects, including executing asynchronous API calls. The function can also dispatch actions—like those synchronous actions we defined earlier.

We can still define these special thunk action creators inside our `actions.js` file:

#### `actions.js` (Asynchronous)

```js
import fetch from 'cross-fetch'

export const REQUEST_POSTS = 'REQUEST_POSTS'
function requestPosts(subreddit) {
  return {
    type: REQUEST_POSTS,
    subreddit
  }
}

export const RECEIVE_POSTS = 'RECEIVE_POSTS'
function receivePosts(subreddit, json) {
  return {
    type: RECEIVE_POSTS,
    subreddit,
    posts: json.data.children.map(child => child.data),
    receivedAt: Date.now()
  }
}

export const INVALIDATE_SUBREDDIT = 'INVALIDATE_SUBREDDIT'
export function invalidateSubreddit(subreddit) {
  return {
    type: INVALIDATE_SUBREDDIT,
    subreddit
  }
}

// Meet our first thunk action creator!
// Though its insides are different, you would use it just like any other action creator:
// store.dispatch(fetchPosts('reactjs'))

export function fetchPosts(subreddit) {
  // Thunk middleware knows how to handle functions.
  // It passes the dispatch method as an argument to the function,
  // thus making it able to dispatch actions itself.

  return function (dispatch) {
    // First dispatch: the app state is updated to inform
    // that the API call is starting.

    dispatch(requestPosts(subreddit))

    // The function called by the thunk middleware can return a value,
    // that is passed on as the return value of the dispatch method.

    // In this case, we return a promise to wait for.
    // This is not required by thunk middleware, but it is convenient for us.

    return fetch(`https://www.reddit.com/r/${subreddit}.json`)
      .then(
        response => response.json(),
        // Do not use catch, because that will also catch
        // any errors in the dispatch and resulting render,
        // causing a loop of 'Unexpected batch number' errors.
        // https://github.com/facebook/react/issues/6895
        error => console.log('An error occurred.', error)
      )
      .then(json =>
        // We can dispatch many times!
        // Here, we update the app state with the results of the API call.

        dispatch(receivePosts(subreddit, json))
      )
  }
}
```

>##### Note on `fetch`

>We use [`fetch` API](https://developer.mozilla.org/en/docs/Web/API/Fetch_API) in the examples. It is a new API for making network requests that replaces `XMLHttpRequest` for most common needs. Because most browsers don't yet support it natively, we suggest that you use [`cross-fetch`](https://github.com/lquixada/cross-fetch) library:

>```js
>// Do this in every file where you use `fetch`
>import fetch from 'cross-fetch'
>```

>Internally, it uses [`whatwg-fetch` polyfill](https://github.com/github/fetch) on the client, and [`node-fetch`](https://github.com/bitinn/node-fetch) on the server, so you won't need to change API calls if you change your app to be [universal](https://medium.com/@mjackson/universal-javascript-4761051b7ae9).

>Be aware that any `fetch` polyfill assumes a [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) polyfill is already present. The easiest way to ensure you have a Promise polyfill is to enable Babel's ES6 polyfill in your entry point before any other code runs:

>```js
>// Do this once before any other code in your app
>import 'babel-polyfill'
>```

How do we include the Redux Thunk middleware in the dispatch mechanism? We use the [`applyMiddleware()`](../api/applyMiddleware.md) store enhancer from Redux, as shown below:

#### `index.js`

```js
import thunkMiddleware from 'redux-thunk'
import { createLogger } from 'redux-logger'
import { createStore, applyMiddleware } from 'redux'
import { selectSubreddit, fetchPosts } from './actions'
import rootReducer from './reducers'

const loggerMiddleware = createLogger()

const store = createStore(
  rootReducer,
  applyMiddleware(
    thunkMiddleware, // lets us dispatch() functions
    loggerMiddleware // neat middleware that logs actions
  )
)

store.dispatch(selectSubreddit('reactjs'))
store
  .dispatch(fetchPosts('reactjs'))
  .then(() => console.log(store.getState()))
```

The nice thing about thunks is that they can dispatch results of each other:

#### `actions.js` (with `fetch`)

```js
import fetch from 'cross-fetch'

export const REQUEST_POSTS = 'REQUEST_POSTS'
function requestPosts(subreddit) {
  return {
    type: REQUEST_POSTS,
    subreddit
  }
}

export const RECEIVE_POSTS = 'RECEIVE_POSTS'
function receivePosts(subreddit, json) {
  return {
    type: RECEIVE_POSTS,
    subreddit,
    posts: json.data.children.map(child => child.data),
    receivedAt: Date.now()
  }
}

export const INVALIDATE_SUBREDDIT = 'INVALIDATE_SUBREDDIT'
export function invalidateSubreddit(subreddit) {
  return {
    type: INVALIDATE_SUBREDDIT,
    subreddit
  }
}

function fetchPosts(subreddit) {
  return dispatch => {
    dispatch(requestPosts(subreddit))
    return fetch(`https://www.reddit.com/r/${subreddit}.json`)
      .then(response => response.json())
      .then(json => dispatch(receivePosts(subreddit, json)))
  }
}

function shouldFetchPosts(state, subreddit) {
  const posts = state.postsBySubreddit[subreddit]
  if (!posts) {
    return true
  } else if (posts.isFetching) {
    return false
  } else {
    return posts.didInvalidate
  }
}

export function fetchPostsIfNeeded(subreddit) {
  // Note that the function also receives getState()
  // which lets you choose what to dispatch next.

  // This is useful for avoiding a network request if
  // a cached value is already available.

  return (dispatch, getState) => {
    if (shouldFetchPosts(getState(), subreddit)) {
      // Dispatch a thunk from thunk!
      return dispatch(fetchPosts(subreddit))
    } else {
      // Let the calling code know there's nothing to wait for.
      return Promise.resolve()
    }
  }
}
```

This lets us write more sophisticated async control flow gradually, while the consuming code can stay pretty much the same:

#### `index.js`

```js
store
  .dispatch(fetchPostsIfNeeded('reactjs'))
  .then(() => console.log(store.getState()))
```

>##### Note about Server Rendering

>Async action creators are especially convenient for server rendering. You can create a store, dispatch a single async action creator that dispatches other async action creators to fetch data for a whole section of your app, and only render after the Promise it returns, completes. Then your store will already be hydrated with the state you need before rendering.

[Thunk middleware](https://github.com/gaearon/redux-thunk) isn't the only way to orchestrate asynchronous actions in Redux:
- You can use [redux-promise](https://github.com/acdlite/redux-promise) or [redux-promise-middleware](https://github.com/pburtchaell/redux-promise-middleware) to dispatch Promises instead of functions.
- You can use [redux-observable](https://github.com/redux-observable/redux-observable) to dispatch Observables.
- You can use the [redux-saga](https://github.com/yelouafi/redux-saga/) middleware to build more complex asynchronous actions.
- You can use the [redux-pack](https://github.com/lelandrichardson/redux-pack) middleware to dispatch promise-based asynchronous actions.
- You can even write a custom middleware to describe calls to your API, like the [real world example](../introduction/Examples.md#real-world) does.

It is up to you to try a few options, choose a convention you like, and follow it, whether with, or without the middleware.

## Async Flow

Without middleware, Redux store only supports synchronous data flow. This is what you get by default with `createStore()`.

You may enhance `createStore()` with `applyMiddleware()`. It is not required, but it lets you express asynchronous actions in a convenient way.

Asynchronous middleware like redux-thunk or redux-promise wraps the store's `dispatch()` method and allows you to dispatch something other than actions, for example, functions or Promises. Any middleware you use can then interpret anything you dispatch, and in turn, can pass actions to the next middleware in the chain. For example, a Promise middleware can intercept Promises and dispatch a pair of begin/end actions asynchronously in response to each Promise.

When the last middleware in the chain dispatches an action, it has to be a plain object. This is when the synchronous Redux data flow takes place.

## Usage with React Router

Lear about Routing with redux from [here](https://redux.js.org/advanced/usage-with-react-router)

## Exploration

- [video](https://www.youtube.com/watch?v=Td-2D-_7Y2E) on async actions in redux
- [video](https://www.youtube.com/watch?v=DJ8fR0mZM44) on Middleware in redux
- [Asynchronous actions in Redux](https://medium.com/@jtbennett/asynchronous-actions-in-redux-8412cf92a26f)
- [Route with redux](https://jlongster.com/A-Simple-Way-to-Route-with-Redux)
- Look at [react-router-redux](https://github.com/reactjs/react-router-redux) library