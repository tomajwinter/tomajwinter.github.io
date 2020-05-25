---
layout: post
title: Functional components in React
tags: [blog]
---

Summary: The core building blocks of React are components. These components may possess enough complexity to require state and lifecycle methods. However some are simple enough that they can be written as pure Javascript functions. This post demonstrates how a component can be rewritten as a Javascript function rather than an ES6 Class

> Conceptually, components are like JavaScript functions. They accept arbitrary inputs (called “props”) and return React elements describing what should appear on the screen...(t)he simplest way to define a component is to write a JavaScript function ([\*React Docs](https://facebook.github.io/react/docs/components-and-props.html))\*

In much the same way that it’s sometimes easy to over-reach for a Store implementation like [Redux](http://redux.js.org/docs/introduction/) when internal state would suffice, it can be easy to introduce unnecessary overhead by reaching for a fully fledged ES6 Class when a functional, stateless implementation would do.

Take the example below. A simple, presentational component which renders a title, subtitle and some further content. For our purposes it’s been called a Section.

```javascript
import React from 'react';
import classNames from 'classnames';

class Section extends React.Component {
  static propTypes = {
    title: React.PropTypes.string.isRequired,
    subtitle: React.PropTypes.string,
    children: PropTypes.node
  }

title = () => {
  return (
    <h2 className='section__title'>{ props.title }</h2>
  );
}

subtitle = () => {
  if (!this.props.subtitle) { return null; }

  return (
    <p className='section__subtitle'>{ props.subtitle }</p>
  );
}

render() {
  return (
    <div className={ classNames(props.className, 'section') }>
      { this.title() }
      { this.subtitle() }
      { props.children }
    </div>
  );
}

export default Section;
```

There’s nothing about the above component which hinges on any implementation of state or utilises lifecycle methods to respond to change. It’s therefore a perfect candidate for being written as a straightforward Javascript function. The first thing to define is the component.

```javascript
import React from 'react'
import classNames from 'classnames'

let Section = props => (
  <div className={classNames(props.className, 'section')}>...</div>
)
export default Section
```

The render method has effectively been replaced by using the [ES6 Arrow Function’s implicit return](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Functions/Arrow_functions). This Section component is now a pure Javascript function. Likewise the functions that return a title and subtitle can be rewritten

```javascript
const title = props => {
  return <h2 className="section__title">{props.title}</h2>
}

const subtitle = props => {
  if (!props.subtitle) {
    return null
  }

  return <p className="section__title">{props.subtitle}</p>
}
```

Note that the relevant props object needs to be passed into these additional functions rather than being with the scope of the class and available by calling this.props.

In addition the Section has its propTypes defined — both .propTypes and .defaultProps can be set as properties in a functional component.

```javascript
Section.propTypes = {
  title: React.PropTypes.string.isRequired,
  subtitle: React.PropTypes.string,
  children: React.PropTypes.node,
}
```

The entire rewritten code is then a stateless Javascript component

```javascript
import React from 'react'
import classNames from 'classnames'

const title = props => {
  return <h2 className="section__title">{props.title}</h2>
}

const subtitle = props => {
  if (!props.subtitle) {
    return null
  }

  return <p className="section__title">{props.subtitle}</p>
}

let Section = props => (
  <div className={classNames(props.className, 'section')}>
    {title(props)}
    {subtitle(props)}
    {props.children}
  </div>
)
Section.propTypes = {
  title: React.PropTypes.string.isRequired,
  subtitle: React.PropTypes.string,
  children: React.PropTypes.node,
}

export default Section
```

The above represents a considerably less cluttered, pure Javascript implementation of the Section component.[ I agree with Jack Franklin’s suggestion](http://javascriptplayground.com/blog/2017/03/functional-stateless-components-react/) that one of the main benefits of a functional stateless component ‘acts as a visual signal’

> “this component is solely props in, rendered UI out”. If I see a class component, I do have to scan through to see what lifecycle methods it may be using, and what callbacks it may have. If I see an FSC, I know it isn’t doing anything fancy.

Another benefit of this approach is that it allows straightforward unit testing — there aren’t any callbacks or user interaction to simulate. The test is that if certain props are passed in, the expected content is rendered.

The React team are committed to this methodology. In the [release notes](https://facebook.github.io/react/blog/2015/10/07/react-v0.14.html#stateless-functional-components) for v0.14 they argued that _“In idiomatic React code, most of the components you write will be stateless, simply composing other components”_. Additionally that, over time, this approach will become more performant as _“we’ll also be able to make performance optimizations specific to these components by avoiding unnecessary checks and memory allocations.”_

Since becoming familiar with this pattern for writing components I’ve endeavoured to implement it wherever possible. Much like the distinction between Presentational & Functional components this pattern also drives your design thinking. As you become more mindful of constructing your code to be as [simple and reusable as possible](https://facebook.github.io/react/docs/thinking-in-react.html).

## Further Reading

- [Presentational and Container Components](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0) — Dan Abramov

- [Functional and Class Components](https://facebook.github.io/react/docs/components-and-props.html#functional-and-class-components) — React Documentation

- [React Functional Stateless Components](https://hackernoon.com/react-stateless-functional-components-nine-wins-you-might-have-overlooked-997b0d933dbc)—Cory House

- [Functional Stateless Components](http://javascriptplayground.com/blog/2017/03/functional-stateless-components-react/) — Jack Franklin

## Background noise

- Kelly Lee Owens — Kelly Lee Owens

- Aldous Harding — Party

- Alison Crutchfield — Tourist In This Town
