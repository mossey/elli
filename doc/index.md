# elli - Erlang web server for HTTP APIs

[![Hex.pm][hex badge]][hex package]
[![Documentation][doc badge]][docs]
[![Erlang][erlang badge]][erlang downloads]
[![Travis CI][travis badge]][travis builds]
[![Coverage Status][coveralls badge]][coveralls link]
[![MIT License][license badge]](LICENSE)

[travis builds]: https://travis-ci.org/elli-lib/elli
[travis badge]: https://travis-ci.org/elli-lib/elli.svg
[hex badge]: https://img.shields.io/hexpm/v/elli.svg
[hex package]: https://hex.pm/packages/elli
[latest release]: https://github.com/elli-lib/elli/releases/latest
[erlang badge]: https://img.shields.io/badge/erlang-%E2%89%A518.0-red.svg
[erlang downloads]: http://www.erlang.org/downloads
[doc badge]: https://img.shields.io/badge/docs-edown-green.svg
[docs]: doc/README.md
[coveralls badge]: https://coveralls.io/repos/github/elli-lib/elli/badge.svg?branch=develop
[coveralls link]: https://coveralls.io/github/elli-lib/elli?branch=develop
[license badge]: https://img.shields.io/badge/license-MIT-blue.svg

Elli is a webserver you can run inside your Erlang application to
expose an HTTP API. Elli is a aimed exclusively at building
high-throughput, low-latency HTTP APIs. If robustness and performance
is more important than general purpose features, then `elli` might be
for you. If you find yourself digging into the implementation of a
webserver, `elli` might be for you. If you're building web services,
not web sites, then `elli` might be for you.

Elli is used in production at Wooga and Game Analytics. Elli requires
OTP 18.0 or newer.


## Installation

To use `elli` you will need a working installation of Erlang 18.0 (or later).

