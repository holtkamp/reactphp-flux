# clue/reactphp-flux [![Build Status](https://travis-ci.org/clue/reactphp-flux.svg?branch=master)](https://travis-ci.org/clue/reactphp-flux)

Flux, the lightweight stream processor to concurrently do many (but not too many) things at once,
built on top of [ReactPHP](https://reactphp.org/).

Let's say you have a large list of users or products that you want to process
by individually sending a (RESTful) HTTP API request to some third party API
for each record. Estimating each call to take around `0.3s` means that having
`10000` users processed sequentially, you would have to wait around 50 minutes
for all jobs to complete. This works perfectly fine for a small number of
operations, but keeping thousands of jobs in memory at once may easly take up
all resources on your side.
Instead, you can use this library to stream your arbitrarily large input list
as individual records to a non-blocking (async) transformation handler. It uses
[ReactPHP](https://reactphp.org) to enable you to concurrently process multiple
records at once. You can control the concurrency limit, so that by allowing
it to process 10 operations at the same time, you can thus process this large
input list around 10 times faster and at the same time you're no longer limited
how many records this list may contain (think processing millions of records).
This library provides a simple API that is easy to use in order to manage any
kind of async operation without having to mess with most of the low-level details.
You can use this to throttle multiple HTTP requests, database queries or pretty
much any API that already uses Promises.

* **Async execution of operations** -
  Choose how many async operations should be processed at once (concurrently).
  Process their results as soon as responses come in.
  The Promise-based design provides a *sane* interface to working with out of bound results.
* **Standard interfaces** -
  Allows easy integration with existing higher-level components by implementing
  ReactPHP's standard [promises](#promises) and [streaming interfaces](#streaming).
* **Lightweight, SOLID design** -
  Provides a thin abstraction that is [*just good enough*](https://en.wikipedia.org/wiki/Principle_of_good_enough)
  and does not get in your way.
  Builds on top of well-tested components and well-established concepts instead of reinventing the wheel.
* **Good test coverage** -
  Comes with an [automated tests suite](#tests) and is regularly tested in the *real world*

**Table of contents**

* [Quickstart example](#quickstart-example)
* [Usage](#usage)
  * [Transformer](#transformer)
    * [Promises](#promises)
    * [Timeout](#timeout)
    * [Streaming](#streaming)
* [Install](#install)
* [Tests](#tests)
* [License](#license)
* [More](#more)

> Note: This project is in an early alpha stage! Feel free to report any issues you encounter.

## Quickstart example

Once [installed](#install), you can use the following code to process an example
user lists by sending a (RESTful) HTTP API request for each user record:

```php
$loop = React\EventLoop\Factory::create();
$browser = new Clue\React\Buzz\Browser($loop);

$concurrency = isset($argv[1]) ? $argv[1] : 3;

// each job should use the browser to GET a certain URL
// limit number of concurrent jobs here
$transformer = new Transformer($concurrency, function ($user) use ($browser) {
    // skip users that do not have an IP address listed
    if (!isset($user['ip'])) {
        return React\Promise\resolve($user);
    }

    // look up country for this IP
    return $browser->get("https://ipapi.co/$user[ip]/country_name/")->then(
        function (ResponseInterface $response) use ($user) {
            // response successfully received
            // add country to user array and return updated user
            $user['country'] = (string)$response->getBody();

            return $user;
        }
    );
});

// load a huge number of users to process from NDJSON file
$input = new Clue\React\NDJson\Decoder(
    new React\Stream\ReadableResourceStream(
        fopen(__DIR__ . '/users.ndjson', 'r'),
        $loop
    ),
    true
);

// process all users by piping through transformer
$input->pipe($transformer);

// log transformed output results
$transformer->on('data', function ($user) {
    echo $user['name'] . ' is from ' . $user['country'] . PHP_EOL;
});
$transformer->on('end', function () {
    echo '[DONE]' . PHP_EOL;
});
$transformer->on('error', 'printf');

$loop->run();
```

See also the [examples](examples).

By changing the `$concurrency` parameter, you can see how processing this list
without concurrency takes near `4s`, while using a concurrency setting of `5`
takes near just `1s` (YMMV obviously).

## Usage

### Transformer

The `Transformer` passes all input data through its transformation handler
and forwards the resulting output data.

It uses ReactPHP's standard [streaming interfaces](#streaming) which allow
to process huge inputs without having to store everything in memory at once
and instead allows you to efficiently process its input in small chunks.
Any data you write to this stream will be passed through its transformation
handler which is responsible for processing and transforming this data and
also takes care of mangaging streaming throughput and back-pressure.

The transformation handler can be any non-blocking (async) callable that uses
[promises](#promises) to signal its eventual results. This callable receives
a single data argument as passed to the writable side and must return a
promise. A succesful fulfillment value will be forwarded to the readable end
of the stream, while an unsuccessful rejection value will emit an `error`
event and then `close()` the stream.

The `new Transformer(int $concurrency, callable $handler)` call
can be used to create a new transformer instance.
You can create any number of transformation streams, for example when you
want to apply different transformations to different kinds of streams.

The `$concurrency` parameter sets a new soft limit for the maximum number
of jobs to handle concurrently. Finding a good concurrency limit depends
on your particular use case. It's common to limit concurrency to a rather
small value, as doing more than a dozen of things at once may easily
overwhelm the receiving side. Using a `1` value will ensure that all jobs
are processed one after another, effectively creating a "waterfall" of
jobs. Using a value less than 1 will throw an `InvalidArgumentException`.

```php
// handle up to 10 jobs concurrently
$transformer = new Transformer(10, $handler);
```

```php
// handle each job after another without concurrency (waterfall)
$transformer = new Transformer(1, $handler);
```

The `$handler` parameter must be a valid callable that accepts your job
parameter (the data from its writable side), invokes the appropriate
operation and returns a Promise as a placeholder for its future result
(which will be made available on its readable side).

```php
// using a Closure as handler is usually recommended
$transformer = new Transformer(10, function ($url) use ($browser) {
    return $browser->get($url);
});
```

```php
// accepts any callable, so PHP's array notation is also supported
$transformer = new Transformer(10, array($browser, 'get'));
```

*Continue with reading more about [promises](#promises).*

#### Promises

This library works under the assumption that you want to concurrently handle
async operations that use a [Promise](https://github.com/reactphp/promise)-based API.
You can use this to concurrently run multiple HTTP requests, database queries
or pretty much any API that already uses Promises.

The demonstration purposes, the examples in this documentation use the async
HTTP client [clue/reactphp-buzz](https://github.com/clue/reactphp-buzz).
Its API can be used like this:

```php
$loop = React\EventLoop\Factory::create();
$browser = new Clue\React\Buzz\Browser($loop);

$promise = $browser->get($url);
```

If you wrap this in a `Transformer` instance as given above, this code will look
like this:

```php
$loop = React\EventLoop\Factory::create();
$browser = new Clue\React\Buzz\Browser($loop);

$transformer = new Transformer(10, function ($url) use ($browser) {
    return $browser->get($url);
});

$transformer->write($url);
```

The `$transformer` instance is a `WritableStreaminterface`, so that writing to it
with `write($data)` will actually be forwarded as `$browser->get($data)` as
given in the `$handler` argument (more about this in the following section about
[streaming](#streaming)).

Each operation is expected to be async (non-blocking), so you may actually
invoke multiple handlers concurrently (send multiple requests in parallel).
The `$handler` is responsible for responding to each request with a resolution
value, the order is not guaranteed.
These operations use a [Promise](https://github.com/reactphp/promise)-based
interface that makes it easy to react to when an operation is completed (i.e.
either successfully fulfilled or rejected with an error):

```php
$transformer = new Transformer(10, function ($url) use ($browser) {
    $promise = $browser->get($url);

    return $promise->then(
        function ($response) {
            var_dump('Result received', $result);

            return json_decode($response->getBody());
        },
        function (Exception $error) {
            var_dump('There was an error', $error->getMessage());

            throw $error;
        }
    );
);
```

Each operation may take some time to complete, but due to its async nature you
can actually start any number of (queued) operations. Once the concurrency limit
is reached, this invocation will simply be queued and this stream will signal
to the writing side that it should pause writing, thus effectively throttling
the writable side (back-pressure). It will automatically start the next
operation once another operation is completed and signal to the writable side
that is may resume writing. This means that this is handled entirely
transparently and you do not need to worry about this concurrency limit
yourself.

This example expects URI strings as input, sends a simple HTTP GET request
and returns the JSON-decoded HTTP response body. You can transform your
fulfillment value to anything that should be made available on the readable
end of your stream. Similar logic may be used to filter your input stream,
such as skipping certain input values or rejecting it by returning a rejected
promise. Accordingly, returning a rejected promise (the equivalent of
throwing an `Exception`) will result in an `error` event that tries to
`cancel()` all pending operations and then `close()` the stream.

#### Timeout

By default, this library does not limit how long a single operation can take,
so that the transformation handler may stay pending for a long time.
Many use cases involve some kind of "timeout" logic so that an operation is
cancelled after a certain threshold is reached.

You can simply use [react/promise-timer](https://github.com/reactphp/promise-timer)
which helps taking care of this through a simple API.

The resulting code with timeouts applied look something like this:

```php
use React\Promise\Timer;

$transformer = new Transformer(10, function ($uri) use ($browser, $loop) {
    return Timer\timeout($browser->get($uri), 2.0, $loop);
});

$transformer->write($uri);
```

The resulting stream can be consumed as usual and the above code will ensure
that execution of this operation can not take longer than the given timeout
(i.e. after it is actually started).

Please refer to [react/promise-timer](https://github.com/reactphp/promise-timer)
for more details.

#### Streaming

The `Transformer` implements the [`DuplexStreamInterface`](https://github.com/reactphp/stream#duplexstreaminterface)
and as such allows you to write to its writable input side and to consume
from its readable output side. Any data you write to this stream will be
passed through its transformation handler which is responsible for processing
and transforming this data (see above for more details).

The `Transformer` takes care of passing data you pass on its writable side to
the transformation handler argument and forwarding resuling data to it
readable end.
Each operation may take some time to complete, but due to its async nature you
can actually start any number of (queued) operations. Once the concurrency limit
is reached, this invocation will simply be queued and this stream will signal
to the writing side that it should pause writing, thus effectively throttling
the writable side (back-pressure). It will automatically start the next
operation once another operation is completed and signal to the writable side
that is may resume writing. This means that this is handled entirely
transparently and you do not need to worry about this concurrency limit
yourself.

The following examples use an async (non-blocking) transformation handler as
given above:

```php
$loop = React\EventLoop\Factory::create();
$browser = new Clue\React\Buzz\Browser($loop);

$transformer = new Transformer(10, function ($url) use ($browser) {
    return $browser->get($url);
});
```

The `write(mixed $data): bool` method can be used to
transform data through the transformation handler like this:

```php
$transformer->on('data', function (ResponseInterface $response) {
    var_dump($response);
});

$transformer->write('http://example.com/');
```

This handler receives a single data argument as passed to the writable side
and must return a promise. A succesful fulfillment value will be forwarded to
the readable end of the stream, while an unsuccessful rejection value will
emit an `error` event, try to `cancel()` all pending operations and then
`close()` the stream.

Note that this class makes no assumptions about any data types. Whatever is
written to it, will be processed by the transformation handler. Whatever the
transformation handler yields will be forwarded to its readable end.

The `end(mixed $data = null): void` method can be used to
soft-close the stream once all transformation handlers are completed.
It will close the writable side, wait for all outstanding transformation
handlers to complete and then emit an `end` event and then `close()` the stream.
You may optionally pass a (non-null) `$data` argument which will be processed
just like a `write($data)` call immediately followed by an `end()` call.

```php
$transformer->on('data', function (ResponseInterface $response) {
    var_dump($response);
});
$transformer->on('end', function () {
    echo '[DONE]' . PHP_EOL;
});

$transformer->end('http://example.com/');
```

The `close(): void` method can be used to
forcefully close the stream. It will try to `cancel()` all pending transformation
handlers and then immediately close the stream and emit a `close` event.

```php
$transformer->on('data', $this->expectCallableNever());
$transformer->on('close', function () {
    echo '[CLOSED]' . PHP_EOL;
});

$transformer->write('http://example.com/');
$transformer->close();
```

The `pipe(WritableStreamInterface $dest): WritableStreamInterface` method can be used to
forward an input stream into the transformer and/or to forward the resulting
output stream to another stream.

```php
$source->pipe($transformer)->pipe($dest);
```

This piping context is particularly powerful because it will automatically
throttle the incoming source stream and wait for the transformation handler
to complete before resuming work (back-pressure). Any additional data events
will be queued in-memory and resumed as appropriate. As such, it allows you
to limit how many operations are processed at once.

Because streams are one of the core abstractions of ReactPHP, a large number
of stream implementations are available for many different use cases. For
example, this allows you to use the following pseudo code to send an HTTP
request for each JSON object in a compressed NDJSON file:

```php
$transformer = new Transformer(10, function ($data) use ($http) {
    return $http->post('https://example.com/?id=' . $data['id'])->then(
        function ($response) use ($data) {
            return array('done' => $data['id']);
        }
    );
});

$source->pipe($gunzip)->pipe($ndjson)->pipe($transformer)->pipe($dest);
```

Keep in mind that the transformation handler may return a rejected promise.
In this case, the stream will emit an `error` event and then `close()` the
stream. If you do not want the stream to end in this case, you explicitly
have to handle any rejected promises and return some placeholder value
instead, for example like this:

```php
$uploader = new Transformer(10, function ($data) use ($http) {
    return $http->post('https://example.com/?id=' . $data['id'])->then(
        function ($response) use ($data) {
            return array('done' => $data['id']);
        },
        function ($error) use ($data) {
            // HTTP request failed => return dummy indicator
            return array(
                'failed' => $data['id'],
                'reason' => $error->getMessage()
            );
        }
    );
});
```

## Install

The recommended way to install this library is [through Composer](https://getcomposer.org).
[New to Composer?](https://getcomposer.org/doc/00-intro.md)

This will install the latest supported version:

```bash
$ composer require clue/reactphp-flux:dev-master
```

This project aims to run on any platform and thus does not require any PHP
extensions and supports running on legacy PHP 5.3 through current PHP 7+ and
HHVM.
It's *highly recommended to use PHP 7+* for this project.

## Tests

To run the test suite, you first need to clone this repo and then install all
dependencies [through Composer](https://getcomposer.org):

```bash
$ composer install
```

To run the test suite, go to the project root and run:

```bash
$ php vendor/bin/phpunit
```

## License

This project is released under the permissive [MIT license](LICENSE).

> Did you know that I offer custom development services and issuing invoices for
  sponsorships of releases and for contributions? Contact me (@clue) for details.

## More

* If you want to learn more about processing streams of data, refer to the documentation of
  the underlying [react/stream](https://github.com/reactphp/stream) component.

* If you only want to process a few dozens or hundreds of operations,
  you may want to use [clue/reactphp-mq](https://github.com/clue/reactphp-mq)
  instead and keep all operations in memory without using a streaming approach.

* If you want to process structured NDJSON files (`.ndjson` file extension),
  you may want to use [clue/reactphp-ndjson](https://github.com/clue/reactphp-ndjson)
  on the input stream before passing the decoded stream to the transformator.

* If you want to process compressed GZIP files (`.gz` file extension),
  you may want to use [clue/reactphp-zlib](https://github.com/clue/reactphp-zlib)
  on the compressed input stream before passing the decompressed stream to the
  decoder (such as NDJSON).
