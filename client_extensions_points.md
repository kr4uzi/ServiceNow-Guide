Client Extension Points are similar to Server Side Extensions but they are more difficult to handle.\
In the next lines, I will implement a extensible Event Handling mechanism. This would e.g. allow other scopes to integrate into your application.\
Typically, Client Extension Points will work together with a "manager" script, which most likely is a UI Script:
```javascript
var x_424426_cextp = x_424426_cextp || {};
x_424426_cextp.MyEvents = (function() {
	"use strict";

	/* eslint no-undef: "error" */
	const events = {};
	return {
		register: function (eventName, callback, thisArg) {
			const obj = {
				callback: callback,
				thisArg: thisArg
			};

			const handlers = (events[eventName] ??= []);
			handlers.push(obj);

			return function () {
				const index = handlers.indexOf(obj);
				if (index != -1) {
					handlers.splice(index, 1);
				}
			};
		},
		fire: function (eventName, ...args) {
			(events[eventName] || []).forEach(handler => {
				try {
					handler.callback?.apply(handler.thisArg, args);
				} catch (e) {
					// ignore errors
				}
			});
		},
		type: 'MyEvents'
	};
})();
```
This UI Script provides a simple event handling mechanism: register (and unregister) + fire

Now create a Client Side Extension Point which in our use case would be called, whenever 'my_event' is fired:
```javascript
(function () {

	x_424426_cextp.MyEvents.register('my_event', function (myArg) {
		g_form.addInfoMessage('My Event: ' + myArg);
	});

})();```
While I used the IIFE pattern, you can also follow the NOW-UI-Script-Pattern of the event manager on the top.

Loading a UI Script dynamically in a Client Script:
```javascript
ScriptLoader.getScripts(['x_424426_cextp.MyEvents.jsdbx'], function () {
  x_424426_cextp.MyEvents.register('my_event', function () {
    g_form.addInfoMessage('Hello World from Client Script');
  });
});
// Service Portal: use the g_ui_scripts (TODO: is this the right object? hardly use that...)
```
Note: A global UI Script doesn't need to be loaded like this, they are loaded on Forms and Lists by default.
... and UI Macros + UI Pages:
```html
<g:include_script src="x_424426_cextp.MyEvents.jsdbx" />
```

When you have client side extension point implementations, you have a list of scripts to load,
therefore the mechanism above (even though technically those extension piont implementations are UI Scripts) won't work.
Instead, you can use the following APIs:
```html
<g:client_extension name="MenuItemScripts" inline="false" />
```
inline="true":
```html
<script>/* Extension Point Implementation 1 */</script>
<script>/* Extension Point Implementation 2 */</script>
```
inline="false":
```html
<script src="SCOPE1.EXTPOINT1.jsdbx" />
<script src="SCOPE2.EXTPOINT2.jsdbx" />
```

You can also obtain the implementation scripts via the Server Side API:
```javascript
// Note: the scope prefix is optional
  const extName = 'x_424426_cextp.ExampleExtensionPoint';
// Note: the return value is a global Array, therefore - if executed in a scoped script - (scripts instanceof Array) == false
const scripts = new GlideClientExtensionPoint().getExtensions(extName);
scripts.forEach(script => gs.info(script));
```

Unfortunately there is no way to directly load Client Side Extension Point Implementations in a Client or UI Script.\
Also no way to retrieve the URLs of those Implemenations so we could’ve made use of the ScriptLoader API.

So either you load the Script via the Server Side API above (combined with a GlideAjax + Client Side Script Include).\
Or - if your Client Side Extension Point has "Allow access over AJAX/REST" = true) - you can load the scripts via a XMLHttpRequest:
```javascript:
function loadExtensionPoints(extPointName, extPointScope, callback) {
  var req = new XMLHttpRequest();
  req.open('get', '/api/now/clientextension/' + extPointName + '?sysparm_scope=' + extPointScope, false);
  req.setRequestHeader('X-UserToken', g_ck);
  req.setRequestHeader('Accept', 'application/json');
  req.addEventListener('load', function () {
    var response = JSON.parse(req.responseText);
    if (response && response.result) {
      response.result.forEach(function (script) {
        try {
          eval(script);
        } catch (e) {
          jslog(e);
        }
      });
    }

    if (callback) {
      callback();
    }
  });

  req.send();
}
```

A full Client Script example looks like this:
```javascript
ScriptLoader.getScripts(['x_424426_cextp.MyEvents.jsdbx'], function () {
  if (x_424426_cextp.MyEvents.extensionsLoaded) {
    x_424426_cextp.MyEvents.fire('my_event', 'myArg1');
  } else {
    loadExtensionPoints('ExampleExtensionPoint', 'x_424426_cextp', function () {
      // prevent multiple loading of the extension points by setting the extensionsLoaded flag
      x_424426_cextp.MyEvents.extensionsLoaded = true;
      x_424426_cextp.MyEvents.fire('my_event', 'myArg1');
    });
  }
});
```