Add `elli` to your application by adding it as a dependency to your
[`rebar.config`](http://www.rebar3.org/docs/configuration):

```erlang
{deps, [elli]}.
```

Afterwards you can run:

```sh
$ rebar3 compile
```


## Usage
```sh
$ rebar3 shell
```

```erlang
%% starting elli
1> {ok, Pid} = elli:start_link([{callback, elli_example_callback}, {port, 3000}]).
```

## Examples

### Callback Module

The best source to learn how to write a callback module
is [src/elli_example_callback.erl](src/elli_example_callback.erl) and
its [generated documentation](doc/elli_example_callback.md). There are a bunch
of examples used in the tests as well as descriptions of all the events.

A minimal callback module could look like this:

```erlang
-module(elli_minimal_callback).
-export([handle/2, handle_event/3]).

-include_lib("elli/include/elli.hrl").
-behaviour(elli_handler).

handle(Req, _Args) ->
    %% Delegate to our handler function
    handle(Req#req.method, elli_request:path(Req), Req).

handle('GET',[<<"hello">>, <<"world">>], _Req) ->
    %% Reply with a normal response. `ok' can be used instead of `200'
    %% to signal success.
    {ok, [], <<"Hello World!">>};

handle(_, _, _Req) ->
    {404, [], <<"Not Found">>}.

%% @doc Handle request events, like request completed, exception
%% thrown, client timeout, etc. Must return `ok'.
handle_event(_Event, _Data, _Args) ->
    ok.
```


### Supervisor Childspec

To add `elli` to a supervisor you can use the following example and adapt it to
your needs.

```erlang
-module(fancyapi_sup).
-behaviour(supervisor).
-export([start_link/0]).
-export([init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    ElliOpts = [{callback, fancyapi_callback}, {port, 3000}],
    ElliSpec = {
        fancy_http,
        {elli, start_link, [ElliOpts]},
        permanent,
        5000,
        worker,
        [elli]},

    {ok, { {one_for_one, 5, 10}, [ElliSpec]} }.
```


## Features

Here's the features Elli _does_ have:

* [Rack][]-style request-response. Your handler function gets a
   complete request and returns a complete response. There's no
   messaging, no receiving data directly from the socket, no writing
   responses directly to the socket. It's a very simple and
   straightforward API. Have a look at [`elli_example_callback`](elli_example_callback.md)
for examples.

* Middlewares allow you to add useful features like compression,
encoding, stats, but only have it used when needed. No features you
don't use on the critical path.

* Short-circuiting of responses using exceptions, allows you to use
   "assertions" that return for example 403 permission
   denied. `is_allowed(Req) orelse throw({403, [], <<"Permission
   denied">>})`.

* Every client connection gets its own process, isolating the failure
of a request from another. For the duration of the connection, only
one process is involved, resulting in very robust and efficient
code.

* Binaries everywhere for strings.

* Instrumentation inside the core of the webserver, triggering user
   callbacks. For example when a request completes, the user callback
   gets the `request_complete` event which contains timings of all the
different parts of handling a request. There's also events for
clients unexpectedly closing a connection, crashes in the user
callback, etc.

* Keep alive, using one Erlang process per connection only active
when there is a request from the client. Number of connections is
only limited by RAM and CPU.

* Chunked transfer in responses for real-time push to clients

* Basic pipelining. HTTP verbs that does not have side-effects(`GET`
   and `HEAD`) can be pipelined, ie. a client supporting pipelining
can send multiple requests down the line and expect the responses
to appear in the same order as requests. Elli processes the
requests one at a time in order, future work could make it possible
to process them in parallel.

* SSL using built-in Erlang/OTP ssl, nice for low volume admin
interfaces, etc. For high volume, you should probably go with
nginx, stunnel or ELB if you're on AWS.

* Implement your own connection handling, for WebSockets, streaming
   uploads, etc. See [`elli_example_callback_handover`](elli_example_callback_handover.md).

## Extensions

* [elli_access_log](https://github.com/elli-lib/elli_access_log):
Access log 
* [elli_basicauth](https://github.com/elli-lib/elli_basicauth):
Basic auth 
* [elli_chatterbox](https://github.com/elli-lib/elli_chatterbox):
HTTP/2 support 
* [elli_cloudfront](https://github.com/elli-lib/elli_cloudfront):
CloudFront signed URLs 
* [elli_cookie](https://github.com/elli-lib/elli_cookie):
Cookies 
* [elli_date](https://github.com/elli-lib/elli_date):
"Date" header 
* [elli_fileserve](https://github.com/elli-lib/elli_fileserve):
Static content 
* [elli_prometheus](https://github.com/elli-lib/elli_prometheus):
Prometheus 
* [elli_stats](https://github.com/elli-lib/elli_stats):
Real-time statistics dashboard 
* [elli_websockets](https://github.com/elli-lib/elli_websocket):
WebSockets 
* [elli_xpblfe](https://github.com/elli-lib/elli_xpblfe):
X-Powered-By LFE

## About

From operating and debugging high-volume, low-latency apps we have
gained some valuable insight into what we want from a webserver. We
want simplicity, robustness, performance, ease of debugging,
visibility into strange client behaviour, really good instrumentation
and good tests. We are willing to sacrifice almost everything, even
basic features to achieve this.

With this in mind we looked at the big names in the Erlang
community: [Yaws][], [Mochiweb][], [Misultin][] and [Cowboy][]. We
found [Mochiweb][] to be the best match. However, we also wanted to
see if we could take the architecture of [Mochiweb][] and improve on
it. `elli` takes the acceptor-turns-into-request-handler idea found
in [Mochiweb][], the binaries-only idea from [Cowboy][] and the
request-response idea from [WSGI][]/[Rack][] (with chunked transfer
being an exception).

On top of this we built a handler that allows us to write HTTP
middleware modules to add practical features, like compression of
responses, HTTP access log with timings, a real-time statistics
dashboard and chaining multiple request handlers.

## Aren't there enough webservers in the Erlang community already?

There are a few very mature and robust projects with steady
development, one recently ceased development and one new kid on the
block with lots of interest. As `elli` is not a general purpose
webserver, but more of a specialized tool, we believe it has a very
different target audience and would not attract effort or users away
from the big names.

## Why another webserver? Isn't this just the NIH syndrome?

[Yaws][], [Mochiweb][], [Misultin][], and [Cowboy][] are great
projects, hardened over time and full of very useful features for web
development. If you value developer productivity, [Yaws][] is an
excellent choice. If you want a fast and lightweight
server, [Mochiweb][] and [Cowboy][] are excellent choices.

Having used and studied all of these projects, we believed that if we
merged some of the existing ideas and added some ideas from other
communities, we could create a core that was better for our use cases.

It started out as an experiment to see if it is at all possible to
significantly improve and it turns out that for our particular use
cases, there is enough improvement to warrant a new project.

## What makes Elli different?

Elli has a very simple architecture. It avoids using more processes
and messages than absolutely necessary. It uses binaries for
strings. The request-response programming model allows middlewares to
do much heavy lifting, so the core can stay very simple. It has been
instrumented so as a user you can understand where time is spent. When
things go wrong, like the client closed the connection before you
could send a response, you are notified about these things so you can
better understand your client behaviour.

## Performance

"Hello World!" micro-benchmarks are really useful when measuring the
performance of the webserver itself, but the numbers usually do more
harm than good when released. I encourage you to run your own
benchmarks, on your own hardware. Mark Nottingham has some
[very good pointers](http://www.mnot.net/blog/2011/05/18/http_benchmark_rules)
about benchmarking HTTP servers.

[Yaws]: https://github.com/klacke/yaws
[Mochiweb]: https://github.com/mochi/mochiweb
[Misultin]: https://github.com/ostinelli/misultin
[Cowboy]: https://github.com/ninenines/cowboy
[WSGI]: https://www.python.org/dev/peps/pep-3333/
[Rack]: https://github.com/rack/rack


## Modules ##


<table width="100%" border="0" summary="list of modules">
<tr><td><a href="elli.md" class="module">elli</a></td></tr>
<tr><td><a href="elli_example_callback.md" class="module">elli_example_callback</a></td></tr>
<tr><td><a href="elli_example_callback_handover.md" class="module">elli_example_callback_handover</a></td></tr>
<tr><td><a href="elli_handler.md" class="module">elli_handler</a></td></tr>
<tr><td><a href="elli_http.md" class="module">elli_http</a></td></tr>
<tr><td><a href="elli_middleware.md" class="module">elli_middleware</a></td></tr>
<tr><td><a href="elli_middleware_compress.md" class="module">elli_middleware_compress</a></td></tr>
<tr><td><a href="elli_request.md" class="module">elli_request</a></td></tr>
<tr><td><a href="elli_sendfile.md" class="module">elli_sendfile</a></td></tr>
<tr><td><a href="elli_tcp.md" class="module">elli_tcp</a></td></tr>
<tr><td><a href="elli_test.md" class="module">elli_test</a></td></tr>
<tr><td><a href="elli_util.md" class="module">elli_util</a></td></tr></table>


## License

Elli is licensed under [The MIT License](LICENSE).

Copyright (c) 2012-2016 Knut Nesheim, 2016-2018 elli-lib team
