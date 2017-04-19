# Efficient Indexing in React-Redux Applications

### Part 1: Cleaner Reducer Design

Very frequently in an application, we need to retrieve a list of data from an API. Here is a commonly used reducer pattern:

```
import { combineReducers } from 'redux';
import types from './types';

const all = (
  state = [],
  action,
) => {
  switch (action.type) {
    case types.LIST_POSTS_SUCCESS:
      return action.result.data;
    default: return state;
  }
};

export const posts = combineReducers({ all });
```
This reducer simply returns the data that was returned in the reponse and places it in to the `posts.all`.

And generally when you need to retrieve a list of data from an API, you will also need to be able to get the data of a singular item. This raises a question- 


> *If I need to pull the data of one `post` from the API, where should this information live in our Redux store? Should it be it's own object? i.e. `posts.current` ?*

This is where an efficient indexing pattern becomes very useful.

```
import { combineReducers } from 'redux';
import types from './types';
import { get, reduce, set, map } from 'lodash;

const all = (
  state = {},
  action,
) => {
  switch (action.type) {
    case types.LIST_POSTS_SUCCESS:
      reduce(
        get(action, 'result.data'),
        (all, cur) => set(all, cur.id, cur),
        state
      );
      return { ...state };
    case types.LOAD_POST_DETAIL_SUCCESS:
      return { ...state, [get(action, 'result.data.id')]: get(action, 'result.data') };
    default: return state;
  }
};

const index = (
  state = [],
  action,
) => {
  switch (action.type) {
    case types.LIST_POSTS_SUCCESS:
      return [...state, map(action.result.data, data => data.id)];
    default: return state;
}
```

Take a look at the code above and observe a few things. We will go over each point in detail.

1. The `post.all` property now has an initial state of an empty object rather than an array.
2. On case `types.LIST_POSTS_SUCCESS`, we have a reducer function that will change an array of objects into an object with `id`s as keys, and the objects as values.

```
[
	{id: 1, foo: 'abc', bar: 'def'},
	{id: 2, foo: 'xyz', bar: 'qwop'},
	{id: 3, foo: 'ghj', bar: 'something'},
]

// turns into

{ 
	1: {id: 1, foo: 'abc', bar: 'def'}, 
	2: {id: 2, foo: 'xyz', bar: 'qwop'}, 
	3: {id: 3, foo: 'ghj', bar: 'something'}, 
}

```

3. When `LOAD_POST_DETAIL_SUCCESS` is dispatched, the retrieved data is assigned into the `all` object in the same format as listed above. This means that if the value exists, then the object either gets inserted into the object if the key(id) does not exist, and if it does exist, the value of the key will be updated to the `action.result.data` object.

> Why is this better than having a `posts.current` reducer that returns the value of the response i.e. `return action.result.data`?

If you are to pull a new post from your API and it has data deferring from the data of the specific post in a `posts.all` array, what checks do you have in place that your two sources of data in your Redux store are always in sync with each other? 

To answer that, one would assume that you would need to map over the `posts.all` array and find the object that has the same value to the `id` property, and update that object. Traversing through the array can be unnessesarily complex.

However, if you were to use the spread operator (as used in the example,) or `Object.assign` to return a new `all` object with dat inserted/updated, it is very easy to navigate to said object since we can use the `id` as a key.

By design, the given example only has one source of truth. This helps prevent the mismatch of data down the pipeline in your React code. 

> But wait, how do I know which post is the current one when I go to a `post` detail page? 

When we removed the `current` object, our React-Redux `connect`-ed component suddenly lost its prop that was synced with the Redux store. How do we solve this?

Simply, when we would like to set the current object, we store it's id in the `current` reducer.

```
const current = (
  state = {
    id: 0,
  },
  action,
) => {
  switch (action.type) {
    case types.SET_CURRENT_POST:
      return {
        id: action.id,
      };
    default: return state;
  }
};
```

Note that the reducer for `current` does not contain any data, only an `id`.

In our post detail container, we would access the object from the `all` attribute of the `postReducer` by passing the current id as an index.

```
// Post Detail Container
componentDidMount() {
  this.props.loadPostDetail(this.props.params.id);
  this.props.setCurrentPost(this.props.params.id);
}
...

...
function mapStateToProps(state) {
  return {
    current: state.productions.all[
      state.productions.current.id
    ],
  };
}
export default connect(mapStateToProps, { loadPostDetail, setCurrentPost })(Post);

```

Note that we dispatch `loadProductionDetail` and `setCurrentPost`, 

* `loadProductionDetail` dispatches the action that fire off an API call and adds that data in `posts.all`.
* `setCurrentPost` dispatches an action that sets `post.current.id` to `this.props.params.id`


### Part 2: Faster and more efficient smart components
This type of indexing is useful when the list of items can grow to the thousands. A very common problem in React-Redux applications is slow and laggy re-rendering of lists.

This is because if you were to do a simple map of an array of items in a connected container, then anytime you were to modify the array, every single item would re-render. Take a look at the following code and note that the container (Posts.js) simply passes down `id`s to each `Box` component. The `Box` components are `connect`-ed, but listen only to the data that corresponds to the `id` passed to them as a prop. This means that the individual `Box` will rerender only if there is a change to the specific `Post` data that it is listening to.


```
// Posts.js
import React from 'react';
import { connect } from 'react-redux';
import Box from 'components/Box';
import { listPosts } from 'modules/posts/actions;

class Posts extends React.Component {

  componentWillMount(){
    this.props.listPosts();
  }

  render() {
    return (
	  posts.map((id) =>
	    <Box key={id} id={id} />
	  );
	);
  }
}

function mapStateToProps(state) {
  return {
    ids: state.posts.index,
  };
}

export default connect(mapStateToProps, { listPosts })(Posts);

```

```
// Box.js
import React from 'react';
import { connect } from 'react-redux';

class Box extends React.Component {
  render() {
    const {title, body} = this.props.post;
    return (
      <div>
        <h1>{ title }</h1>
        <div>{ body }</div>
      </div>
    );
  }
}
function mapStateToProps(state, props) {
  const { id } = props;
  const { posts } = state;
  return {
    post: posts.all[id],
  };
}
```
