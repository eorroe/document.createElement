# `document.createElement()` Upgrade Proposal

## Usage

```JS
document.createElement('div');
```
Returns:
```HTML
<div></div>
```

```JS
document.createElement('div#id.class1.class2');

document.createElement('div', {
  id: 'id',
  className: 'class1 class2'
});
```
Both Return:
```HTML
<div id="id" class="class1 class2"></div>
```

Passing the `{ id: 'id', className: 'class1 class2' }` overrides the string passed `div#id.class1.class2`.

**So DON'T USE BOTH**

```JS
document.createElement('div', {
  'attributes': {
    'id': 'id',
    'class': 'class1 class2',
    'data-index': 1
  },
  'dataset': {
    'num': 5
  },
  'style': {
    'color': 'lightblue',
    'background': 'lightgrey',
    'fontSize': '20px'
  },
  'innerHTML': '<h1 class="text">Hello World</h1>',
  'onclick': function() {...}
  // etc
});
```
Returns:
```HTML
<div id="id" class="class1 class2" data-index="1" data-num="5" style="color: rgb(173, 216, 230); font-size: 20px; background: rgb(211, 211, 211);">
  <h1 class="text">Hello World</h1>
</div>
```


## 2 Ways of doing this:

```JS
var cachedCreateElement = Document.prototype.createElement;

Document.prototype.createElement = function(tagName, props) {
	var splitTagName = tagName.match(/(^|[\.\#])[^\.\#]*/g);
	var element = cachedCreateElement.call(this, splitTagName[0]);
	for(var i = 0, l = splitTagName.length; i < l; i++ ) {
		var str = splitTagName[i];
		if(str[0] === '#') {
			element.id = str.slice(1);
		} else if(str[0] === '.') {
			element.classList.add(str.slice(1));
		}
	}
	for(var prop in props) {
		switch(prop) {
			case 'attributes':
				for(var attr in props[prop]) element.setAttribute(attr, props[prop][attr]);
			break;
			case 'dataset':
			case 'style':
				for(var key in props[prop]) element[prop][key] = props[prop][key];
			break;
			default:
				element[prop] = props[prop];
			break;
		}
	}
	return element;
}
```

This would not have to be like this if the one property `attributes` on the `Element.prototype` was changed to work differently:

As of right now the following is not possible:

```JS
document.body.attributes.index = 1;
```
And have the body look like this:

```HTML
<body index="1"></body>
```

Why? because the `attributes` property is the only one that I can find on Elements that return this:
```JS
document.body.attributes;

//returns:

NamedNodeMap {0: index}
```

Meaning there indexed for some weird reason.

Now if that were changed we could have `document.createElement()` work like this:

```JS
var cachedCreateElement = Document.prototype.createElement;

Document.prototype.createElement = function(tagName, props) {
	var splitTagName = tagName.match(/(^|[\.\#])[^\.\#]*/g);
	var element = cachedCreateElement.call(this, splitTagName[0]);
	for(var i = 0, l = splitTagName.length; i < l; i++ ) {
		var str = splitTagName[i];
		if(str[0] === '#') {
			element.id = str.slice(1);
		} else if(str[0] === '.') {
			element.classList.add(str.slice(1));
		}
	}
	for(var prop in props) {
		var p = props[prop];
		if(p.constructor === Object) {
			for(var key in p) element[prop][key] = p[key];
		} else if(p.constructor === Array) {
			for(var i = 0, l = p.length; i < l; i++) element[prop][i] = p[i];
		} else {
			element[prop] = p;
		}
	}
	return element;
}
```

# Conclusion

Well this is the way I'd do it, I'm not sure how browsers would natively do it.

What you'd have to keep in mind is if a property is not settable normally:

```JS
var div = document.createElement('div'); // Using original method
div.parentElement = document.body; // This doesn't work

// So no point in doing this:
document.createElement('div', {
  parentElement: document.body,
  // or
  parentNode: document.body
});
```

Properties that aren't natively settable will not be set.
