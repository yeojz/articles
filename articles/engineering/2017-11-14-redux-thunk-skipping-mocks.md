---
title: "Redux Thunk - skipping mocks using withExtraArgument"
date: 2017-11-14
first_written: 2017-11-14
note: ""
original_url: "https://medium.com/@yeojz/redux-thunk-skipping-mocks-using-withextraargument-513d38d38554"
---


> **tl;dr:** Do take a look at redux-thunk's withExtraArgument. It may help further de-couple common dependencies for simpler test code.

If you have been using Redux for a period of time, you will have had to pair it up with redux-thunk or something similar to handle asynchronous actions at some point of time.

A common use case for thunks is to allow an action creator to make API calls. Your code would probably look something like the following snippet:

```js
import api from './api';

function fetchData(id) {
  return (dispatch) => {
    api.getSomeData(id).then((response) => {
      dispatch({
        type: FETCH_DATA_SUCCESS, 
        payload: response.data
      });
    });
  }
}

export default fetchData;
```

However, if you want to test this code, you would probably have to resort to using mocks on your dependencies. As such, your corresponding spec/test file would probably look something like the following:

```js
import api from './api';
import fetchData from './fetchData'

test('it should call with correct args', function () {
  const method = jest.spyOn(api, 'getSomeData')
    .mockImplementation(() => Promise.resolve({
      data: 'value'
    }));

  const dispatch = jest.fn();

  fetchData(1)(dispatch);

  expect(method).toHaveBeenCalledWith(1);
  expect(dispatch).toHaveBeenCalledWith({
    type: FETCH_DATA_SUCCESS, 
    payload: 'value'
  });
});
```


What if we want to further de-couple this? Why not pass `api` as an argument of the thunk function instead? That is exactly what "*withExtraArgument*" method of `redux-thunk`, which introduced in v2.1.0, can help you achieve.

> Early users of redux-thunk v1.x may have overlooked the addition of "withExtraArgument" , which was introduced since v2.1.0.

This was something I have totally missed for the longest of time until a recent project. With this, your action creator will look something like this instead:

```js
// When adding your thunk middleware
import thunk from 'redux-thunk';
import api from './api';

const store = createStore(
  reducer,
  applyMiddleware(thunk.withExtraArgument({ api }))
)

// Your action creator becomes:
function fetchData(id) {
  return (dispatch, getState, { api }) => {
    api.getSomeData(id).then((response) => {
      dispatch({
        type: FETCH_DATA_SUCCESS, 
        payload: response.data
      });
    });
  }
}

export default fetchData;
```

As such, the corresponding test code would probably look something like the following:

```js
import fetchData from './fetchData'

test('it should call with correct args', function () {
  const api = {
    getSomeData: jest.fn(() => Promise.resolve({ data: 'value' }))
  }
  const dispatch = jest.fn();
  
  fetchData(1)(dispatch, null, { api });

  expect(api.getSomeData).toHaveBeenCalledWith(1);
  expect(dispatch).toHaveBeenCalledWith({
    type: FETCH_DATA_SUCCESS, 
    payload: 'value'
  });
});
```

Since api (or some other dependency) is now an argument as part of your thunk function, you can easily pass in your stubs or test methods without having to resort full mocks.

> This might seem like a trivial change. However, when you are trying to mock "imports/requires" that are non-objects, the above alternative helps to simplify testing by a margin.

While `jest`, used in the snippets above, provides module mocking out of the box, the same cannot be said with other testing frameworks. You may have to use other libraries like `rewire`, `mockery` or even `babel-plugin-rewire` to transpile your code, resulting in a more complicated the test code.

With that said, your mileage and benefit may vary. Usage of "*withExtraArguments*" may not applicable in all scenarios due to existing codebase or code-splitting/bundling requirements, but it is something worth keeping in the back of the head while setting up the next project / refactoring codes.