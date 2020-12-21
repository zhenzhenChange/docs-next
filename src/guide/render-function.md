# Render Functions

Vue recommends using templates to build applications in the vast majority of cases. However, there are situations where we need the full programmatic power of JavaScript. That's where we can use the **render function**.

Let's dive into an example where a `render()` function would be practical. Say we want to generate anchored headings:

```html
<h1>
  <a name="hello-world" href="#hello-world">
    Hello world!
  </a>
</h1>
```

Anchored headings are used very frequently, we should create a component:

```vue-html
<anchored-heading :level="1">Hello world!</anchored-heading>
```

The component must generate a heading based on the `level` prop, and we quickly arrive at this:

```js
const app = Vue.createApp({})

app.component('anchored-heading', {
  template: `
    <h1 v-if="level === 1">
      <slot></slot>
    </h1>
    <h2 v-else-if="level === 2">
      <slot></slot>
    </h2>
    <h3 v-else-if="level === 3">
      <slot></slot>
    </h3>
    <h4 v-else-if="level === 4">
      <slot></slot>
    </h4>
    <h5 v-else-if="level === 5">
      <slot></slot>
    </h5>
    <h6 v-else-if="level === 6">
      <slot></slot>
    </h6>
  `,
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

This template doesn't feel great. It's not only verbose, but we're duplicating `<slot></slot>` for every heading level. And when we add the anchor element, we have to again duplicate it in every `v-if/v-else-if` branch.

While templates work great for most components, it's clear that this isn't one of them. So let's try rewriting it with a `render()` function:

```js
const app = Vue.createApp({})

