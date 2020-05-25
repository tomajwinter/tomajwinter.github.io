---
layout: post
title: Fetching data asynchronously in React Components
tags: [blog]
---

This post lays out one approach to building (and testing) React components that make a request to an external data source on componentDidMount. It touches on two approaches — Promises (ES6) and async/await (ES2017).

## 1. The Challenge

It is common for React components to be responsible for fetching data from external data sources. One pattern is to use the componentDidMount lifecycle method to undertake any data fetching and setup. In order for this to be robust the UI should account for latency in the request as well as the possibility of the expected data not successfully being returned.

## 2. Some setup

In order to not couple our Components too closely with any library, and to make testing more flexible, we will implement aHTTPWrapper class. (The actual library being used to make HTTP requests is [axios](https://github.com/axios/axios)). Abstracting it into a wrapper allows for more flexible testing as well as the possibility that you might want to replace your external dependency at some stage.

The below example lays out the wrapper’s implementation of a get request. Just a couple of things to note about the function:

- It uses object destructuring to allow optional arguments (e.g in some cases we might want to include Authentication headers in config, but not in every case)

- It returns a promise (wrapping the same approach as axios)

```javascript
  import axios from 'axios';

  class HTTPWrapper {
    get = ({ url, config } = {}) => new Promise((resolve, reject) => {
      axios
        .get(url, config)
        .then((response) => {
          resolve(response);
        })
        .catch((errors) => {
          reject(errors);
        });
  });

  export default new HTTPWrapper();
```

We will also build a class that handles API calls by making use of this HTTPWrapper . From our component’s perspective, it only need know that it is calling a function, with an ID, that will return some data. This intermediate UserAPI class sits between our component and HTTPWrapper and could look something like:

```javascript
import HTTPWrapper from '../utils/HTTPWrapper'

class UserAPI {
  getDataForUser = id =>
    HTTPWrapper.get({
      url: `my/external/service/v1/users/${id}`,
    })
}

export default new UserAPI()
```

These two extra classes can feel like additional boilerplate — but as well as keeping code concise and with single, smaller responsibilities it also will make it easier to test interactions across (and beyond) our app.

## 3. Our UserDetails component

UserDetails is designed to fetch data about a user and return a greeting message. We want to ensure that during the time a request is being undertaken it displays something meaningful. Then, once it has data available, it updates the UI.

```javascript
import React, { Component } from 'react'
import PropTypes from 'prop-types'
import UserAPI from '../../api/UserAPI'

class UserDetails extends Component {
  static propTypes = {
    userId: PropTypes.string.isRequired,
  }

  constructor(props) {
    super(props)
    this.state = {
      user: null,
    }
  }

  componentDidMount() {
    // get some data
  }

  displayUser = () => {
    const { user } = this.state
    const { name } = user
    return <span>{`Hello ${name}!`}</span>
  }

  loading = () => <span>Loading...</span>

  render() {
    const { user } = this.state
    return user ? this.displayUser() : this.loading()
  }
}

export default UserDetails
```

A common place for a component to make requests for data is in the componentDidMount lifecycle method. Because this request won’t be instaneous it makes sense for us to use an asyncronous pattern. One approach that uses a Promise returned from our UserAPI.getDataForUser function is

```javascript
    componentDidMount() {
        const { userId } = this.props;
        UserAPI.getDataForUser(userId)
          .then(response => {
            this.setState({
              user: response,
            });
          })
          .catch(err => {
            // Handle a failed request
          })
      }
```

This works well and allows us to update component state on a successful or unsuccessful request. There is also an alternative implementation of asynchronous behaviour that utilises async/await.

```javascript
    async componentDidMount() {
        try {
          const { userId } = this.props;
          const userResponse = await UserAPI.getDataForUser(userId);
          this.setState({
            user: userResponse,
          });
        } catch (err) {
          // Handle a failed request
        }
      }
```

Both of these approaches will allow our component to display a loading state until the request for data has concluded, and at that point change to welcome the user. Both also allow us to catch any problems and display a meaningful error state to the end user whilst passing information onto an error service.

## **Testing**

When testing our component it’s good practice to avoid testing implemetation details. An example of some things that we’re not trying to couple our tests to might be

- The HTTP Library we’re using

- Whether our component is utilising promises or async/await to fetch data

- The internal logic and function calls in UserDetails (e.g. functions like loading and displayUser)

A good set of tests here could include expectations that

a) Our component loads without any errors

b) Our component makes a request to the external API with expected parameters

c) Our component’s UI updates based the external response

```javascript
import React from 'react'
import { mount } from 'enzyme'
import UserAPI from '../../api/UserAPI'
import UserDetails from './UserDetails'

const mockedApiRequest = jest.fn(() =>
  Promise.resolve({
    name: 'Username',
  })
)

const props = {
  userId: '1',
}

afterEach(() => {
  jest.clearAllMocks()
})

it('fetches from external API successfully', async () => {
  UserAPI.getDataForUser = mockedApiRequest
  const wrapper = mount(<UserDetails {...props} />)
  expect(mockedApiRequest).toBeCalledTimes(1)
  expect(mockedApiRequest).toBeCalledWith('1')
  await mockedApiRequest
  expect(
    wrapper
      .find('span')
      .last()
      .text()
  ).toEqual('Hello Username!')
})
```

Using [enzyme](https://airbnb.io/enzyme/) our component is mounted which will cause our componentDidMount to be invoked (although, as of Enzyme 3, shallow rendering components does also invoke React lifecycle methods). We have stubbed the UserAPI.getDataForUser function with a jest mock that returns a Promise and a response we have modelled on our API.

Providing that the component mounts without internal error and we’ve stubbed the correct API call, our expectations that the API is called with particular arguments will pass. Without creating tests that declare asynchronous behaviour they would continue to run and move onto the next expectation (that ‘Hello Username!’ is visible) before the wrapper object has reacted to any subsequent change in state.

Jest supports asyncronous testing and we can tell the test that it should wait for the resolution of a successful Promise before continuing to test any expectations. By passingthe async flag into our test we canawait the resolution of the Promise we have defined in order to check the component re-renders as expected.

## 5. Conclusion

The above discussion covers two patterns that allow us to safely fetch external data to populate component state, as well as an asyncronous approach to testing that avoids implementation details and gives confidence in our component’s behaviour & rendered output.

Some of the things we’ve not touched on include [browser compatibility](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function#Browser_compatibility) for asynchronous approaches

## Further reading

- [async functions by MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)

- [Understanding Jest Mocks](https://medium.com/@rickhanlonii/understanding-jest-mocks-f0046c68e53c)

- [Jest documentation around async/await](https://jestjs.io/docs/en/tutorial-async#async-await)

## Background Noise

- **Remind Me Tomorrow** by Sharon Van Etten
- **Why Hasn't Everything Already Disappeared** by Deerhunter
- **IT WON/T BE LIKE THIS ALL THE TIME** by The Twilight Sad
