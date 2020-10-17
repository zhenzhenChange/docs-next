# Custom Directives

## Intro

In addition to the default set of directives shipped in core (like `v-model` or `v-show`), Vue also allows you to register your own custom directives. Note that in Vue, the primary form of code reuse and abstraction is components - however, there may be cases where you need some low-level DOM access on plain elements, and this is where custom directives would still be useful. An example would be focusing on an input element, like this one:

<p class="codepen" data-height="300" data-theme-id="39028" data-default-tab="result" data-user="Vue" data-slug-hash="JjdxaJW" data-editable="true" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="Custom directives: basic example">
  <span>See the Pen <a href="https://codepen.io/team/Vue/pen/JjdxaJW">
  Custom directives: basic example</a> by Vue (<a href="https://codepen.io/Vue">@Vue</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

When the page loads, that element gains focus (note: `autofocus` doesn't work on mobile Safari). In fact, if you haven't clicked on anything else since visiting this page, the input above should be focused now. Also, you can click on the `Rerun` button and input will be focused.

Now let's build the directive that accomplishes this:

```js
const app = Vue.createApp({})
// Register a global custom directive called `v-focus`
app.directive('focus', {
  // When the bound element is mounted into the DOM...
  mounted(el) {
    // Focus the element
    el.focus()
  }
})
```

If you want to register a directive locally instead, components also accept a `directives` option:

```js
directives: {
  focus: {
    // directive definition
    mounted(el) {
      el.focus()
    }
  }
}
```

Then in a template, you can use the new `v-focus` attribute on any element, like this:

```html
<input v-focus />
```

## Hook Functions

A directive definition object can provide several hook functions (all optional):

- `beforeMount`: called when the directive is first bound to the element and before parent component is mounted. This is where you can do one-time setup work.

- `mounted`: called when the bound element's parent component is mounted.

- `beforeUpdate`: called before the containing component's VNode is updated

:::tip Note
We'll cover VNodes in more detail [later](render-function.html#the-virtual-dom-tree), when we discuss render functions.
:::

- `updated`: called after the containing component's VNode **and the VNodes of its children** have updated.

- `beforeUnmount`: called before the bound element's parent component is unmounted

- `unmounted`: called only once, when the directive is unbound from the element and the parent component is unmounted.

You can check the arguments passed into these hooks (i.e. `el`, `binding`, `vnode`, and `prevVnode`) in [Custom Directive API](../api/application-api.html#directive)

### Dynamic Directive Arguments

Directive arguments can be dynamic. For example, in `v-mydirective:[argument]="value"`, the `argument` can be updated based on data properties in our component instance! This makes our custom directives flexible for use throughout our application.

Let's say you want to make a custom directive that allows you to pin elements to your page using fixed positioning. We could create a custom directive where the value updates the vertical positioning in pixels, like this:

```vue-html
<div id="dynamic-arguments-example" class="demo">
  <p>Scroll down the page</p>
  <p v-pin="200">Stick me 200px from the top of the page</p>
</div>
```

```js
const app = Vue.createApp({})

app.directive('pin', {
  mounted(el, binding) {
    el.style.position = 'fixed'
    // binding.value is the value we pass to directive - in this case, it's 200
    el.style.top = binding.value + 'px'
  }
})

app.mount('#dynamic-arguments-example')
```

This would pin the element 200px from the top of the page. But what happens if we run into a scenario when we need to pin the element from the left, instead of the top? Here's where a dynamic argument that can be updated per component instance comes in very handy:

```vue-html
<div id="dynamicexample">
  <h3>Scroll down inside this section ↓</h3>
  <p v-pin:[direction]="200">I am pinned onto the page at 200px to the left.</p>
</div>
```

```js
const app = Vue.createApp({
  data() {
    return {
      direction: 'right'
    }
  }
})

app.directive('pin', {
  mounted(el, binding) {
    el.style.position = 'fixed'
    // binding.arg is an argument we pass to directive
    const s = binding.arg || 'top'
    el.style[s] = binding.value + 'px'
  }
})

app.mount('#dynamic-arguments-example')
```

Result:

<p class="codepen" data-height="300" data-theme-id="39028" data-default-tab="result" data-user="Vue" data-slug-hash="YzXgGmv" data-editable="true" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="Custom directives: dynamic arguments">
  <span>See the Pen <a href="https://codepen.io/team/Vue/pen/YzXgGmv">
  Custom directives: dynamic arguments</a> by Vue (<a href="https://codepen.io/Vue">@Vue</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

Our custom directive is now flexible enough to support a few different use cases. To make it even more dynamic, we can also allow to modify a bound value. Let's create an additional property `pinPadding` and bind it to the `<input type="range">`

```vue-html{4}
<div id="dynamicexample">
  <h2>Scroll down the page</h2>
  <input type="range" min="0" max="500" v-model="pinPadding">
  <p v-pin:[direction]="pinPadding">Stick me {{ pinPadding + 'px' }} from the {{ direction }} of the page</p>
</div>
```

```js{5}
const app = Vue.createApp({
  data() {
    return {
      direction: 'right',
      pinPadding: 200
    }
  }
})
```

Now let's extend our directive logic to recalculate the distance to pin on component update:

```js{7-10}
app.directive('pin', {
  mounted(el, binding) {
    el.style.position = 'fixed'
    const s = binding.arg || 'top'
    el.style[s] = binding.value + 'px'
  },
  updated(el, binding) {
    const s = binding.arg || 'top'
    el.style[s] = binding.value + 'px'
  }
})
```

Result:

<p class="codepen" data-height="300" data-theme-id="39028" data-default-tab="result" data-user="Vue" data-slug-hash="rNOaZpj" data-editable="true" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="Custom directives: dynamic arguments + dynamic binding">
  <span>See the Pen <a href="https://codepen.io/team/Vue/pen/rNOaZpj">
  Custom directives: dynamic arguments + dynamic binding</a> by Vue (<a href="https://codepen.io/Vue">@Vue</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

## Function Shorthand

In previous example, you may want the same behavior on `mounted` and `updated`, but don't care about the other hooks. You can do it by passing the callback to directive:

```js
app.directive('pin', (el, binding) => {
  el.style.position = 'fixed'
  const s = binding.arg || 'top'
  el.style[s] = binding.value + 'px'
})
```

## Object Literals

If your directive needs multiple values, you can also pass in a JavaScript object literal. Remember, directives can take any valid JavaScript expression.

```vue-html
<div v-demo="{ color: 'white', text: 'hello!' }"></div>
```

```js
app.directive('demo', (el, binding) => {
  console.log(binding.value.color) // => "white"
  console.log(binding.value.text) // => "hello!"
})
```

## Usage on Components

In 3.0, with fragments support, components can potentially have more than one root nodes. This creates an issue when a custom directive is used on a component with multiple root nodes.

To explain the details of how custom directives will work on components in 3.0, we need to first understand how custom directives are compiled in 3.0. For a directive like this:

```vue-html
<div v-demo="test"></div>
```

Will roughly compile into this:

```js
const vDemo = resolveDirective('demo')

return withDirectives(h('div'), [[vDemo, test]])
```

Where `vDemo` will be the directive object written by the user, which contains hooks like `mounted` and `updated`.

`withDirectives` returns a cloned VNode with the user hooks wrapped and injected as VNode lifecycle hooks (see [Render Function](render-function.html) for more details):

```js
{
  onVnodeMounted(vnode) {
    // call vDemo.mounted(...)
  }
}
```

**As a result, custom directives are fully included as part of a VNode's data. When a custom directive is used on a component, these `onVnodeXXX` hooks are passed down to the component as extraneous props and end up in `this.$attrs`.**

This also means it's possible to directly hook into an element's lifecycle like this in the template, which can be handy when a custom directive is too involved:

```vue-html
<div @vnodeMounted="myHook" />
```

This is consistent with the [attribute fallthrough behavior](component-attrs.html). So, the rule for custom directives on a component will be the same as other extraneous attributes: it is up to the child component to decide where and whether to apply it. When the child component uses `v-bind="$attrs"` on an inner element, it will apply any custom directives used on it as well.
