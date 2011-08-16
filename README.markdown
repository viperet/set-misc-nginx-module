Name
====

**ngx_set_misc** - Various set_xxx directives added to nginx's rewrite module (md5/sha1, sql/json quoting, and many more)

*This module is not distributed with the Nginx source.* See [the installation instructions](http://wiki.nginx.org/NginxHttpSetMiscModule#Installation).

Version
=======

This document describes set-misc-nginx-module [v0.22rc2](https://github.com/agentzh/set-misc-nginx-module/downloads) released on 27 July 2011.

Synopsis
========


    location /foo {
        set $a $arg_a;
        set_if_empty $a 56;

        # GET /foo?a=32 will yield $a == 32
        # while GET /foo and GET /foo?a= will
        # yeild $a == 56 here.
    }

    location /bar {
        set $foo "hello\n\n'\"\\";
        set_quote_sql_str $foo $foo; # for mysql

        # OR in-place editing:
        #   set_quote_sql_str $foo;

        # now $foo is: 'hello\n\n\'\"\\'
    }

    location /bar {
        set $foo "hello\n\n'\"\\";
        set_quote_pgsql_str $foo;  # for PostgreSQL

        # now $foo is: E'hello\n\n\'\"\\'
    }

    location /json {
        set $foo "hello\n\n'\"\\";
        set_quote_json_str $foo $foo;

        # OR in-place editing:
        #   set_quote_json_str $foo;

        # now $foo is: "hello\n\n'\"\\"
    }

    location /baz {
        set $foo "hello%20world";
        set_unescape_uri $foo $foo;

        # OR in-place editing:
        #   set_unescape_uri $foo;

        # now $foo is: hello world
    }

    upstream_list universe moon sun earth;
    upstream moon { ... }
    upstream sun { ... }
    upstream earth { ... }
    location /foo {
        set_hashed_upstream $backend universe $arg_id;
        drizzle_pass $backend; # used with ngx_drizzle
    }

    location /base32 {
        set $a 'abcde';
        set_encode_base32 $a;
        set_decode_base32 $b $a;

        # now $a == 'c5h66p35' and
        # $b == 'abcde'
    }

    location /base64 {
        set $a 'abcde';
        set_encode_base64 $a;
        set_decode_base64 $b $a;

        # now $a == 'YWJjZGU=' and
        # $b == 'abcde'
    }

    location /hex {
        set $a 'abcde';
        set_encode_hex $a;
        set_decode_hex $b $a;

        # now $a == '6162636465' and
        # $b == 'abcde'
    }

    # GET /sha1 yields the output
    #   aaf4c61ddcc5e8a2dabede0f3b482cd9aea9434d
    location /sha1 {
        set_sha1 $a hello;
        echo $a;
    }

    # ditto
    location /sha1 {
        set $a hello;
        set_sha1 $a;
        echo $a;
    }

    # GET /today yields the date of today in local time using format 'yyyy-mm-dd'
    location /today {
        set_local_today $today;
        echo $today;
    }

    # GET /signature yields the hmac-sha-1 signature
    # given a secret and a string to sign
    # this example yields the base64 encoded singature which is
    # "HkADYytcoQQzqbjQX33k/ZBB/DQ="
    location /signature {
        set $secret_key 'secret-key';
        set $string_to_sign "some-string-to-sign";
        set_hmac_sha1 $signature $secret_key $string_to_sign;
        set_encode_base64 $signature $signature;
        echo $signature;
    }

    location = /rand {
        set $from 3;
        set $to 15;
        set_random $rand $from $to;

        # or write directly
        #   set_random $rand 3 15;

        echo $rand;  # will print a random integer in the range [3, 15]
    }


Description
===========

This module extends the standard NginxHttpRewriteModule's directive set to provide more functionalities like URI escaping and unescaping, JSON quoting, Hexadecimal/MD5/SHA1/Base32/Base64 digest encoding and decoding, random number generator, and more!

Every directive provided by this module can be mixed freely with other [NginxHttpRewriteModule](http://wiki.nginx.org/NginxHttpRewriteModule)'s directives, like [if](http://wiki.nginx.org/NginxHttpRewriteModule#if) and [set](http://wiki.nginx.org/NginxHttpRewriteModule#set). (Thanks to the [Nginx Devel Kit](https://github.com/simpl/ngx_devel_kit)!)

Directives
==========

set_if_empty
------------
**syntax:** *set_if_empty $dst <src>*

**default:** *no*

**context:** *location, location if*

Assign the value of the argument `<src>` if and only if variable `$dst` is empty (i.e., not found or has an empty string value).

In the following example,


    set $a 32;
    set_if_empty $a 56;


the variable `$dst` will take the value 32 at last. But in the sample


    set $a '';
    set $value "hello, world"
    set_if_empty $a $value;


`$a` will take the value `"hello, world"` at last.

set_quote_sql_str
-----------------
**syntax:** *set_quote_sql_str $dst <src>*

**syntax:** *set_quote_sql_str $dst*

**default:** *no*

**context:** *location, location if*

**category:** *ndk_set_var_value*

When taking two arguments, this directive will quote the value of the second argument `<src>` by MySQL's string value quoting rule and assign the result into the first argument, variable `$dst`. For example,


    location /test {
        set $value "hello\n\r'\"\\";
        set_quote_sql_str $quoted $value;
    
        echo $quoted;
    }


Then request `GET /test` will yield the following output


    'hello\n\r\'\"\\'


Please note that we're using [NginxHttpEchoModule](http://wiki.nginx.org/NginxHttpEchoModule)'s [echo directive](http://wiki.nginx.org/NginxHttpEchoModule#echo) here to output values of nginx variables directly.

When taking a single argument, this directive will does in-place modification of the argument variable. For example,


    location /test {
        set $value "hello\n\r'\"\\";
        set_quote_sql_str $value;
    
        echo $value;
    }


then request `GET /test` will give exactly the same output as the previous example.

This directive is usually used to prevent SQL injection.

This directive can be invoked by [NginxHttpLuaModule](http://wiki.nginx.org/NginxHttpLuaModule)'s [ndk.set_var.DIRECTIVE](http://wiki.nginx.org/NginxHttpLuaModule#ndk.set_var.DIRECTIVE) interface and [NginxHttpArrayVarModule](http://wiki.nginx.org/NginxHttpArrayVarModule)'s [array_map_op](http://wiki.nginx.org/NginxHttpArrayVarModule#array_map_op) directive.

set_quote_pgsql_str
-------------------
**syntax:** *set_quote_pgsql_str $dst <src>*

**syntax:** *set_quote_pgsql_str $dst*

**default:** *no*

**context:** *location, location if*

**category:** *ndk_set_var_value*

Very much like [set_quote_sql_str](http://wiki.nginx.org/NginxHttpSetMiscModule#set_quote_sql_str), but with PostgreSQL quoting rules for SQL string literals.

set_quote_json_str
------------------
**syntax:** *set_quote_json_str $dst <src>*

**syntax:** *set_quote_json_str $dst*

**default:** *no*

**context:** *location, location if*

**category:** *ndk_set_var_value*

When taking two arguments, this directive will quote the value of the second argument `<src>` by JSON string value quoting rule and assign the result into the first argument, variable `$dst`. For example,


    location /test {
        set $value "hello\n\r'\"\\";
        set_quote_json_str $quoted $value;
    
        echo $quoted;
    }


Then request `GET /test` will yield the following output


    "hello\n\r'\"\\"


Please note that we're using [NginxHttpEchoModule](http://wiki.nginx.org/NginxHttpEchoModule)'s [echo directive](http://wiki.nginx.org/NginxHttpEchoModule#echo) here to output values of nginx variables directly.

When taking a single argument, this directive will does in-place modification of the argument variable. For example,


    location /test {
        set $value "hello\n\r'\"\\";
        set_quote_json_str $value;
    
        echo $value;
    }


then request `GET /test` will give exactly the same output as the previous example.

This directive can be invoked by [NginxHttpLuaModule](http://wiki.nginx.org/NginxHttpLuaModule)'s [ndk.set_var.DIRECTIVE](http://wiki.nginx.org/NginxHttpLuaModule#ndk.set_var.DIRECTIVE) interface and [NginxHttpArrayVarModule](http://wiki.nginx.org/NginxHttpArrayVarModule)'s [array_map_op](http://wiki.nginx.org/NginxHttpArrayVarModule#array_map_op) directive.

set_unescape_uri
----------------
**syntax:** *set_unescape_uri $dst <src>*

**syntax:** *set_unescape_uri $dst*

**default:** *no*

**context:** *location, location if*

**category:** *ndk_set_var_value*

When taking two arguments, this directive will unescape the value of the second argument `<src>` as a URI component and assign the result into the first argument, variable `$dst`. For example,


    location /test {
        set_unescape_uri $key $arg_key;
        echo $key;
    }


Then request `GET /test?key=hello+world%21` will yield the following output


    hello world!


The nginx standard [$arg_PARAMETER](http://wiki.nginx.org/NginxHttpCoreModule#.24arg_PARAMETER) variable holds the raw (escaped) value of the URI parameter. So we need the `set_unescape_uri` directive to unescape it first.

Please note that we're using [NginxHttpEchoModule](http://wiki.nginx.org/NginxHttpEchoModule)'s [echo directive](http://wiki.nginx.org/NginxHttpEchoModule#echo) here to output values of nginx variables directly.

When taking a single argument, this directive will does in-place modification of the argument variable. For example,


    location /test {
        set $key $arg_key;
        set_unescape_uri $key;

        echo $value;
    }


then request `GET /test?key=hello+world%21` will give exactly the same output as the previous example.

This directive can be invoked by [NginxHttpLuaModule](http://wiki.nginx.org/NginxHttpLuaModule)'s [ndk.set_var.DIRECTIVE](http://wiki.nginx.org/NginxHttpLuaModule#ndk.set_var.DIRECTIVE) interface and [NginxHttpArrayVarModule](http://wiki.nginx.org/NginxHttpArrayVarModule)'s [array_map_op](http://wiki.nginx.org/NginxHttpArrayVarModule#array_map_op) directive.

set_escape_uri
--------------
**syntax:** *set_escape_uri $dst <src>*

**syntax:** *set_escape_uri $dst*

**default:** *no*

**context:** *location, location if*

**category:** *ndk_set_var_value*

Very much like the [set_unescape_uri](http://wiki.nginx.org/NginxHttpSetMiscModule#set_unescape_uri) directive, but does the conversion the other way around, i.e., URL component escaping.

set_hashed_upstream
-------------------

set_encode_base32
-----------------

RFC forces the [A-Z2-7] RFC-3548 compliant encoding, but we're using the "base32hex" encoding [0-9a-v].

By default, the "=" character is used to pad the left-over bytes due to alignment. But the padding behavior can be completely disabled by setting set_misc_base32_padding off.

set_misc_base32_padding
-----------------------
**syntax:** *set_misc_base32_padding on|off*

**default:** *on*

**context:** *http, server, server if, location, location if*

This directive can control whether to pad left-over bytes with the "=" character when encoding a base32 digest by the [set_encode_base32](http://wiki.nginx.org/NginxHttpSetMiscModule#set_encode_base32) directive.

set_decode_base32
-----------------

RFC forces the [A-Z2-7] RFC-3548 compliant encoding, but we're using the "base32hex" encoding [0-9a-v].

set_encode_base64
-----------------

set_decode_base64
-----------------

set_encode_hex
--------------

set_decode_hex
--------------

set_sha1
--------

set_md5
-------

set_local_today
---------------

set_hmac_sha1
-------------

set_random
----------
**syntax:** *set_random $res <from> <to>*

Note that only non-negative numbers in the "from" to "to" argument are allowed. A (psuedo) random number in the range `[<from>, <to>]` (inclusive) will be assigned to `$res`. For now, there's no way to configure a custom random generator seed.

Behind the scene, it makes use of the standard C function `rand()`.

Caveats
=======

Do not use [$arg_PARAMETER](http://wiki.nginx.org/NginxHttpCoreModule#.24arg_PARAMETER) or [$http_HEADER](http://wiki.nginx.org/NginxHttpCoreModule#.24http_HEADER) or other special variables defined in the nginx core module as the target variable in this module's directives. For instance,


    set_if_empty $arg_user 'foo';  # DO NOT USE THIS!


may lead to data corruption.

Installation
============

Grab the nginx source code from [nginx.org](http://nginx.org/), for example,
the version 1.0.5 (see [nginx compatibility](http://wiki.nginx.org/NginxHttpSetMiscModule#Compatibility)), and then build the source with this module:


    $ wget 'http://sysoev.ru/nginx/nginx-1.0.5.tar.gz'
    $ tar -xzvf nginx-1.0.5.tar.gz
    $ cd nginx-1.0.5/
    
    # Here we assume you would install you nginx under /opt/nginx/.
    $ ./configure --prefix=/opt/nginx \
        --add-module=/path/to/set-misc-nginx-module
     
    $ make -j2
    $ make install


Download the latest version of the release tarball of this module from [set-misc-nginx-module file list](http://github.com/agentzh/set-misc-nginx-module/downloads).

Compatibility
=============

The following versions of Nginx should work with this module:

* **1.0.x**                       (last tested: 1.0.5)
* **0.9.x**                       (last tested: 0.9.4)
* **0.8.x**                       (last tested: 0.8.54)
* **0.7.x >= 0.7.46**             (last tested: 0.7.68)

If you find that any particular version of Nginx above 0.7.46 does not work with this module, please consider [reporting a bug](http://wiki.nginx.org/NginxHttpSetMiscModule#Report_Bugs).

Report Bugs
===========

Although a lot of effort has been put into testing and code tuning, there must be some serious bugs lurking somewhere in this module. So whenever you are bitten by any quirks, please don't hesitate to

1. send a bug report or even patches to <agentzh@gmail.com>,
1. or create a ticket on the [issue tracking interface](http://github.com/agentzh/set-misc-nginx-module/issues) provided by GitHub.

Source Repository
=================

Available on github at [agentzh/set-misc-nginx-module](http://github.com/agentzh/set-misc-nginx-module).

ChangeLog
=========

Test Suite
==========

This module comes with a Perl-driven test suite. The [test cases](http://github.com/agentzh/set-misc-nginx-module/tree/master/t/) are
[declarative](http://github.com/agentzh/set-misc-nginx-module/blob/master/t/escape-uri.t) too. Thanks to the [Test::Nginx](http://search.cpan.org/perldoc?Test::Nginx) module in the Perl world.

To run it on your side:


    $ PATH=/path/to/your/nginx-with-set-misc-module:$PATH prove -r t


You need to terminate any Nginx processes before running the test suite if you have changed the Nginx server binary.

Because a single nginx server (by default, `localhost:1984`) is used across all the test scripts (`.t` files), it's meaningless to run the test suite in parallel by specifying `-jN` when invoking the `prove` utility.

Getting involved
================

You'll be very welcomed to submit patches to the [author](http://wiki.nginx.org/NginxHttpSetMiscModule#Author) or just ask for a commit bit to the [source repository](http://wiki.nginx.org/NginxHttpSetMiscModule#Source_Repository) on GitHub.

Author
======

agentzh (章亦春) *<agentzh@gmail.com>*

This wiki page is also maintained by the author himself, and everybody is encouraged to improve this page as well.

Copyright & License
===================

Copyright (c) 2009, 2010, 2011, Taobao Inc., Alibaba Group ( <http://www.taobao.com> ).

Copyright (c) 2009, 2010, 2011, Zhang "agentzh" Yichun (章亦春) <agentzh@gmail.com>.

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
* [Nginx Devel Kit](https://github.com/simpl/ngx_devel_kit)
* [The ngx_openresty bundle](http://openresty.org)
