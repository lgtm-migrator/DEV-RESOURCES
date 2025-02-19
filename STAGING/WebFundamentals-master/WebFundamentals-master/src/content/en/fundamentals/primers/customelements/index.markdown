---
layout: shared/narrow
title: "Using custom elements to build reusable web components"
description: "Custom elements allow web developers to define new HTML tags, extend existing ones, and create reusable web components."
published_on: 2016-06-29
updated_on: 2016-06-29
authors:
  - ericbidelman
translation_priority: 1
order: 4
notes:
  extend:
    - "Some browsers have expressed distaste for implementing the <code>is=\"\"</code> syntax. This is unfortunate for accessibility and progressive enhancement. If you think extending native HTML elements is useful, voice your thoughts <a href='https://github.com/w3c/webcomponents/issues/509'>on Github</a>."
  children:
    - Overwriting an element's children with new content is generally not a good idea because it's unexpected. Users would be surprised to have their markup thrown out. A better way to add element-defined content is to use shadow DOM, which we'll talk about next.
---

<style>
  .forward {
    font-size: 14px;
    padding: 16px;
    background-color: #FFF9C4;
    border-radius: 3px;
  }
  .forward p {
    margin-bottom: 0;
  }
  .wf-highlight-list--note {
    margin-top: 24px;
  }
  .wf-talkinghead::before {
    background-image: url(/web/imgs/contributors/{{page.authors[0]}}.jpg);
  }
</style>

<!-- <script>
function supportsCustomElements() {
  return 'customElements' in window;
}
</script>
 -->

<div class="forward">
  <b>Already familiar with custom elements?</b>
  <p>This article describes the new <a href="https://html.spec.whatwg.org/multipage/scripting.html#custom-elements" target="_blank">Custom Elements v1 spec</a>. If you've been using custom elements, chances are you're familiar with the <a href="https://www.chromestatus.com/features/4642138092470272">v0 version shipped in Chrome 33</a>. The concepts are the same, but the v1 spec has important API differences. Keep reading to see what's new or check out the section on <a href="#historysupport">History and browser support</a> for more info.</p>
</div>

### TL;DR {#tldr}

