# @routes/graceful-shutdown [![](https://circleci.com/gh/omrilotan/express-graceful-shutdown.svg?style=svg)](https://circleci.com/gh/omrilotan/express-graceful-shutdown) <a href="https://www.npmjs.com/package/@routes/graceful-shutdown"><img src="https://img.shields.io/npm/v/@routes/graceful-shutdown.svg"></a> [![](https://img.shields.io/badge/source--000000.svg?logo=github&style=social)](https://github.com/omrilotan/express-graceful-shutdown)

## 💀 Shut down server gracefully

[net.Server](https://nodejs.org/api/net.html#net_class_net_server) or [Express](https://expressjs.com/en/api.html#app.listen), whatever you're using should be fine

```js
const graceful = require('@routes/graceful-shutdown');

const server = app.listen(1337); // express will return the server instance here
graceful(server);
```

### Arguments
- First argument is an net.Server instance (including Express server)
- Second argument is options:

| option | type | meaning | default
| - | - | - | -
| `timeout` | Number | Time (in milliseconds) to wait before forcefully shutting down | `10000` (10 sec)
| `logger` | Object | Object with `info` and `error` functions (supports async loggers) | `console`
| `events` | Array | Process events to handle | `['SIGTERM', 'SIGINT']`

Example of using options
```js
graceful(server, {
	timeout: 3e4,
	logger: winston.createLogger({level: 'error'}),
});
```

## What happens on process termination?
1. Process termination is interrupted
2. Open connections are instructed to end (FIN packet) and receive a new timeout to allow them to close in time
3. The server is firing a close function
4. After correct closing the process exists with exit code 0
	- _End correct behaviour_
5. After `timeout` passes - an error log is printed with the amount of connection that will be forcefully terminated
6. Process exists with an exit code 1

## Procedure as a module
In case you wish to apply termination interruption yourself, use `procedure` interface:
```js
const { procedure } = require('@routes/graceful-shutdown');

const shutdown = procedure(server, {
	timeout: 1e4,
	logger: {info: msg => null, error: msg => null},
	onsuccess: () => null,
	onfail: () => null,
});

// Apply procedure to termination
process.stdin.resume();
process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);
```

| option | type | meaning | default
| - | - | - | -
| `timeout` | Number | Time (in milliseconds) to wait before forcefully shutting down | `10000` (10 sec)
| `logger` | Object | Object with `info`, and `error` functions (supports async loggers) | `console`
| `onsuccess` | Function | Callback when shutdown finished correctly | `process.exit(0)`
| `onsuccess` | Function | Callback when shutdown finished incorrectly | `process.exit(1)`

## `server.shuttingDown`
After shutdown has initiated, the attribute `shuttingDown` is attached to server with value of `true`.

User can query this value on service to know not to send any more requests to the service
```js
app.get(
	'/health',
	(request, response) => response.status(server.shuttingDown ? 503 : 200).end()
);
```
