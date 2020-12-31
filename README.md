<a name="top"></a>
## Path Tracing

This is an example of the Toolkit's ability to trace an overlay along the shortest path from one Node to another.

![Path Tracing](demo-path-tracing.png)

This page gives you an in-depth look at how the application is put together.

- [package.json](#package)
- [Page Setup](#setup)
- [Toolkit Setup](#toolkitSetup)
- [Templates](#templates)
- [Data Loading](#loading)
- [View](#view)
- [Rendering](#rendering)
- [Path Traversal](#traversal)


<a id="package"></a>
### package.json

```javascript
{
    "dependencies": {
        "font-awesome": "^4.7.0",
        "jsplumbtoolkit": "file:./jsplumbtoolkit.tgz"
    }
}

```

[TOP](#top)

---

<a id="setup"></a>
### Page Setup

#### CSS

```xml
<link href="node_modules/font-awesome/css/font-awesome.min.css" rel="stylesheet">
<link rel="stylesheet" href="node_modules/jsplumbtoolkit/dist/css/jsplumbtoolkit-defaults.css">
<link rel="stylesheet" href="node_modules/jsplumbtoolkit/dist/css/jsplumbtoolkit-demo.css">
<link rel="stylesheet" href="app.css">
```
Font Awesome, `jsplumbtoolkit-demo.css`, and `app.css` are used for this demo and are not jsPlumb Toolkit requirements. `jsplumbtoolkit-defaults.css` is recommended for 
all apps using the Toolkit, at least when you first start to build your app. This stylesheet contains sane defaults for the various widgets in the Toolkit. 

#### JS

```xml
<script src="node_modules/jsplumbtoolkit/dist/js/jsplumbtoolkit.js"></script>
<script src="app.js"></script>
```

We import `jsplumbtoolkit.js` from `node_modules` (it was listed in `package.json`). `app.js` contains the demo code; it is discussed on this page.

[TOP](#top)

---

<a id="toolkitSetup"></a>
### Toolkit Setup

```javascript
var toolkit = jsPlumbToolkit.newInstance();
```

[TOP](#top)

---

<a id="templates"></a>
### Templates

The app uses a single template, with the default ID:

```xml
<script type="jtk" id="jtk-template-default">
    <div>
        <div class="name">
            <span>${name}</span>
        </div>
    </div>
</script>
```

[TOP](#top)

---

<a id="loading"></a>
### Data Loading

Data for this application is randomly generated from a demo support class:

```javascript

var data = jsPlumbToolkitDemoSupport.randomGraph(5, 10);

toolkit.load({
  data: data
});
```

[TOP](#top)

---

<a name="view"></a>
### View

In the View, we configure the appearance of Edges and Nodes, and we also handle node clicks to instigate path traversals. See [below](#traversal) for a discussion of how path traversals are handled.
 
 
 ```javascript
 
 var transport = null;
 
 var view = {
     edges: {
         "default": {
             paintStyle: { lineWidth: 1, strokeStyle: '#89bcde' },
             overlays: [
                 ["Arrow", { fillStyle: "#89bcde", width: 10, length: 10, location:1 } ]
             ]
         },
         "bidirectional":{
             overlays: [
                 ["Arrow", { fillStyle: "#89bcde", width: 10, length: 10, location:0, direction:-1 } ]
             ]
         }
     },
     nodes:{
         "default":{
             events: {
                 tap:function(params) {
                     // on node click...
                     if (source == null) {
                         //... either set the current path source. here we also add a class
                         // so you can see its selected.
                         source = params;
                         jsPlumb.addClass(source.el, "jtk-animate-source");
                     }
                     else {
                         // ...or trace a path from the current source to the clicked node.
                         if (transport != null) {
                             transport.cancel();
                         }

                         // ...or trace a path from the current source to the clicked node.
                         transport = renderer.tracePath({
                             source:source.node,
                             target:params.node,
                             overlay:["Diamond", {
                                 width:15,
                                 length:15,
                                 fill: "#89bcde"
                             }],
                             options: {
                                 speed: 50
                             },
                             listener: stateChange
                         });
                         // cleanup the source for the next one.
                         jsPlumb.removeClass(source.el, "jtk-animate-source");
                         source = null;

                         if (transport.pathExists === false) {
                             alert("No path found!");
                         }
                     }
                 }
             }
         }
     }
 };
 ```

<a name="rendering"></a>
### Rendering

```javascript
var renderer = toolkit.load({type: "json", data: data}).render({
    container: canvasElement,
    view:view,
    layout: {
        type: "Spring",
        padding:[ 30, 30 ]
    },
    miniview: {
        container: miniviewElement
    },
    lassoFilter: ".controls, .controls *, .miniview, .miniview *",
    events: {
        canvasClick: function (e) {
            toolkit.clearSelection();
        },
        modeChanged: function (mode) {
            jsPlumb.removeClass(jsPlumb.getSelector("[mode]"), "selected-mode");
            jsPlumb.addClass(jsPlumb.getSelector("[mode='" + mode + "']"), "selected-mode");
        }
    },
    jsPlumb: {
        Anchor:"Continuous",
        Connector: [ "StateMachine", { cssClass: "connectorClass", hoverClass: "connectorHoverClass" } ],
        Endpoint: "Blank"
    }
});
```

The meaning of each parameter is as follows:

- **container**  The element into which to render. You can also provide an element ID here.
- **view** Mappings from Node/Edge/Port types to their visual representation/behaviour.
- **layout** We use a Spring layout in this demonstration.
- **jsPlumb** Defaults for the Surface's backing instance of jsPlumb.
- **miniview** Tthe element to turn into a Miniview. You can also provide an element ID here.
- **events** Provides event handler for events on the Surface. Here we listen for clicks and we clear the current selection, as well as events that tell us the Surface's mode has changed.
- **lassoFilter** A Selector identifying which elements should not respond to the lasso's start drag event.

[TOP](#top)

---

<a name="traversal"></a>
### Path Tracing

Path tracing is handled in two stages:

- when a user taps a node and no previous node has been selected, that node becomes the source
- when a user taps a node and there is already a source selected, the path is computed and traversed.

In versions of the Toolkit prior to 1.18.2, this method launched an animation and returned either true if the path existed (and therefore the animation was kicked off) or false otherwise. From 1.18.2 onwards, the return value of `tracePath` is a handler object that allows you to pause/restart/cancel the animation as you like.  For more information, see the [Path Tracing](https://docs.jsplumbtoolkit.com/toolkit/current/articles/path-tracing) page.

The tap event is captured and handled inside the view:

```javascript
tap:function(params) {
 // on node click...
 if (source == null) {
     //... either set the current path source. here we also add a class
     // so you can see its selected.
     source = params;
     jsPlumb.addClass(source.el, "jtk-animate-source");
 }
 else {
     if (transport != null) {
         transport.cancel();
     }
     
     transport = renderer.tracePath({
         source:source.node,
         target:params.node,
         overlay:["Diamond", {
             width:15,
             length:15,
             fill: "#89bcde"
         }],
         options: {
             speed: 50
         },
         listener: stateChange
     });
     
     // cleanup the source for the next one.
     jsPlumb.removeClass(source.el, "jtk-animate-source");
     source = null;

     if (transport.pathExists === false) {
         alert("No path found!");
     }
 }
}
```

The key piece of code here is the call to `tracePath` on the renderer:

```javascript
transport = renderer.tracePath({
     source:source.node,
     target:params.node,
     overlay:["Diamond", {
        width:15,
        length:15,
        fill: "#89bcde"
     }],
     options: {
        speed: 50
     },
     listener: stateChange
 });
 ```
 
 The meaning of each parameter is:
 
 - **source** Either a Node, Node ID, or a DOM element to use as the path source
 - **target** Either a Node, Node ID, or a DOM element to use as the path target
 - **overlay** An Overlay specification in the [standard format](https://docs.jsplumbtoolkit.com/toolkit/current/articles/overlays.html) used by the Community edition of jsPlumb.
 - **options** Options for the path traversal. For a discussion of these, see [here](https://docs.jsplumbtoolkit.com/toolkit/current/articles/path-tracing)
 - **listener** Nominates a function that the path traversal handler should call back with various lifecycle events.  See below.
 
 
<a name="transport"></a>
### Transport Controls

As mentioned above, `tracePath` returns a handler object that allows you to pause/restart/cancel the trace animation. You can also bind a listener to lifecycle events, either in the `tracePath` method call or on the handler object that it returns. In the code snippet shown above there is a `listener`, bound to the method `stateChange`. In this demonstration, `stateChange` looks like 
this:

```javascript
function stateChange(state) {
    document.querySelector(".controls").setAttribute("state", state);
    if (state === "stopped") {
        transport = null;
    }
}
```

`transport` is the demonstration page's local reference to the transport handler. Whenever the transport handler undergoes a state change, this method is called with the name of the new state, valid values for which are:

- "playing"
- "paused"
- "stopped"

We control the transport via the play/pause/cancel buttons in the top left of the canvas.  Our code to manage those is as follows:

```javascript
jsPlumb.on(".controls .transport", "tap", function() {
    var action = this.getAttribute("action");
    if (transport !== false) {
        transport[action]();
    }
});
```