With [Custom Elements][spec], web developers can **create new HTML tags**,
beef-up existing HTML tags, or extend the components other developers have authored.
The API is the foundation of [web components](http://webcomponents.org/). It brings
a web standards-based way to create reusable components using nothing more than
vanilla JS/HTML/CSS. The result is less code, modular code, and more reuse in our apps.

## Introduction {#intro}

{% include shared/toc.liquid %}

The browser gives us an excellent tool for structuring web applications.
It's called HTML.  You may have heard of it! It's declarative, portable, well supported, and easy to work with. Great as HTML may be, its vocabulary and extensibility are limited. [HTML living standard](https://html.spec.whatwg.org/multipage/) lacks a way to automatically associate JS behavior with your markup...until now.


Custom elements are the answer to modernizing HTML; filling in the missing pieces,
and bundling structure with behavior. If HTML doesn't provide the solution to a problem,
we can create a custom element that does. **Custom elements teach the browser new tricks while preserving the benefits of HTML**.

## Defining a new element {#define}

To define a new HTML element we need the power of JavaScript!

The `customElements` interface is used for defining a custom element and teaching
the browser about a new tag. Call `customElements.define()` with the tag name you want
to create and a JavaScript `class` that extends the base `HTMLElement`.

**Example** - defining a mobile drawer panel, `<app-drawer>`:

{% highlight javascript %}
class AppDrawer extends HTMLElement {...}
window.customElements.define('app-drawer', AppDrawer);

// Or use an anonymous class if you don't want a named constructor in current scope.
window.customElements.define('app-drawer', class extends HTMLElement {...});
{% endhighlight %}

Example usage:

{% highlight html %}
<app-drawer></app-drawer>
{% endhighlight %}

It's important to remember that using a custom element is no different than using a `<div>` or any other element. Instances can be declared on the page, created dynamically in JavaScript, event listeners can be attached, etc. Keep reading for more examples.

### Defining an element's JavaScript API {#jsapi}

The functionality of a custom element is defined using an ES2015 [`class`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes) which extends `HTMLElement`.
Extending `HTMLElement` ensures the custom element inherits the entire DOM API and
means any properties/methods that you add to the class become part of the element's DOM interface.
Essentially, use the class to create a **public JavaScript API** for your tag.

**Example** - defining the DOM interface of `<app-drawer>`:

{% highlight javascript %}
class AppDrawer extends HTMLElement {

  // A getter/setter for an open property.
  get open() {
    return this.hasAttribute('open');
  }

  set open(val) {
    // Reflect the value of the open property as an HTML attribute.
    if (val) {
      this.setAttribute('open', '');
    } else {
      this.removeAttribute('open');
    }
    this.toggleDrawer();
  }

  // A getter/setter for a disabled property.
  get disabled() {
    return this.hasAttribute('disabled');
  }

  set disabled(val) {
    // Reflect the value of the disabled property as an HTML attribute.
    if (val) {
      this.setAttribute('disabled', '');
    } else {
      this.removeAttribute('disabled');
    }
  }

  // Can define constructor arguments if you wish.
  constructor() {
    // If you define a ctor, always call super() first!
    // This is specific to CE and required by the spec.
    super();

    // Setup a click listener on <app-drawer> itself.
    this.addEventListener('click', e => {
      // Don't toggle the drawer if it's disabled.
      if (this.disabled) {
        return;
      }
      this.toggleDrawer();
    });
  }

  toggleDrawer() {
    ...
  }
}

customElements.define('app-drawer', AppDrawer);
{% endhighlight %}

In this example, we're creating a drawer that has an `open` property, `disabled` property,
and a `toggleDrawer()` method. It also [reflects properties as HTML attributes](#reflectattr).

A neat feature of custom elements is that **`this` inside a class definition refers to
the DOM element itself** i.e. the instance of the class. In our example, `this` refers to `<app-drawer>`. This (😉) is how the element can attach a `click` listener to itself! And you're not limited to event listeners. The entire DOM API is available inside element code. Use `this` to access the element's properties, inspect its children (`this.children`), query nodes (`this.querySelectorAll('.items')`), etc.

**Rules on creating custom elements**

1. The name of a custom element **must contain a dash (-)**. So `<x-tags>`, `<my-element>`, and `<my-awesome-app>` are all valid names, while `<tabs>` and `<foo_bar>` are not. This requirement is so the HTML parser can distinguish custom elements from regular elements. It also ensures forward compatibility when new tags are added to HTML.
2. You can't register the same tag more than once. Attempting to do so will throw a `DOMException`. Once you've told the browser about a new tag, that's it. No take backs.
3. Custom elements cannot be self-closing because HTML only allows <a href='https://www.w3.org/TR/html-markup/syntax.html#void-elements' target='blank'>a few elements</a> to be self-closing. Always declare a closing tag (<code>&lt;app-drawer&gt;&lt;/app-drawer&gt;</code>".

## Extending elements {#extend}

The Custom Elements API is useful for creating new HTML elements, but it's also
useful for extending other custom elements or even the browser's built-in HTML.

### Extending a custom element {#extendcustomeel}

Extending another custom element is done by extending its class definition.

**Example** - create `<fancy-app-drawer>` that extends `<app-drawer>`:

{% highlight javascript %}
class FancyDrawer extends AppDrawer {
  constructor() {
    super(); // always call super() first in the ctor. This also calls the extended class' ctor.
    ...
  }

  toggleDrawer() {
    // Possibly different toggle implementation?
    // Use ES2015 if you need to call the parent method.
    // super.toggleDrawer()
  }

  anotherMethod() {
    ...
  }
}

customElements.define('fancy-app-drawer', FancyDrawer);
{% endhighlight %}

### Extending native HTML elements {#extendhtml}

Let's say you wanted to create a fancier `<button>`. Instead of replicating the behavior and
functionality of `<button>`, a better option is to progressively enhance the existing element using custom elements.

A **customized built-in element** is a custom element that extends one of the browser's
built-in HTML tags. The primary benefit of extending an existing element is to
gain all of its features (DOM properties, methods, accessibility). There's no better way to write a [progressive web app](https://developers.google.com/web/progressive-web-apps/) than to **progressively enhance existing HTML elements**.

To extend an element, you'll need to create a class definition that inherits from
the correct DOM interface. For example, a custom element that extends `<button>`
needs to inherit from `HTMLButtonElement` instead of `HTMLElement`. Similarly, an
element that extends `<img>` needs to extend `HTMLImageElement`.

**Example** - extending `<button>`:

{% highlight javascript %}
// See https://html.spec.whatwg.org/multipage/indices.html#element-interfaces
// for the list of other DOM interfaces.
class FancyButton extends HTMLButtonElement {
  constructor() {
    super(); // always call super() first in the ctor.
    this.addEventListener('click', e => this.drawRipple(e.offsetX, e.offsetY));
  }

  // Material design ripple animation.
  drawRipple(x, y) {
    let div = document.createElement('div');
    div.classList.add('ripple');
    this.appendChild(div);
    div.style.top = `${y - div.clientHeight/2}px`;
    div.style.left = `${x - div.clientWidth/2}px`;
    div.style.backgroundColor = window.getComputedStyle(this).color;
    div.classList.add('run');
    div.addEventListener('transitionend', e => div.remove());
  }
}

customElements.define('fancy-button', FancyButton, {extends: 'button'});
{% endhighlight %}

Notice that the call to `define()` changes slightly when extending a native element. The required third parameter tells the browser which tag you're extending. This is necessary because many HTML tags share the same DOM interface. `<section>`, `<address>`, and `<em>` (among others) all share `HTMLElement`; both `<q>` and `<blockquote>` share `HTMLQuoteElement`; etc.. Specifying `{extends: 'blockquote'}` lets the browser know you're creating a souped-up `<blockquote>` instead of a `<q>`. See [the HTML spec](https://html.spec.whatwg.org/multipage/indices.html#element-interfaces) for
the full list of HTML's DOM interfaces.

> Extending `HTMLButtonElement` endows our fancy button with all the DOM properties/methods
of `<button>`. That checks off a bunch of stuff we don't have to implement ourselves: `disabled` property, `click()` method, `keydown` listeners, `tabindex` management. Instead, our focus can be progressively enhancing `<button>` with custom functionality, namely, the `drawRipple()` method. Less code, more reuse!
{: .wf-talkinghead }

Consumers of a customized built-in element can use it in several ways.
They can declare it by adding the `is=""` attribute on the native tag:

{% highlight html %}
<!-- This <button> is a fancy button. -->
<button is="fancy-button" disabled>Fancy button!</button>
{% endhighlight %}

create an instance in JavaScript:

{% highlight javascript %}
// Custom elements overload createElement() to support the is="" attribute.
let button = document.createElement('button', {is: 'fancy-button'});
button.textContent = 'Fancy button!';
button.disabled = true;
document.body.appendChild(button);
{% endhighlight %}

or use the `new` operator:

{% highlight javascript %}
let button = new FancyButton();
button.textContent = 'Fancy button!';
button.disabled = true;
{% endhighlight %}

Here's another example that extends `<img>`.

**Example** - extending `<img>`:

{% highlight javascript %}
customElements.define('bigger-img', class extends Image {
  constructor(width, height) {
    super(width * 10, height * 10);
  }
}, {extends: 'img'});
{% endhighlight %}

Users declare this component as:

{% highlight html %}
<!-- This <img> is a bigger img. -->
<img is="bigger-img">
{% endhighlight %}

or create an instance in JavaScript:

{% highlight html %}
const BiggerImage = customElements.get('bigger-img');
const image = new BiggerImage(15, 20); // pass ctor values like so.
console.assert(image.width === 150);
console.assert(image.height === 200);
{% endhighlight %}

{% include shared/note.liquid list=page.notes.extend %}

## Custom element reactions {#reactions}

A custom element can define special lifecycle hooks for running code during
interesting times of its existence. These are called **custom element reactions**.

<table class="table">
  <thead>
    <tr>
      <th>Name</th>
      <th>Called when</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>constructor</code></td>
      <td>An instance of the element is created or <a href="#upgrades">upgraded</a>. Useful for initializing state, settings up event listeners, or <a href="#shadowdom">creating shadow dom</a>. See the <a href="https://html.spec.whatwg.org/multipage/scripting.html#custom-element-conformance">spec</a> for restrictions on what you can do in the <code>constructor</code>.</td>
    </tr>
    <tr>
      <td><code>connectedCallback</code></td>
      <td>Called every time the element is inserted into the DOM. Useful for running setup code, such as fetching resources or rendering. Generally, you should try to delay work until this time.</td>
    </tr>
    <tr>
      <td><code>disconnectedCallback</code></td>
      <td>Called every time the element is removed from the DOM. Useful for running clean up code (removing event listeners, etc.).</td>
    </tr>
    <tr>
      <td><code>attributeChangedCallback(attrName, oldVal, newVal)</code></td>
      <td>An attribute was added, removed, updated, or replaced. Also called for initial values when an element is created by the parser, or <a href="#upgrades">upgraded</a>. <b>Note:</b> only attributes listed in the <code>observedAttributes</code> property will receive this callback.</td>
    </tr>
  </tbody>
</table>

The browser calls the `attributeChangedCallback()` for any attributes whitelisted
in the `observedAttributes` array (see [Observing changes to attributes](#attrchanges)).
Essentially, this is a performance optimization. When users change a common
attribute like `style` or `class`, you don't want to be spammed with tons of callbacks.

**Reaction callbacks are synchronous**. If someone calls `el.setAttribute(...)`
on your element, the browser will immediately call `attributeChangedCallback()`. Similarly,
you'll receive a `disconnectedCallback()` right after your element is removed from
the DOM (e.g. the user calls `el.remove()`).

**Example:** adding custom element reactions to `<app-drawer>`:

{% highlight javascript %}
class AppDrawer extends HTMLElement {
  constructor() {
    super(); // always call super() first in the ctor.
    ...
  }
  connectedCallback() {
    ...
  }
  disconnectedCallback() {
    ...
  }
  attributeChangedCallback(attrName, oldVal, newVal) {
    ...
  }
}
{% endhighlight %}

Define reactions if/when it make senses. If your element is sufficiently complex and opens a connection to IndexedDB in `connectedCallback()`, do the necessary cleanup work in `disconnectedCallback()`. But be careful! You can't rely on your element being removed from the DOM in all circumstances. For example, `disconnectedCallback()` will never be called if the user closes the tab.

## Properties and attributes

### Reflecting properties to attributes {#reflectattr}

It's common for HTML properties to reflect their value back to the DOM as an HTML attribute.
For example, when the values of `hidden` or `id` are changed in JS:

{% highlight javascript %}
div.id = 'my-id';
div.hidden = true;
{% endhighlight %}

the values are applied to the live DOM as attributes:

{% highlight html %}
<div id="my-id" hidden>
{% endhighlight %}

This is called "[reflecting properties to attributes](https://www.w3.org/TR/html5/infrastructure.html#reflect)". Almost every property in HTML does this. Why? Attributes are also useful for configuring an
element declaratively and certain APIs like accessibility and CSS selectors rely on attributes to work.

Reflecting a property is useful anywhere you want to **keep the element's DOM
representation in sync with its JavaScript state**. One reason you might want
to reflect a property is so user-defined styling applies when JS state changes.

Recall our `<app-drawer>`. A consumer of this component may want to fade it out
and/or prevent user interaction when it's disabled:

{% highlight css %}
app-drawer[disabled] {
  opacity: 0.5;
  pointer-events: none;
}
{% endhighlight %}

When the `disabled` property is changed in JS, we want that attribute to be added
to the DOM so the user's selector matches. The element can provide that behavior
by reflecting the value to an attribute of the same name:

{% highlight javascript %}
...

get disabled() {
  return this.hasAttribute('disabled');
}

set disabled(val) {
  // Reflect the value of `disabled` as an attribute.
  if (val) {
    this.setAttribute('disabled', '');
  } else {
    this.removeAttribute('disabled');
  }
  this.toggleDrawer();
}
{% endhighlight %}

### Observing changes to attributes {#attrchanges}

HTML attributes are a convenient way for users to declare initial state:

{% highlight html %}
<app-drawer open disabled></app-drawer>
{% endhighlight %}

Elements can react to attribute changes by defining a `attributeChangedCallback`.
The browser will call this method for every change to attributes listed in the `observedAttributes` array.

{% highlight javascript %}
class AppDrawer extends HTMLElement {
  ...

  static get observedAttributes() {
    return ['disabled', 'open'];
  }

  get disabled() {
    return this.hasAttribute('disabled');
  }

  set disabled(val) {
    if (val) {
      this.setAttribute('disabled', '');
    } else {
      this.removeAttribute('disabled');
    }
  }

  // Only called for the disabled and open attributes due to observedAttributes
  attributeChangedCallback(name, oldValue, newValue) {
    // When the drawer is disabled, update keyboard/screen reader behavior.
    if (this.disabled) {
      this.setAttribute('tabindex', '-1');
      this.setAttribute('aria-disabled', 'true');
    } else {
      this.setAttribute('tabindex', '0');
      this.setAttribute('aria-disabled', 'false');
    }
    // TODO: also react to the open attribute changing.
  }
}
{% endhighlight %}

In the example, we're setting additional attributes on the `<app-drawer>` when a
`disabled` attribute is changed. Although we're not doing it here, you could also
**use the `attributeChangedCallback` to keep a JS property in sync with its attribute**.

## Element upgrades {#upgrades}

### Progressively enhanced HTML

We've already learned that custom elements are defined by calling `customElements.define()`.
But this doesn't mean you have to define + register a custom element all in one go.

**Custom elements can be used _before_ their definition is registered**.

Progressive enhancement is a feature of custom elements. In other words, you can declare a bunch of `<app-drawer>` elements on the page and never invoke `customElements.define('app-drawer', ...)` until much later. This is because the browser treats potential custom elements differently thank [unknown tags](#unknown). The process of calling `define()` and endowing an existing element with a class definition is called "element upgrades".

To know when a tag name becomes defined, you can use `window.customElements.whenDefined()`.
It vends a Promise that resolves when the element becomes defined.

{% highlight javascript %}
customElements.whenDefined('app-drawer').then(() => {
  console.log('app-drawer defined');
});
{% endhighlight %}

**Example** - delay work until a set of child elements are upgraded

{% highlight html %}
<share-buttons>
  <social-button type="twitter"><a href="...">Twitter</a></social-button>
  <social-button type="fb"><a href="...">Facebook</a></social-button>
  <social-button type="plus"><a href="...">G+</a></social-button>
</share-buttons>
{% endhighlight %}

{% highlight javascript %}
// Fetch all the children of <share-buttons> that are not defined yet.
let undefinedButtons = buttons.querySelectorAll(':not(:defined)');

let promises = [...undefinedButtons].map(socialButton => {
  return customElements.whenDefined(socialButton.localName);
));

// Wait for all the social-buttons to to be upgraded.
Promise.all(promises).then(() => {
  // All social-button children are ready.
});
{% endhighlight %}

> I think of custom elements as being in a state of limbo before they're defined. The [spec](https://dom.spec.whatwg.org/#concept-element-custom-element-state) defines an element's state as "undefined", "uncustomized", or "custom". Built-in elements like `<div>` are always "defined".
{: .wf-talkinghead }

## Element-defined content {#addingmarkup}

Custom elements can manage their own content by using the DOM APIs inside element code. [Reactions](#reactions) come in handy for this.

**Example** - create an element with some default HTML:

{% highlight javascript %}
customElements.define('x-foo-with-markup', class extends HTMLElement {
  connectedCallback() {
    this.innerHTML = "<b>I'm an x-foo-with-markup!</b>";
  }
  ...
});
{% endhighlight %}

Declaring this tag will produce:

{% highlight html %}
<x-foo-with-markup>
 <b>I'm an x-foo-with-markup!</b>
</x-foo-with-markup>
{% endhighlight %}

<!-- <div class="demoarea">
  <x-foo-with-markup></x-foo-with-markup>
</div>

<script>
if (supportsCustomElements()) {
  customElements.define('x-foo-with-markup', class extends HTMLElement {
    connectedCallback() {
      this.innerHTML = "<b>I'm an x-foo-with-markup!</b>";
    }
  });
}
</script> -->

{% include shared/note.liquid list=page.notes.children %}

### Creating an element that uses Shadow DOM {#shadowdom}

> I'm not going to cover the features of [Shadow DOM][sd_spec] in this article, but
it's a powerful API to combine with custom elements. By itself, Shadow DOM is
composition tool. When it's used in conjunction with custom elements, magical things happen.
{: .wf-talkinghead }

Shadow DOM provides a way for an element to own, render, and style a chunk of DOM
that's separate from the rest of the page. Heck, you could even hide away an
entire app within a single tag:

{% highlight html %}
<!-- chat-app's implementation details are hidden away in Shadow DOM. -->
<chat-app></chat-app>
{% endhighlight %}

To use Shadow DOM in a custom element, call `this.attachShadow` inside your `constructor`:

    customElements.define('x-foo-shadowdom', class extends HTMLElement {
      constructor() {
        super(); // always call super() first in the ctor.

        // Attach a shadow root to the element.
        let shadowRoot = this.attachShadow({mode: 'open'});
        shadowRoot.innerHTML = `
          <style>:host { ... }</style> <!-- look ma, scoped styles -->
          <b>I'm in shadow dom!</b>
          <slot></slot>
        `;
      }
      ...
    });

Example usage:

{% highlight html %}
<x-foo-shadowdom>
  <p><b>User's</b> custom text</p>
</x-foo-shadowdom>

<!-- renders as -->
<x-foo-shadowdom>
  <b>I'm in shadow dom!</b>
  <p><b>User's</b> custom text</p>
</x-foo-shadowdom>

{% endhighlight %}

<!-- <div class="demoarea">
  <x-foo-shadowdom>
    <p><b>User's</b> custom text</p>
  </x-foo-shadowdom>
</div>

<script>
if (supportsCustomElements()) {
  customElements.define('x-foo-shadowdom', class extends HTMLElement {
    constructor() {
      super(); // always call super() first in the ctor.
      let shadowRoot = this.attachShadow({mode: 'open'});
      shadowRoot.innerHTML = `
        <b>I'm in shadow dom!</b>
        <slot></slot>
      `;
    }
  });
}
</script> -->

### Creating elements from a &lt;template&gt; {#fromtemplate}

For those unfamiliar, the [`<template>` element](http://www.whatwg.org/specs/web-apps/current-work/multipage/scripting-1.html#the-template-element) allows you to declare fragments of DOM which are parsed, inert at page load, and can be activated later at runtime. It's another API primitive in the web components family. **Templates are an ideal placeholder for declaring the structure of a custom element**.

**Example:** registering an element with Shadow DOM content created from a `<template>`:

{% highlight html %}
<template id="x-foo-from-template">
  <style>
    p { color: orange; }
  </style>
  <p>I'm in Shadow DOM. My markup was stamped from a &lt;template&gt;.</p>
</template>

<script>
  customElements.define('x-foo-from-template', class extends HTMLElement {
    constructor() {
      super(); // always call super() first in the ctor.
      let shadowRoot = this.attachShadow({mode: 'open'});
      const t = document.querySelector('#x-foo-from-template');
      const instance = t.content.cloneNode(true);
      shadowRoot.appendChild(instance);
    }
    ...
  });
</script>
{% endhighlight %}

These few lines of code pack a punch. Let's understanding the key things going on:

1. We're defining a new element in HTML: `<x-foo-from-template>`
2. The element's Shadow DOM is created from a `<template>`
3. The element's DOM is local to the element thanks to Shadow DOM
4. The element's internal CSS is scoped to the element thanks to Shadow DOM

<!-- <div class="demoarea">
  <x-foo-from-template></x-foo-from-template>
</div>

<template id="x-foo-from-template">
  <style>:host p { color: orange; }</style>
  <p>I'm in Shadow DOM. My markup was stamped from a &lt;template&gt;.</p>
</template>

<script>
if (supportsCustomElements()) {
  customElements.define('x-foo-from-template', class extends HTMLElement {
    constructor() {
      super();
      let shadowRoot = this.attachShadow({mode: 'open'});
      const t = document.querySelector('#x-foo-from-template');
      shadowRoot.appendChild(t.content.cloneNode(true));
    }
  });
}
</script> -->

## Styling a custom element {#styling}

Even if your element defines its own styling using Shadow DOM, users can
style your custom element from their page. These are called "user-defined styles".

{% highlight html %}
<!-- user-defined styling -->
<style>
  app-drawer {
    display: flex;
  }
  panel-item {
    transition: opacity 400ms ease-in-out;
    opacity: 0.3;
    flex: 1;
    text-align: center;
    border-radius: 50%;
  }
  panel-item:hover {
    opacity: 1.0;
    background: rgb(255, 0, 255);
    color: white;
  }
  app-panel > panel-item {
    padding: 5px;
    list-style: none;
    margin: 0 7px;
  }
</style>

<app-drawer>
  <panel-item>Do</panel-item>
  <panel-item>Re</panel-item>
  <panel-item>Mi</panel-item>
</app-drawer>
{% endhighlight %}

You might be asking yourself how CSS specificity works if the element has
styles defined within Shadow DOM. In terms of specificity, user styles win.
They'll always override element-defined styling. See the section on [Creating an element that uses Shadow DOM](#shadowdom).

### Pre-styling unregistered elements {#prestyle}

Before an element is [upgraded](#upgrades) you can target it in CSS using the `:defined` pseudo-class.
This is useful for pre-styling a component. For example, you may wish to prevent
layout or other visual FOUC by hiding undefined components and fading them in
when they become defined.

**Example** - hide `<app-drawer>` before it's defined:

{% highlight css %}
app-drawer:not(:defined) {
  /* Pre-style, give layout, replicate app-drawer's eventual styles, etc. */
  display: inline-block;
  height: 100vh;
  opacity: 0;
  transition: opacity 0.3s ease-in-out;
}
{% endhighlight %}

After `<app-drawer>` becomes defined, the selector (`app-drawer:not(:defined)`)
no longer matches.

## Misc details {#details}

### Unknown elements vs. undefined custom elements {#unknown}

HTML is lenient and flexible to work with. For example, declare `<randomtagthatdoesntexist>` on a page and the browser is perfectly happy accepting it. Why do non-standard tags work? The answer is the [HTML specification](http://www.whatwg.org/specs/web-apps/current-work/multipage/elements.html#htmlunknownelement) allows it. Elements that are not defined by the specification get parsed as `HTMLUnknownElement`.

The same is not true for custom elements. Potential custom elements are parsed as
an `HTMLElement` if they're created with a valid name (includes a "-"). You can check this in a browser that supports custom elements. Fire up the Console: <span class="kbd">Ctrl</span>+<span class="kbd">Shift</span>+<span class="kbd">J</span> (or <span class="kbd">Cmd</span>+<span class="kbd">Opt</span>+<span class="kbd">J</span> on Mac) and paste in the following lines of code:

{% highlight javascript %}
// "tabs" is not a valid custom element name
document.createElement('tabs') instanceof HTMLUnknownElement === true

// "x-tabs" is a valid custom element name
document.createElement('x-tabs') instanceof HTMLElement === true
{% endhighlight %}

## API reference

The `customElements` interface defines useful methods for working with custom elements.

**`define(tagName, constructor, options)`**

Defines a new custom element in the browser.

Example

{% highlight javascript %}
customElements.define('my-app', class extends HTMlElement { ... });
customElements.define(
  'fancy-button', class extends HTMlButtonElement { ... }, {extends: 'button'});
{% endhighlight %}

**`get(tagName)`**

Given a valid custom element tag name, returns the element's constructor. Returns `undefined`
if no element definition has been registered.

Example

{% highlight javascript %}
let Drawer = customElements.get('app-drawer');
let drawer = new Drawer();
{% endhighlight %}

**`whenDefined(tagName)`**

Returns a Promise that resolves when the custom element is defined. If the element
is already defined, resolve immediately. Rejects if the tag name is not a valid
custom element name

Example

{% highlight javascript %}
customElements.whenDefined('app-drawer').then(() => {
  console.log('ready!');
});
{% endhighlight %}

## History and browser support {#historysupport}

If you've been following web components for the last couple of years, you'll know that
Chrome 36+ implemented a version of the Custom Elements API that uses `document.registerElement()`
instead of `customElements.define()`. That's now considered a deprecated version of
the standard, called v0. `customElements.define()` is the new hotness and what
browser vendors are starting to implement. It's called Custom Elements v1.

If you happen to be interested in the old v0 spec, check out the [html5rocks article](http://www.html5rocks.com/en/tutorials/webcomponents/customelements/).

### Browser support

Chrome ([status](https://www.chromestatus.com/features/4696261944934400)) has an implementation of
the Custom Elements v1 under a runtime flag: `--enable-blink-features=CustomElementsV1`. Safari has [begun prototyping](https://bugs.webkit.org/show_bug.cgi?id=150225). Edge has [begun prototyping](https://twitter.com/AaronGustafson/status/717028669948977153). Mozilla has an [open bug](https://bugzilla.mozilla.org/show_bug.cgi?id=889230) to implement.

Until browser support is widely available, there's a [polyfill](https://github.com/webcomponents/webcomponentsjs/tree/v1) for Custom Elements v1. **Note**: the `:define` CSS pseudo-class is not polyfilled.

To feature detect custom elements, check for the existence of `window.customElements`:

{% highlight javascript %}
function supportsCustomElements() {
  return 'customElements' in window;
}

if (supportsCustomElements()) {
  // Good to go!
} else {
  // Use polyfills
}
{% endhighlight %}

## Conclusion

Custom elements give us a new tool for defining new HTML tags in the browser and creating reusable
components. Combine them with the other new platform primitives like Shadow DOM and `<template>`, and we start to realize the grand picture of Web Components:

- Cross-browser (web standard) for creating and extending reusable components.
- Requires no library or framework to get started. Vanilla JS/HTML FTW!
- Provides a familiar programming model. It's just DOM/CSS/HTML.
- Works well with other new web platform features (Shadow DOM, `<template>`, CSS custom properties, etc.)
- Tightly integrated with the browser's DevTools.
- Leverage existing accessibility features.
- ...

<div class="mdl-grid">
  <div class="mdl-cell--3-col">
    <h6>Published</h6>
    <time datetime="{{ page.published_on | date: '%F' }}">{{ page.published_on | date: '%B %-d, %Y' }}</time>
  </div>
  <div class="mdl-cell--2-col">
    <h6>Updated</h6>
    <time datetime="{{ page.updated_on | date: '%F' }}">{{ page.updated_on | date: '%B %-d, %Y' }}</time>
  </div>
</div>

<!-- <script>
if (!supportsCustomElements()) {
  let demos = document.querySelectorAll('.demoarea');
  Array.from(demos).forEach(function(demo) {
    demo.hidden = true;
  });
}
</script> -->

[spec]: https://html.spec.whatwg.org/multipage/scripting.html#custom-elements
[sd_spec]: http://w3c.github.io/webcomponents/spec/shadow/
