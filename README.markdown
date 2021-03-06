Name
====

**ngx_headers_more** - Set and clear input and output headers...more than "add"!

*This module is not distributed with the Nginx source.* See [the installation instructions](http://wiki.nginx.org/HttpHeadersMoreModule#Installation).

Version
=======

This document describes headers-more-nginx-module [v0.16](http://github.com/agentzh/headers-more-nginx-module/tags) released on 16 January 2012.

Synopsis
========


    # set the Server output header
    more_set_headers 'Server: my-server';

    # set and clear output headers
    location /bar {
        more_set_headers 'X-MyHeader: blah' 'X-MyHeader2: foo';
        more_set_headers -t 'text/plain text/css' 'Content-Type: text/foo';
        more_set_headers -s '400 404 500 503' -s 413 'Foo: Bar';
        more_clear_headers 'Transfer-Encoding' 'Content-Type';
        
        # your proxy_pass/memcached_pass/or any other config goes here...
    }

    # set output headers
    location /type {
        more_set_headers 'Content-Type: text/plain';
        # ...
    }

    # set input headers
    location /foo {
        set $my_host 'my dog';
        more_set_input_headers 'Host: $my_host';
        more_set_input_headers -t 'text/plain' 'X-Foo: bah';
       
        # now $host and $http_host have their new values...
        # ...
    }

    # replace input header X-Foo *only* if it already exists
    more_set_input_headers -r 'X-Foo: howdy';


Description
===========

This module allows you to add, set, or clear any output
or input header that you specify.

This is an enhanced version of the standard
[headers](http://wiki.nginx.org/HttpHeadersModule) module because it provides more utilities like
resetting or clearing "builtin headers" like `Content-Type`,
`Content-Length`, and `Server`.

It also allows you to specify an optional HTTP status code
criteria using the `-s` option and an optional content
type criteria using the `-t` option while modifying the
output headers with the [more_set_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_headers) and
[more_clear_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_clear_headers) directives. For example,


    more_set_headers -s 404 -t 'text/html' 'X-Foo: Bar';


Input headers can be modified as well. For example


    location /foo {
        more_set_input_headers 'Host: foo' 'User-Agent: faked';
        # now $host, $http_host, $user_agent, and
        #   $http_user_agent all have their new values.
    }


The option `-t` is also available in the
[more_set_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_input_headers) and
[more_clear_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_clear_input_headers) directives (for request header filtering) while the `-s` option
is not allowed.

Unlike the standard [headers](http://wiki.nginx.org/HttpHeadersModule) module, this module's directives will by
default apply to all the status codes, including `4xx` and `5xx`.

Directives
==========

more_set_headers
----------------
**syntax:** *more_set_headers [-t &lt;content-type list&gt;]... [-s &lt;status-code list&gt;]... &lt;new-header&gt;...*

**default:** *no*

**context:** *http, server, location, location if*

**phase:** *output-header-filter*

Adds or replaces the specified output headers when the response status code matches the codes specified by the `-s` option *AND* the response content type matches the types specified by the `-t` option.

If either `-s` or `-t` is not specified or has an empty list value, then no match is required. Therefore, the following directive set the `Server` output header to the custom value for *any* status code and *any* content type:


      more_set_headers    "Server: my_server";


A single directive can set/add multiple output headers. For example


      more_set_headers 'Foo: bar' 'Baz: bah';


Multiple occurrences of the options are allowed in a single directive. Their values will be merged together. For instance


      more_set_headers -s 404 -s '500 503' 'Foo: bar';


is equivalent to


      more_set_headers -s '404 500 503' 'Foo: bar';


The new header should be the one of the forms:

1. `Name: Value`
1. `Name: `
1. `Name`

The last two effectively clear the value of the header `Name`.

Nginx variables are allowed in header values. For example:


       set $my_var "dog";
       more_set_headers "Server: $my_var";


But variables won't work in header keys due to performance considerations.

Multiple set/clear header directives are allowed in a single location, and they're executed sequentially.

Directives inherited from an upper level scope (say, http block or server blocks) are executed before the directives in the location block.

Note that although `more_set_headers` is allowed in *location* if blocks, it is *not* allowed in the *server* if blocks, as in


      ?  # This is NOT allowed!
      ?  server {
      ?      if ($args ~ 'download') {
      ?          more_set_headers 'Foo: Bar';
      ?      }
      ?      ...
      ?  }


Behind the scene, use of this directive and its friend [more_clear_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_clear_headers) will (lazily) register an ouput header filter that modifies `r->headers_out` the way you specify.

more_clear_headers
------------------
**syntax:** *more_clear_headers [-t &lt;content-type list&gt;]... [-s &lt;status-code list&gt;]... &lt;new-header&gt;...*

**default:** *no*

**context:** *http, server, location, location if*

**phase:** *output-header-filter*

Clears the specified output headers.

In fact,


       more_clear_headers -s 404 -t 'text/plain' Foo Baz;


is exactly equivalent to


       more_set_headers -s 404 -t 'text/plain' "Foo: " "Baz: ";


or


       more_set_headers -s 404 -t 'text/plain' Foo Baz


See [more_set_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_headers) for more details.

Wildcard `*` can also be used to specify a header name pattern. For example, the following directive effectively clears *any* output headers starting by "`X-Hidden-`":


    more_clear_headers 'X-Hidden-*';


The `*` wildcard support was first introduced in [v0.09](http://wiki.nginx.org/HttpHeadersMoreModule#v0.09).

more_set_input_headers
----------------------
**syntax:** *more_set_input_headers [-r] [-t &lt;content-type list&gt;]... &lt;new-header&gt;...*

**default:** *no*

**context:** *http, server, location, location if*

**phase:** *rewrite tail*

Very much like [more_set_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_headers) except that it operates on input headers (or request headers) and it only supports the `-t` option.

Behind the scene, use of this directive and its friend [more_clear_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_clear_input_headers) will (lazily) register a `rewrite phase` handler that modifies `r->headers_in` the way you specify. Note that it always run at the *end* of the `rewrite` so that it runs *after* the standard [rewrite module](http://wiki.nginx.org/HttpRewriteModule) and works in subrequests as well.

If the `-r` option is specified, then the headers will be replaced to the new values *only if* they already exist.

more_clear_input_headers
------------------------
**syntax:** *more_clear_input_headers [-t &lt;content-type list&gt;]... &lt;new-header&gt;...*

**default:** *no*

**context:** *http, server, location, location if*

**phase:** *rewrite tail*

Clears the specified input headers.

In fact,


       more_clear_input_headers -s 404 -t 'text/plain' Foo Baz;


is exactly equivalent to


       more_set_input_headers -s 404 -t 'text/plain' "Foo: " "Baz: ";


or


       more_set_input_headers -s 404 -t 'text/plain' Foo Baz


See [more_set_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_input_headers) for more details.

Limitations
===========

* Unlike the standard [headers](http://wiki.nginx.org/HttpHeadersModule) module, this module does not automatically take care of the constraint among the `Expires`, `Cache-Control`, and `Last-Modified` headers. You have to get them right yourself or use the [headers](http://wiki.nginx.org/HttpHeadersModule) module together with this module.

Installation
============

Grab the nginx source code from [nginx.org](http://nginx.org/), for example,
the version 1.0.11 (see [nginx compatibility](http://wiki.nginx.org/HttpHeadersMoreModule#Compatibility)), and then build the source with this module:


    wget 'http://nginx.org/download/nginx-1.0.11.tar.gz'
    tar -xzvf nginx-1.0.11.tar.gz
    cd nginx-1.0.11/
    
    # Here we assume you would install you nginx under /opt/nginx/.
    ./configure --prefix=/opt/nginx \
        --add-module=/path/to/headers-more-nginx-module
     
    make
    make install


Download the latest version of the release tarball of this module from [headers-more-nginx-module file list](http://github.com/agentzh/headers-more-nginx-module/tags).

Also, this module is included and enabled by default in the [ngx_openresty bundle](http://openresty.org).

Compatibility
=============

The following versions of Nginx should work with this module:

* **1.1.x**                       (last tested: 1.1.5)
* **1.0.x**                       (last tested: 1.0.11)
* **0.9.x**                       (last tested: 0.9.4)
* **0.8.x**                       (last tested: 0.8.54)
* **0.7.x >= 0.7.44**             (last tested: 0.7.68)

Earlier versions of Nginx like 0.6.x and 0.5.x will *not* work.

If you find that any particular version of Nginx above 0.7.44 does not work with this module, please consider [reporting a bug](http://wiki.nginx.org/HttpHeadersMoreModule#Report_Bugs).

Report Bugs
===========

Although a lot of effort has been put into testing and code tuning, there must be some serious bugs lurking somewhere in this module. So whenever you are bitten by any quirks, please don't hesitate to

1. send a bug report or even patches to <agentzh@gmail.com>,
1. or create a ticket on the [issue tracking interface](http://github.com/agentzh/headers-more-nginx-module/issues) provided by GitHub.

Source Repository
=================

Available on github at [agentzh/headers-more-nginx-module](http://github.com/agentzh/headers-more-nginx-module).

ChangeLog
=========

v0.16
-----

January 16, 2012

* bugfix: the on-demand handler/filter registration mechanism did not work fully for config reload via the `HUP` signal.
* bugfix: when setting a multi-value response header to a single value, the single value might be repeated on each old value.
* feature: added some debugging outputs that can be enabled by the `--with-debug` option when building nginx or [ngx_openresty](http://openresty.org).
* bugfix: we should set header hash using `ngx_hash_key_lc`, not simply to `1`.
* bugfix: Setting `Cache-Control` response headers might not work with other nginx output filter modules because we did not properly prepare the `r->cache_control` array at the same time.
* bugfix: [more_set_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_input_headers) and [more_clear_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_clear_input_headers) did not handle the `Accept-Encoding` request headers properly. thanks 天街夜色.
* bugfix: the [more_set_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_input_headers) directive might cause invalid memory reads because Nginx request header values must be null terminated. thanks Maxim Dounin.
* bugfix: removing builtin headers in huge request headers with 20+ entries could result in data loss. thanks Chris Dumoulin for the patch in [github issue #6](https://github.com/agentzh/headers-more-nginx-module/issues/6).

v0.15
-----

July 06, 2011

* now more_set_headers supports overriding charset in Content-Type. thanks ML.
* fixed an issue in more_clear_headers: we should remove all the instances of the headers specified, not only the first occurrence. thanks Li Yang.
* back-ported a bugfix from ngx_lua: in output header set, we should always set the header->hash to 1. thanks moodydeath for reporting it.
* fixed a bug when clearing the Accept-Ranges header. thanks Bo Blangstrup.

v0.14
-----

January 25, 2011

* now we postpone the rewrite phase handler only once rather than on every main request previously. this will save some CPU cycles on every request if [more_set_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_input_headers) or [more_clear_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_clear_input_headers) are used.
* fixed two spots where we did not check against null pointers when out of memory.
* now we use the 2-clause bsd license instead.
* various coding style fixes.

v0.13
-----

July 07, 2010

* fixed a bug in rewrite phase postponing algorithm which may cause [ngx_eval](http://www.grid.net.ru/nginx/eval.en.html)'s eval block running *after* [ngx_rewrite](http://wiki.nginx.org/HttpRewriteModule)'s directives. thanks Liseen Wan (xunxin).

v0.12
-----

June 22, 2010

* fixed a bug in the Content-Type output header setting handler. we should always clear `r->headers_out.content_type_lowcase`, or it'll confuse output filters like that of the gzip module.

v0.11
-----

June 08, 2010

* fixed the variables-in-Range-header issue in [more_set_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_input_headers) reported by Alexander Vetrin.

v0.10
-----

June 06, 2010

* now we can remove an input and output header *completely*, including both custom and builtin headers.

v0.09
-----

June 02, 2010

* fixed a memory initialization issue for [more_set_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_input_headers) without the `-r` option, we should always initialize `hv.replace` even when replace == 0. This may result in server segfaults and was introduced in [v0.08](http://wiki.nginx.org/HttpHeadersMoreModule#v0.08).
* implemented wildcard support in [more_clear_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_clear_headers). Thanks Bernd Dorn.

v0.08
-----

March 12, 2010

* applied the patch from Bernd Dorn to add the `-r` option to the [more_set_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_input_headers) directive.

v0.07
-----

December 24, 2009

* fixed the [more_clear_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_clear_headers) directive for builtin headers like `Server` and `Last-Modified` by always inserting an empty header when absent. Thanks Sebastiaan Deckers for reporting it.

v0.06
-----

December 15, 2009

* now the input header handler runs at the *end* of the `rewrite phase` such that it works in subrequests by default.

v0.05
-----

November 18, 2009

* fixed variables in [more_set_input_headers](http://wiki.nginx.org/HttpHeadersMoreModule#more_set_input_headers) by registering the handler in the `access phase` instead of the `rewrite` phase.

Test Suite
==========

This module comes with a Perl-driven test suite. The [test cases](http://github.com/agentzh/headers-more-nginx-module/tree/master/t/) are
[declarative](http://github.com/agentzh/headers-more-nginx-module/blob/master/t/sanity.t) too. Thanks to the [Test::Nginx](http://search.cpan.org/perldoc?Test::Nginx) module in the Perl world.

To run it on your side:


    $ PATH=/path/to/your/nginx-with-headers-more-module:$PATH prove -r t


To run the test suite with valgrind's memcheck, use the following commands:


    $ export PATH=/path/to/your/nginx-with-headers-more-module:$PATH
    $ TEST_NGINX_USE_VALGRIND=1 prove -r t


You need to terminate any Nginx processes before running the test suite if you have changed the Nginx server binary.

Because a single nginx server (by default, `localhost:1984`) is used across all the test scripts (`.t` files), it's meaningless to run the test suite in parallel by specifying `-jN` when invoking the `prove` utility.

Some parts of the test suite requires modules [proxy](http://wiki.nginx.org/HttpProxyModule), [rewrite](http://wiki.nginx.org/HttpRewriteModule), and [echo](http://wiki.nginx.org/HttpEchoModule) to be enabled as well when building Nginx.

TODO
====

* Support variables in new headers' keys.

Getting involved
================

You'll be very welcomed to submit patches to the [author](http://wiki.nginx.org/HttpHeadersMoreModule#Author) or just ask for a commit bit to the [source repository](http://wiki.nginx.org/HttpHeadersMoreModule#Source_Repository) on GitHub.

Authors
=======

* Zhang "agentzh" Yichun (章亦春) *&lt;agentzh@gmail.com&gt;*
* Bernd Dorn ( <http://www.lovelysystems.com/> )

This wiki page is also maintained by the author himself, and everybody is encouraged to improve this page as well.

Copyright & License
===================

The code base is borrowed directly from the standard [headers](http://wiki.nginx.org/HttpHeadersModule) module in Nginx 0.8.24. This part of code is copyrighted by Igor Sysoev.

Copyright (c) 2009, 2010, 2011, Taobao Inc., Alibaba Group ( <http://www.taobao.com> ).

Copyright (c) 2009, 2010, 2011, Zhang "agentzh" Yichun (章亦春) <agentzh@gmail.com>.

Copyright (c) 2010, 2011, Bernd Dorn.

This module is licensed under the terms of the BSD license.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions
are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.
* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED
TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF
LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

See Also
========

* The original thread on the Nginx mailing list that inspires this module's development: ["A question about add_header replication"](http://forum.nginx.org/read.php?2,11206,11738).
* The orginal announcement thread on the Nginx mailing list: ["The "headers_more" module: Set and clear output headers...more than 'add'!"](http://forum.nginx.org/read.php?2,23460).
* The original [blog post](http://agentzh.blogspot.com/2009/11/headers-more-module-scripting-input-and.html) about this module's initial development.
* The [echo module](http://wiki.nginx.org/HttpEchoModule) for Nginx module's automated testing.
* The standard [headers](http://wiki.nginx.org/HttpHeadersModule) module.