app.component('anchored-heading', {
  render() {
    return Vue.h(
      'h' + this.level, // tag name
      {}, // props/attributes
      this.$slots.default() // array of children
    )
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

The `render()` function implementation is much simpler, but also requires greater familiarity with component instance properties. In this case, you have to know that when you pass children without a `v-slot` directive into a component, like the `Hello world!` inside of `anchored-heading`, those children are stored on the component instance at `$slots.default()`. If you haven't already, **it's recommended to read through the [instance properties API](../api/instance-properties.html) before diving into render functions.**

## The DOM tree

Before we dive into render functions, it’s important to know a little about how browsers work. Take this HTML for example:

```html
<div>
  <h1>My title</h1>
  Some text content
  <!-- TODO: Add tagline -->
</div>
```

When a browser reads this code, it builds a [tree of "DOM nodes"](https://javascript.info/dom-nodes) to help it keep track of everything.

The tree of DOM nodes for the HTML above looks like this:

![DOM Tree Visualization](/images/dom-tree.png)

Every element is a node. Every piece of text is a node. Even comments are nodes! Each node can have children (i.e. each node can contain other nodes).

Updating all these nodes efficiently can be difficult, but thankfully, we never have to do it manually. Instead, we tell Vue what HTML we want on the page, in a template:

```html
<h1>{{ blogTitle }}</h1>
```

Or in a render function:

```js
render() {
  return Vue.h('h1', {}, this.blogTitle)
}
```

And in both cases, Vue automatically keeps the page updated, even when `blogTitle` changes.

## The Virtual DOM tree

Vue keeps the page updated by building a **virtual DOM** to keep track of the changes it needs to make to the real DOM. Taking a closer look at this line:

```js
return Vue.h('h1', {}, this.blogTitle)
```

What is the `h()` function returning? It's not _exactly_ a real DOM element. It returns a plain object which contains information describing to Vue what kind of node it should render on the page, including descriptions of any child nodes. We call this node description a "virtual node", usually abbreviated to **VNode**. "Virtual DOM" is what we call the entire tree of VNodes, built by a tree of Vue components.

## `h()` Arguments

The `h()` function is a utility to create VNodes. It could perhaps more accurately be named `createVNode()`, but it's called `h()` due to frequent use and for brevity. It accepts three arguments:

```js
// @returns {VNode}
h(
  // {String | Object | Function } tag
  // An HTML tag name, a component or an async component.
  // Using function returning null would render a comment.
  //
  // Required.
  'div',

  // {Object} props
  // An object corresponding to the attributes, props and events
  // we would use in a template.
  //
  // Optional.
  {},

  // {String | Array | Object} children
  // Children VNodes, built using `h()`,
  // or using strings to get 'text VNodes' or
  // an object with slots.
  //
  // Optional.
  [
    'Some text comes first.',
    h('h1', 'A headline'),
    h(MyComponent, {
      someProp: 'foobar'
    })
  ]
)
```

If there are no props then the children can usually be passed as the second argument. In cases where that would be ambiguous, `null` can be passed as the second argument to keep the children as the third argument.

## Complete Example

With this knowledge, we can now finish the component we started:

```js
const app = Vue.createApp({})

/** Recursively get text from children nodes */
function getChildrenTextContent(children) {
  return children
    .map(node => {
      return typeof node.children === 'string'
        ? node.children
        : Array.isArray(node.children)
        ? getChildrenTextContent(node.children)
        : ''
    })
    .join('')
}

app.component('anchored-heading', {
  render() {
    // create kebab-case id from the text contents of the children
    const headingId = getChildrenTextContent(this.$slots.default())
      .toLowerCase()
      .replace(/\W+/g, '-') // replace non-word characters with dash
      .replace(/(^-|-$)/g, '') // remove leading and trailing dashes

    return Vue.h('h' + this.level, [
      Vue.h(
        'a',
        {
          name: headingId,
          href: '#' + headingId
        },
        this.$slots.default()
      )
    ])
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

## Constraints

### VNodes Must Be Unique

All VNodes in the component tree must be unique. That means the following render function is invalid:

```js
render() {
  const myParagraphVNode = Vue.h('p', 'hi')
  return Vue.h('div', [
    // Yikes - duplicate VNodes!
    myParagraphVNode, myParagraphVNode
  ])
}
```

If you really want to duplicate the same element/component many times, you can do so with a factory function. For example, the following render function is a perfectly valid way of rendering 20 identical paragraphs:

```js
render() {
  return Vue.h('div',
    Array.from({ length: 20 }).map(() => {
      return Vue.h('p', 'hi')
    })
  )
}
```

## Creating Component VNodes

To create a VNode for a component, the first argument passed to `h` should be the component itself:

```js
render() {
  return Vue.h(ButtonCounter)
}
```

If we need to resolve a component by name then we can call `resolveComponent`:

```js
render() {
  const ButtonCounter = Vue.resolveComponent('ButtonCounter')
  return Vue.h(ButtonCounter)
}
```

`resolveComponent` is the same function that templates use internally to resolve components by name.

A `render` function will normally only need to use `resolveComponent` for components that are [registered globally](/guide/component-registration.html#global-registration). [Local component registration](/guide/component-registration.html#local-registration) can usually be skipped altogether. Consider the following example:

```js
// We can simplify this
components: {
  ButtonCounter
},
render() {
  return Vue.h(Vue.resolveComponent('ButtonCounter'))
}
```

Rather than registering a component by name and then looking it up we can use it directly instead:

```js
render() {
  return Vue.h(ButtonCounter)
}
```

## Replacing Template Features with Plain JavaScript

### `v-if` and `v-for`

Wherever something can be easily accomplished in plain JavaScript, Vue render functions do not provide a proprietary alternative. For example, in a template using `v-if` and `v-for`:

```html
<ul v-if="items.length">
  <li v-for="item in items">{{ item.name }}</li>
</ul>
<p v-else>No items found.</p>
```

This could be rewritten with JavaScript's `if`/`else` and `map()` in a render function:

```js
props: ['items'],
render() {
  if (this.items.length) {
    return Vue.h('ul', this.items.map((item) => {
      return Vue.h('li', item.name)
    }))
  } else {
    return Vue.h('p', 'No items found.')
  }
}
```

In a template it can be useful to use a `<template>` tag to hold a `v-if` or `v-for` directive. When migrating to a `render` function, the `<template>` tag is no longer required and can be discarded.

### `v-model`

The `v-model` directive is expanded to `modelValue` and `onUpdate:modelValue` props during template compilation—we will have to provide these props ourselves:

```js
props: ['modelValue'],
emits: ['update:modelValue'],
render() {
  return Vue.h(SomeComponent, {
    modelValue: this.modelValue,
    'onUpdate:modelValue': value => this.$emit('update:modelValue', value)
  })
}
```

### `v-on`

We have to provide a proper prop name for the event handler, e.g., to handle `click` events, the prop name would be `onClick`.

```js
render() {
  return Vue.h('div', {
    onClick: $event => console.log('clicked', $event.target)
  })
}
```

#### Event Modifiers

For the `.passive`, `.capture`, and `.once` event modifiers, they can be concatenated after the event name using camel case.

For example:

```javascript
render() {
  return Vue.h('input', {
    onClickCapture: this.doThisInCapturingMode,
    onKeyupOnce: this.doThisOnce,
    onMouseoverOnceCapture: this.doThisOnceInCapturingMode
  })
}
```

For all other event and key modifiers, no special API is necessary, because we can use event methods in the handler:

| Modifier(s)                                           | Equivalent in Handler                                                                                                |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `.stop`                                               | `event.stopPropagation()`                                                                                            |
| `.prevent`                                            | `event.preventDefault()`                                                                                             |
| `.self`                                               | `if (event.target !== event.currentTarget) return`                                                                   |
| Keys:<br>`.enter`, `.13`                              | `if (event.keyCode !== 13) return` (change `13` to [another key code](http://keycode.info/) for other key modifiers) |
| Modifiers Keys:<br>`.ctrl`, `.alt`, `.shift`, `.meta` | `if (!event.ctrlKey) return` (change `ctrlKey` to `altKey`, `shiftKey`, or `metaKey`, respectively)                  |

Here's an example with all of these modifiers used together:

```js
render() {
  return Vue.h('input', {
    onKeyUp: event => {
      // Abort if the element emitting the event is not
      // the element the event is bound to
      if (event.target !== event.currentTarget) return
      // Abort if the key that went up is not the enter
      // key (13) and the shift key was not held down
      // at the same time
      if (!event.shiftKey || event.keyCode !== 13) return
      // Stop event propagation
      event.stopPropagation()
      // Prevent the default keyup handler for this element
      event.preventDefault()
      // ...
    }
  })
}
```

### Slots

We can access slot contents as Arrays of VNodes from [`this.$slots`](../api/instance-properties.html#slots):

```js
render() {
  // `<div><slot></slot></div>`
  return Vue.h('div', this.$slots.default())
}
```

```js
props: ['message'],
render() {
  // `<div><slot :text="message"></slot></div>`
  return Vue.h('div', this.$slots.default({
    text: this.message
  }))
}
```

For component VNodes, we need to pass the children to `h` as an Object rather than an Array. Each property is used to populate the slot of the same name:

```js
render() {
  // `<div><child v-slot="props"><span>{{ props.text }}</span></child></div>`
  return Vue.h('div', [
    Vue.h(
      Vue.resolveComponent('child'),
      null,
      // pass `slots` as the children object
      // in the form of { name: props => VNode | Array<VNode> }
      {
        default: (props) => Vue.h('span', props.text)
      }
    )
  ])
}
```

The slots are passed as functions, allowing the child component to control the creation of each slot's contents. Any reactive data should be accessed within the slot function to ensure that it's registered as a dependency of the child component and not the parent. Conversely, calls to `resolveComponent` should be made outside the slot function, otherwise they'll resolve relative to the wrong component:

```js
// `<MyButton><MyIcon :name="icon" />{{ text }}</MyButton>`
render() {
  // Calls to resolveComponent should be outside the slot function
  const Button = Vue.resolveComponent('MyButton')
  const Icon = Vue.resolveComponent('MyIcon')
  
  return Vue.h(
    Button,
    null,
    {
      // Use an arrow function to preserve the `this` value
      default: (props) => {
        // Reactive properties should be read inside the slot function
        // so that they become dependencies of the child's rendering
        return [
          Vue.h(Icon, { name: this.icon }),
          this.text
        ]
      }
    } 
  )
}
```

If a component receives slots from its parent, they can be passed on directly to a child component:

```js
render() {
  return Vue.h(Panel, null, this.$slots)
}
```

They can also be passed individually or wrapped as appropriate:

```js
render() {
  return Vue.h(
    Panel,
    null,
    {
      // If we want to pass on a slot function we can
      header: this.$slots.header,
      
      // If we need to manipulate the slot in some way
      // then we need to wrap it in a new function
      default: (props) => {
        const children = this.$slots.default ? this.$slots.default(props) : []
        
        return children.concat(Vue.h('div', 'Extra child'))
      }
    } 
  )
}
```

### `<component>` and `is`

Behind the scenes, templates use `resolveDynamicComponent` to implement the `is` attribute. We can use the same function if we need all the flexibility provided by `is` in our `render` function:

```js
// `<component :is="name"></component>`
render() {
  const Component = Vue.resolveDynamicComponent(this.name)
  return Vue.h(Component)
}
```

Just like `is`, `resolveDynamicComponent` supports passing a component name, an HTML element name, or a component options object.

However, that level of flexibility is usually not required. It's often possible to replace `resolveDynamicComponent` with a more direct alternative.

For example, if we only need to support component names then `resolveComponent` can be used instead.

If the VNode is always an HTML element then we can pass its name directly to `h`:

```js
// `<component :is="bold ? 'strong' : 'em'"></component>`
render() {
  return Vue.h(this.bold ? 'strong' : 'em')
}
```

Similarly, if the value passed to `is` is a component options object then there's no need to resolve anything, it can be passed directly as the first argument of `h`.

Much like a `<template>` tag, a `<component>` tag is only required in templates as a syntactical placeholder and should be discarded when migrating to a `render` function.

## JSX

If we're writing a lot of `render` functions, it might feel painful to write something like this:

```js
Vue.h(
  Vue.resolveComponent('anchored-heading'),
  {
    level: 1
  },
  {
    default: () => [Vue.h('span', 'Hello'), ' world!']
  }
)
```

Especially when the template version is so concise in comparison:

```vue-html
<anchored-heading :level="1"> <span>Hello</span> world! </anchored-heading>
```

That's why there's a [Babel plugin](https://github.com/vuejs/jsx-next) to use JSX with Vue, getting us back to a syntax that's closer to templates:

```jsx
import AnchoredHeading from './AnchoredHeading.vue'

const app = createApp({
  render() {
    return (
      <AnchoredHeading level={1}>
        <span>Hello</span> world!
      </AnchoredHeading>
    )
  }
})

app.mount('#demo')
```

For more on how JSX maps to JavaScript, see the [usage docs](https://github.com/vuejs/jsx-next#installation).

## Template Compilation

You may be interested to know that Vue's templates actually compile to render functions. This is an implementation detail you usually don't need to know about, but if you'd like to see how specific template features are compiled, you may find it interesting. Below is a little demo using `Vue.compile` to live-compile a template string:

<iframe src="https://vue-next-template-explorer.netlify.app/" width="100%" height="420"></iframe>
