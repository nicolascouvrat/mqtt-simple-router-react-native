# MQTT simple router

This simple module allows you to centralize both subscribing to [MQTT](https://www.npmjs.com/package/mqtt) topics and reacting to subsequent messages on that topic in a router-type file.
The router uses [path-to-regexp](https://www.npmjs.com/package/path-to-regexp) in order to provide path recognition and named parameters

The creation of this module was motivated by the annoyance that is having both subscription code and listening code separated, and the nightmare that are chained if / elseif statements.
It aims at providing a simple solution to these problems.

## Installation

This is a [Node.js](https://nodejs.org/en/) module available for download on the [npm registry](https://www.npmjs.com/package/mqtt-simple-router)

To intall this module, simply run `npm install`
```
$ npm install mqtt-simple-router
```

### Dependencies

MQTT requires the [path-to-regexp](https://www.npmjs.com/package/path-to-regexp) module, available on npm.

### Usage - Quick start

The basic idea is simple and heavily inspired by Express.js's router : define routes for your router first, then wrap that router around the MQTT client once connected.

Routes can be added simply with `router.auto('your/mqtt/path', your_function(request, next))`. The router will automatically subscribe to all relevant topic(s) on `router.wrap(client)`.
**Note:** Due to the way MQTT functions, the `router.manual()` method that required a subscription later on to the topic in version 1.0 has been **removed** in 1.1. Indeed, using an auto-subscription on a "wide" topic (for instance `message/#`) would render all `router.manual()` calls useless for "narrower" topics, making code behave in an expected way. A manual switch of the routes themselves is planned for a later version, to obtain a behavior close to what was originally planned for `router.manual()`.

#### `request` parameter

The `request` passed to the callback function is an object containing `request.topic` (the original topic), `request.payload` (The MQTT payload, that can be modified by previous functions calling `next()`), `request.params` containing named parameters value (see the **named parameters** section), and `request.path` (the path corresponding to the latest layer matched by the topic).

#### `next` parameter

Much like Express.js's router, next() allows you to chain several layers for a single request. For example, all topics matching `/json/#` could first be json parsed by a first layer, then different layers `/json/topic1` and `/json/topic2` could apply more specific processing depending on the received topic. Note that:
**no argument** should be passed to the `next()` function except for errors (see the **error handling** section below), and that not calling `next()` will stop the matching process (i.e. the router will stop at the first match, even if there are others below) and lead to some layers being unaccessible.

```js
var mqtt = require('mqtt');
var mqttRouter = require('mqtt-simple-router');

var client = mqtt.connect('mqtt://localhost');

var router = new mqttRouter();

// the request object returned is an object containing
// .topic (the original topic)
// .payload
// .path and
// .params (containing named parameter values, or undefined)

router.auto('/all/hello', function(request) {
    // automatically subscribe on router.wrap()
    console.log('new message on ' + request.topic + ': ' + request.payload);
});

router.auto('/all/hello', function(request) {
    // this function will never executed, as the previous one does not call next() and will therefore 'block'
})

router.auto('/all/hello/message', function(request, next) {

    console.log('new message on ' + request.topic + ': ' + request.payload);
    request.payload = 'modified';
    next();

});

router.auto('/all/hello/message', function(request, next) {

    console.log('new message on ' + request.topic + ': ' + request.payload);
    // payload will be equal to 'modified'

});

// wrap the router around the client, subscribing to topics specified by auto
router.wrap(client);
```
#### Named parameters

Named parameters can be used, by preceding them with a semicolon `:`. Their value can then be accessed in `topic.params.parameter_name`.
The module also supports the MQTT jokers, treating '+' as a nameless parameter.

```js
router.auto('/:channel', function(request, next) {
    console.log('new message on channel ' + request.params.channel + ': ' + request.payload);
});

router.manual('/+/:subchannel/#', function(request, next) {
    console.log('new message containing subchannel ' + request.params.subchannel + ': ' + request.payload);
});
```

### Error handling

To counter the default behavior of the Node.js MQTT client -- if an error happens during message processing, the client stops listening altogether -- a basic error handling system has been implemented.
If an error happens at any point in any of the layers, the router starts looking for the nearest error-handling layer. If there is none, the error bubbles out of the layer stack and is intercepted by the default error handler.

#### Default error handling

Already built in. By default, simply logs the error and the layer in which it originally happened, then goes on to process the next request.
Can be modified at router instanciation using `router.defaultHandler()`:
```js
router.defaultHandler(function(error, request) {
    // all errors non treated by custom error handlers will end up here
    console.log("An error has happened!");
    // ...
});
```

#### Custom error handling

Can be implemented in the same way as regular layers using `router.auto()`. They will only be called if an error has happened at some point before in the layer stack, otherwise the router simply ignores them. Custom error handlers **must** have 3 arguments that correspond to the error, the request and the next function (in this order).

In a similar fashion to regular layers, error-handling layer can call on `next(error)` to pass the error to the next error handler (or to the default handler).

```js
router.auto(function(error, request, next) {
    if  (error.name === 'RangeError') {
        console.log('There was a range error :(')
    }
    else {
        next(error);
        // other errors are send to the next handler
    }
})
```

## Authors

* **Nicolas Couvrat** - [website](http://www.nicolascouvrat.com)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details

