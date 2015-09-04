# attaché

A [hapi.js](http://hapijs.com/) plugin that registers a [Consul](http://consul.io/) service.

[![Build Status](https://travis-ci.org/kanongil/attache.svg?branch=master)](https://travis-ci.org/kanongil/attache)

## Example

```js
var Hapi = require('hapi');

var server = new Hapi.Server();
server.connection({ labels: 'public' });
server.connection({ labels: 'private' });

server.register({
    register: require('attache'),
    options: {
        service: {
            name: 'myservice'
        }
    }
}, function (err) {

    if (err) {
        throw err;
    }

    server.select('public').route({
        method: 'GET',
        path: '/',
        handler: function (request, reply) {

            return reply('Hello World!');
        }
    });

    server.select('private').route({
        method: 'GET',
        path: '/',
        handler: function (request, reply) {

            return reply('Secret Hello!');
        }
    });

    server.route({
        method: 'GET',
        path: '/_health',
        handler: function (request, reply) {

            return reply('OK');
        }
    });

    server.start(function (err) {

        for (var idx = 0; idx < server.connections.length; idx++) {
            var connection = server.connections[idx];
            console.log('Server ' + connection.settings.labels + ' started at', connection.info.uri);
        }
    });
});
```

This will register a `"myservice"` service with Consul, with two instances using the `public` and `private` tags.

The service can be discovered using the standard Consul interfaces (eg. DNS or HTTP).

## Usage

Install using `npm install attache`, and register with Hapi using `server.register()`.

### Service registration & de-registration

Service registration is performed automatically as part of the `server.start()` processing.
Any errors are returned through the callback.

Service de-registration is performed as part of the `server.stop()` processing. Once the server has started,
it is important that `server.stop()` is called (and completed) before exiting the process.
Otherwise, the service won't be deregistered. Note that `server.stop()` must also be called when `server.start()`
returns with an error.

**If the service is not de-registered, it will linger in the Consul registry.** If configured with a health
check, it will eventually be marked as unhealthy.

### Plugin registration options

All configuration is optional:

 * `service` - Object containing service registration information:
   * `name` - String with service name, as registered with Consul. Default: `"hapi"`.
   * `check` - Object describing optional Consul health check:
     * `path` - String with path to http health check endpoint. Return any `2xx` code to indicate "passing",
                `429` for "warning", and any other response for "failure". If `false`, disables health checks.
                Default: `"/_health"`.
     * `interval` - Number with check interval in ms, or a String. Default: `"5s"`.
 * `consul` - Object with consul agent connection information:
   * `host` - String with agent address. Default: `"127.0.0.1"`.
   * `port` - Number with agent HTTP(s) port. Default: `8500`.
   * `secure` - Boolean indicating if HTTPS is required. Default: `false`
   * `ca` - Array of Strings or Buffers of trusted certificates in PEM format.

If a health check is configured, you can either point it to an existing path, or preferably create a custom route:

```js
server.route({
    method: 'GET',
    path: '/_health',
    handler: function (request, reply) {

        return reply('OK');
    }
});
```

Note that Consul does not use the returned value but it is recorded as part of the health check, useable
for debugging, etc.

### Connection plugin options

Each connection is registered as a unique service instance with tags matching the connections labels.
The instances have host unique ids, generated based on the service name, listening port, and process PID.
Alternatively, a custom id can be specified with the connection plugin option:

```js
server.connection({ plugins: { attache: { id: 'myservice-1' } } });
```

### Logging

All internal logging is tagged with `attache` and performed at the server level.
Any de-registration errors are only exposed through this mechanism.
