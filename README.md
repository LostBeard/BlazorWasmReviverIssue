# BlazorWasmReviverIssue
This project demonstrates an issue in Blazor Wasm framework reviver when deserializing a cross-origin window. 

### What
The reviver uses `Object.prototype.hasOwnProperty` to check if a property exists to determine if it should handle deserialization. 

### The problem
When you call `hasOwnProperty` on a cross-origin window for a non-allowed property a security exception is thrown.

### The solution
Wrap `hasOwnProperty` or the entire conditional in a try/catch to prevent the exception from crashing the Blazor Wasm -> JS interop.

### Problem code
- From running blazor app, not repo source

Current: blazor.webassembly.js
```js
const me = "__internalId";
e.attachReviver( (e, t) => t && "object" == typeof t && Object.prototype.hasOwnProperty.call(t, me) && "string" == typeof t[me] ? function(e) {
    const t = `[${fe(e)}]`;
    return document.querySelector(t)
}(t[me]) : t);
```

### Pseudo fix
- An example of what fixed code might look like so it does not throw when reviving a cross-origin window

Example fixed: blazor.webassembly.js
```js
const me = "__internalId";
e.attachReviver((e, t) => {
    try {
        if (t && "object" == typeof t && Object.prototype.hasOwnProperty.call(t, me) && "string" == typeof t[me]) {
            const r = `[${fe(t[me])}]`;
            return document.querySelector(r);
        }
    } catch (error) {
        if (error instanceof DOMException || error.name === 'SecurityError') {
            // Safely fall through if the object is a cross-origin window
        } else {
            throw error;
        }
    }
    return t;
});
```

### Test code
- This code triggers the issue if it is not fixed.
```cs
// create a new cross-origin popup window
var popup = JS.Invoke<IJSInProcessObjectReference>("open", url, "Google Auth", "height=800,width=1200");

if (popup is not null)
{
    // read the closed property using Reflect.get
    // passing popup this way passes it through the Blazor Wasm reviver
    while (!JS.Invoke<bool>("Reflect.get", popup, "closed"))
        await Task.Delay(1000);
    Log($"popup was closed");
}
```

### Patch
- The below code can be loaded into a Blazor Wasm app to runtime patch the Blazor Wasm framework call so it will not throw during revive of a cross-origin window.
```js
const originalHasOwnPropertyPrototype = Object.prototype.hasOwnProperty;
Object.prototype.hasOwnProperty = function (prop) {
    try {
        // Attempt the normal engine execution
        return originalHasOwnPropertyPrototype.apply(this, [prop]);
    } catch (error) {
        // Catch DOMException / SecurityError (e.g., cross-origin window access)
        if (error instanceof DOMException || error.name === 'SecurityError') {
            // Return false so Blazor treats it as a non-existent property
            return false;
        }
        // Re-throw genuine, unrelated engine errors
        throw error;
    }
};
```