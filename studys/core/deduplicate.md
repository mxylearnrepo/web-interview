ES5
```js
const deduplicate = arr => arr.filter((item, index) => arr.indexOf(item) === index)
```

ES6
```js
const deduplicate = arr => [...new Set(arr)]
```