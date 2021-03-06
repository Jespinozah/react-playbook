# All about React's `cloneElement()`
## What is it?
[React.cloneElement()](https://facebook.github.io/react/docs/react-api.html#cloneelement) allows us to clone a runtime `element` (not the class), and apply an enhancement to it.

```jsx
class Press_ extends React.Component {
  render() {
    // React.cloneElement() requires a single child
    const Child = React.Children.only(this.props.children)

    // compute props to pass
    const propsToPass = {onPress: this.props.onPress}

    // return new element with props to pass
    // note: this could also be wrapped in other components, if we wanted
    return React.cloneElement(Child, propsToPass)
  }
}

class MyComponent extends React.Component {
  handlePress = () => {
    // ...
  }

  render() {
    return (
      <Press_ onPress={this.handlePress}>
        <div />
      </Press_>
    )
  }
}

ReactDom.Render(<MyComponent />, /* ... */)
```

The code above renders a `div` with the `onPress` wired up. It is a contrived example, as a we can just pass in an `onPress` directly to the `div`, but what if `<Press_ />` computed something complicated, then translated the result to something `div` could use? That is quite a powerful abstraction!

Also, this `<Press_ />` component allows us to clearly separate concerns and reuse them on other elements. We might reuse our `Press_` on an `image` or `span`, for example. We might have other `cloneElement()` components for concerns like `<Style_ />`, `<Animate_ />`, `<Touch_ />`, etc.

*Wondering about this `Name_` convention? Check out [Injector Components](https://github.com/kylpo/react-playbook/blob/master/patterns/Injector-Component.md).*

## How does it work?
Given this page, what do we expect to see in console?

```jsx
class Cloning_ extends React.Component {
  componentDidMount() {
    console.log('Cloning Mount')
  }

  render() {
    console.log('Cloning Render')

    const { children, ...propsToPass } = this.props
    const child = React.Children.only(children)

    console.log('ChildProps: ', child.props)

    return React.cloneElement(child, propsToPass)
  }
}

class Div extends React.Component {
  static defaultProps = {
    bye: 'bye'
  }

  componentDidMount() {
    console.log('Div Mount')
  }

  render() {
    console.log('Div Render')

    return (
      <div>{this.props.children}</div>
    )
  }
}

export default class App extends React.Component {
  render() {
    return (
      <Cloning_>
        <Div hi="hi" />
      </Cloning_>
    )
  }
}
```

```bash
Cloning Render
ChildProps:  Object {hi: "hi", bye: "bye"}
Div Render

Div Mount
Cloning Mount
```

Hmmm, how can `Cloning_` have the child `Div` without `Div` being rendered first? Ahhhh, it must create an instance of it, which `Cloning_` can use. So, lets `console.log()` in each of their `constructor()`s.

```bash
Cloning Constructor
Cloning Render
ChildProps:  Object {hi: "hi", bye: "bye"}

Div Constructor
Div Render

Div Mount
Cloning Mount
```

Nope, that wasn't it.

And why can we see its props - especially the `defaultProps`!? What about its state? If we gave it a state of `state = {why: 'why'}`, could `Cloning_` see it?

First of all, no. `Cloning_` can not see it's child's state. Why? Well, because it doesn't exist yet. Keep in mind that JSX maps to `React.createElement()`, which returns an `element` (a simple object description React uses to apply to DOM).

```jsx
React.createElement(
  Cloning_,
  null,
  React.createElement(
    Div,
    { hi: 'hi' }
  )
)
```

When `App` is rendered (from `ReactDOM.render()`), in creating a `Cloning_` element, it must first create the `Div` element. It is normal javascript execution order, but still nice to see:

```jsx
function createElement(label, component, props, children) {
  console.log(`${label} element created`)

  return React.createElement(
    component,
    props,
    children
  )
}
export default class App extends React.Component {
  render() {
    return (
      createElement(
        'Cloning_',
        Cloning_,
        null,
        createElement(
          'Div',
          Div,
          { hi: 'hi' },
        ),
      )
    )
  }

```

```bash
Div element created
Cloning_ element created
```

Back on topic: `Cloning_` is acting on the `element` of `Div`, and returning a new `element`. The returned `element` is a clone, with potentially modified passed in props.

Oh, is it like `Object.assign()` then? Almost like `Object.assign({}, divElement, {newProp: 'newProp'})`?

Yes, actually. The clone's props will even override the child props. Seen here:
```jsx
class Cloning_ extends React.Component {
  render() {
    const { children, ...propsToPass } = this.props
    const child = React.Children.only(children)

    console.log('childProps: ', child.props)

    const clone = React.cloneElement(child, propsToPass)
    console.log('cloneProps: ', clone.props)

    return clone
  }
}

export default class App extends React.Component {
  render() {
    return (
      <div >
        <Cloning_ hi='bye'>
          <Div hi='hi' />
        </Cloning_>
      </div>
    )
  }
}
```
```bash
childProps:  Object {hi: "hi", bye: "bye"}
cloneProps:  Object {hi: "bye", bye: "bye"}
```

Notice the change in the value of `hi`.

OK, one last thing. See that `bye: "bye"` part? That crept in because `Div` has `static defaultProps = {bye: 'bye'}`. Thanks to it being a `static`, `React.CreateElement()` is able to use it for the returned `element`.

### Order of operations in rendering elements, components, parents, and children?
Allllright, lets finish this exploration. What do you expect to be logged in this example?

```jsx
class Cloning_ extends React.Component {
  constructor() {
    super()

    console.log('Cloning Constructor')
  }

  componentDidMount() {
    console.log('Cloning Mount')
  }

  render() {
    console.log('Cloning Render')

    const { children, ...propsToPass } = this.props
    const child = React.Children.only(children)

    return React.cloneElement(child, propsToPass)
  }
}

class Div extends React.Component {
  constructor() {
    super()

    console.log('Div Constructor')
  }

  componentDidMount() {
    console.log('Div Mount')
  }

  render() {
    console.log('Div Render')

    return (
      <div>{this.props.children}</div>
    )
  }
}

function createElement(label, component, props, children) {
  console.log(`${label} element`)

  return React.createElement(
    component,
    props,
    children
  )
}

export default class App extends React.Component {
  render() {
    /*
    * <Cloning_ >
    *   <Div/>
    * </Cloning_>
    */
   return (
     createElement(
       'Cloning_',
       Cloning_,
       {hi: 'hi'},
       createElement(
         'Div',
         Div,
         null,
       ),
     )
   )
  }
}

// elsewhere...

ReactDOM.render(
  <App />,
  document.getElementById('root')
)
```

```bash
Div element
Cloning_ element

Cloning Constructor
Cloning Render
Div Constructor
Div Render

Div Mount
Cloning Mount
```

### Putting it all together
The [offical docs](https://facebook.github.io/react/docs/components-and-props.html) have a template of how updates happen. Let's modify it with our use case:
1. `ReactDOM.render()` is called with with the `<App />` element
2. Using the `App` element, React creates and renders the `App` component
3. Our `App` component creates `Div`, then `Cloning_` elements and returns the tree (`<Cloning_><Div hi='hi' /></Cloning_>`) as the result.
4. Using the `Cloning_` element, React creates and renders  the `Cloning_` component with `{hi: 'hi'}` as the props.
5. Our `Cloning_` component returns a cloned `<Div />` element, with `{hi: 'hi'}` passed as props, as the result.
6. Using the `Div` element, React creates and renders the `Div` component
7. Our `Div` component returns a `<div />` element as the result.
8. React DOM efficiently updates the DOM to match the tree of elements.

## Performance Considerations
### How does `cloneElement()` compare to `createElement()`?
Basically just a React.createElement with a single `for` loop to copy over props passed. See its [source](https://github.com/facebook/react/blob/master/src/isomorphic/classic/element/ReactElement.js#L300-L369), and [this](https://twitter.com/spikebrehm/status/829032493286248448) tweet thread.

### Re-renders
Based on what we learned in TODO scu doc, these cloning wrappers will be re-rendering often. Consider caching the computation outside of `render` so its `render` can do as little work as possible. (Note: I have not tried this yet, but plan to.)

### `cloneElement()` of a `PureComponent` child
When cloning a `PureComponent`, the 2nd argument of the `cloneElement()` can be a new object, but no value of that object should be a new object, array, or function.

Below is an example of safe usage:
```jsx
class Cloning_ extends React.Component {
  render() {
    console.log('Cloning Render')

    const { children, ...propsToPass } = this.props
    const child = React.Children.only(children)

    console.log('passProps', propsToPass)

    return React.cloneElement(child, propsToPass)
  }
}

class PureComponent extends React.PureComponent {
  render() {
    console.log('PureComponent Render')
    return (
      <div />
    )
  }
}

export default class App extends React.Component {
  state = {}

  componentDidMount() {
    setTimeout(() => this.setState({hi: 'hi'}), 2000)
  }

  render() {
    return (
      <Cloning_ hi='hi'>
        <PureComponent />
      </Cloning_>
    )
  }
}

```
```bash
# first render
Cloning Render
passProps Object {hi: "hi"}
PureComponent Render

# after setState
Cloning Render
passProps Object {hi: "hi"}
# GOOD! missing PureComponent Render
```

And now the result of the bad case, where `Cloning_` returns a new object with a new object: `React.cloneElement(child, {nestedNewObject: {}})`
```bash
# first render
Cloning Render
passProps Object {hi: "hi"}
PureComponent Render

# after setState
Cloning Render
passProps Object {hi: "hi"}
PureComponent Render # BAD!
```

See [this](https://github.com/facebook/react/issues/7412) github issue for more.

## Nesting
Think about nested `cloneElement()`s:
```jsx
<Cloning_>
  <Cloning2_>
    <Div />
  </Cloning2_>
</Cloning_>
```

To be a good cloning citizen, be sure to pass props through that are not related to yours! In the above example, `Cloning2_` should pass `Cloning_`'s props to `Div`, as well as its own.

## References
Props to the current official [docs](https://facebook.github.io/react/docs/components-and-props.html), which explain all of this nicely. I just needed more examples to solidify everything.

---

Note: I publish these to learn from your responses! Please let me know if you have any thoughts on the subject.
