
# Custom Element Default Style
The proposal is to allow web authors to define a set of styles to apply to a custom element as an option to `customElements.define`. 

## Background
Currently, in order to apply a style to custom element, you need to use a shadow root and put the style in it, even when you don’t need any content inside the shadow root.

For example:
```js
class MyElement extends HTMLElement {
  constructor() {
    super();
    const shadowRoot = this.attachShadow({mode: 'open'});
    shadowRoot.innerHTML = '<style>:host { color: red; }</style>';
  }
};
window.customElements.define('my-element', MyElement);
```
The tree structure in the above case:
```
my-element
 #shadow root
  <style> :host { color: red; } </style>
```
This presents several problems:

* You need a shadow root just for styling the element itself.
* Constructing a custom element instance involves attaching a shadow root to the element and putting the style in the shadow root, and the style is parsed every time an instance of the custom element type is created. This can be significantly slower than a native element. 
* If many instances of the same custom element are created, the same stylesheet is duplicated many times, causing performance and memory hit.

To style a custom element that does not otherwise need a shadowRoot incurs an unfortunate performance penalty and is cumbersome.  Providing the abilty to assign styles to a custom element at define time gives the platform an opportunity to optimize beyond what could be achieved when interpreting user code that installs the shadowRoot, style, and slot at construct/connected time.


## Example

```html
<style id='redColor'>
  my-element {
    color: red;
  }
</style>
<style id='blueBackground'>
  * {
    background-color: blue;
  }
</style>
<script>
  class MyElement extends HTMLElement {
    constructor() {
      super();
    }
  };
  let colorStyleSheet = document.querySelector('#redColor').sheet;
  let backgroundColorStyleSheet = document.querySelector('#blueBackground').sheet;
  window.customElements.define('my-element', MyElement, { styles: [colorStyleSheet, backgroundColorStyleSheet] });
</script>
<my-element>This text should be red with blue background</my-element>
```

## Behavior

* The default styles for every custom element with the same definition is the same.
* The default styles can be modified afterwards by modifying the stylesheet objects through CSSOM, and the change will be reflected.
	* For example, if we do this after the example above:
		```js
		colorStyleSheet.deleteRule(0); // delete the rule that colors the text red
		```
	  `my-element` will have the normal color instead of red.
* The default styles will borrow Shadow DOM cascading order. The default styles will be treated as if they are the first stylesheets in the custom element's shadow root's stylesheets, if exists.
	* For example, if we do this:
		```js
		let shadowRoot = myElement.attachShadowRoot({ mode: 'open'});
		shadowRoot.innerHTML = '<style> :host { color: green; } </style>';
		```
		Then the text in `my-element` will be colored green instead, because the shadow root style has priority since it comes later in the stylesheet list.
* Default style can only affect the custom element itself, not its descendants. 
* Style inheritance will work as usual.
* Simple selectors and compound selectors match custom element from its default style, but complex selectors do not match anything. Only the element itself is ever matched.
	* The following can match the custom element:
		* universal selector (`*`) 
		* class/id/tag name selector that match custom element:
		`.element, #element, my-element`
		* compound selectors that match custom element:
		`.element.el`
	* The following will not match:
		* class/id/tag name selectors that don’t match custom elements
		* compound selectors that don't match custom element:
		`.element.notElement`
		* complex selectors of any kind:
		`* > div, * div, * ~ div`

