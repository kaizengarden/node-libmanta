---
title: node-libmanta
markdown2extras: tables, code-friendly
apisections:
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# node-libmanta

This repo serves to hold any/all common code that is shared between Manta
components.


# Mahi

This package contains a node client for communicating with
[Mahi](https://github.com/joyent/mahi), which is the authentication/identity
cache that Manta maintains in redis.

## createMahiClient(options)

Creates a client, and emits 'connect' when the client has established a
connection to redis.  Note that the `createMahiClient` call will handle DNS
resolution and retry/backoff logic, and maintains a local in-memory cache, so
that most common things are handled for you, and you can just directly use this
client.

    var libmanta = require('libmanta');

	var opts = {
        host: process.env.MAHI_HOST,
        log: bunyan.createLogger({
		    name: 'mahi',
			stream: process.stdout,
			level: 'info',
			serializers: bunyan.stdSerializers
        });
    };
    var mahi = libmanta.createMahiClient(opts);
    mahi.once('error', function (err) {
	    console.error(err.toString());
	    process.exit(1);
    });
    mahi.once('connect', function () {
	    ...
    });

The options that can be passed in at construct time:

| Name           | JS Type | Required | Description                                                             |
| -------------- | ------- | -------- | ----------------------------------------------------------------------- |
| cache          | Object  | No       | params to pass to [lru-cache](https://github.com/isaacs/node-lru-cache) |
| checkInterval  | Number  | No       | amount of milliseconds to wait between health checks                    |
| connectTimeout | Number  | No       | amount of milliseconds to wait for acquiring a socket to redis          |
| host           | String  | Yes      | IP/DNS name of redis host                                               |
| log            | Object  | Yes      | [bunyan](https://github.com/trentm/node-bunyan) logger                  |
| maxTimeout     | Number  | No       | Maximum amount of time between retries                                  |
| minTimeout     | Number  | No       | Minimum amount of time before first retry                               |
| port           | Number  | No       | Redis port                                                              |
| redis_options  | Object  | No       | params to pass to [node_redis](https://github.com/mranney/node_redis)   |
| retries        | Number  | No       | Number of times to attempt establishing a connection                    |


## mahi.userFromLogin(login, [opts], cb)

Loads a user record as a JS object given the login name.

    mahi.userFromLogin('poseidon', function (err, user) {
	    assert.ifError(err);
		console.log('%j', user);
    });

The returned `user` object will look like:

    {
        uuid: '56f237ac-8cb0-4468-896f-5aa00ff8ffc9',
        groups: [ 'operators' ],
        keys: {
            '24:ae:9d:...': 'PEM ENCODED PUBLIC KEY'
        },
        login: 'poseidon'
    }

Note that keys will not be the SSH public key, but rather the OpenSSL-friendly PEM.

The `opts` param is only if you wish to override the logger.

## mahi.userFromUUID(uuid, [opts], cb)

Loads a user record as a JS object given the UUID.

    var uuid = '56f237ac-8cb0-4468-896f-5aa00ff8ffc9';
    mahi.userFromUUID(uuid, function (err, user) {
	    assert.ifError(err);
		console.log('%j', user);
    });

The returned `user` object will look like:

    {
        uuid: '56f237ac-8cb0-4468-896f-5aa00ff8ffc9',
        groups: [ 'operators' ],
        keys: {
            '24:ae:9d:...': 'PEM ENCODED PUBLIC KEY'
        },
        login: 'poseidon'
    }

Note that keys will not be the SSH public key, but rather the OpenSSL-friendly PEM.

The `opts` param is only if you wish to override the logger.

# Queue

libmanta contains an alternate `queue` implementation, similar to the one in
[vasync](https://github.com/davepacheco/node-vasync), except that this queue is
intended to be short-lived, and ended.  There are several places in manta that
need a queue that lasts for some burst of requests (essentially using it as a
temporary throttle).  The usage of this is very simple; you push onto the queue,
and then `close` it.  When you get an `end` event, you're done.

    var fs = require('fs');

    var bytes = 0;
    var q = libmanta.createQueue({
	    limit: 10,  // allow at most 10 parallel runners
	    worker: function stat(p, cb) {
		    fs.stat(p, function (err, stats) {
			    if (err) {
				    cb(err); // q will emit error
					return;
                }

				if (stats.isFile())
				    bytes += stats.size;

                cb();
			});
        }
    });

    q.on('error', function (err) {
	    console.error('stat error: %s', err.stack);
    });

    q.once('end', function () {
	    console.log('/tmp uses %d bytes', bytes);
    });

    fs.readdir('/tmp', function (err, files) {
        assert.ifError(err);
		files.forEach(function (f) {
            q.push('/tmp/' + f);
        });
        q.close();
    });

Options that can be passed at construct time:

| Name   | JS Type  | Required | Description                                             |
| ------ | -------- | -------- | ------------------------------------------------------- |
| limit  | Number   | Yes      | Maximum number of concurrent worker functions to invoke |
| worker | Function | Yes      | worker function to call, of the form `f(arg, cb)`       |

Events available are `drain`, `end` and `error`, which do the usual things.
