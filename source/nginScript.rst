
.. meta::
  :description: Introduction to nginScript, the NGINX javascript.
  
nginScript Documentation
========================

Section 1: Overview
-------------------

nginScript is the next stage of the evolution of NGINX’s capabilities.  It’s currently at a very early stage of design and development, and we welcome feedback and suggestions to help shape the future direction of this feature.

Why?
^^^^

NGINX is a powerful web server and layer-7 proxy, configured through directives which access functionality in NGINX modules (both built-in and third-party). Variables and conditional blocks let you define complex configurations.  You can do a lot with NGINX, but you’ll eventually hit a ceiling which is limited by the imagination and capabilities of the modules built by the NGINX team and the wider community.

nginScript will let you break through that ceiling, building solutions and solving problems that no one else has anticipated.

What is nginScript
^^^^^^^^^^^^^^^^^^^^^^^^

nginScript is built of two parts:

- **A custom VM** (virtual machine) and bytecode compiler that implements a large subset of the JavaScript language.  Unlike contemporary JavaScript VMs, the NGINX VM is tailored for the NGINX environment.  It has a very low startup and tear-down time so that it can run JS snippets efficiently.  It uses a short-term memory-pool architecture rather than a garbage collector for efficient and predictable memory usage.  Finally, we’re planning a pre-emption capability so that blocking operations (such as an HTTP subrequest) can be suspended and resumed, which aligns perfectly with the event-driven NGINX architecture
- **A Configuration syntax** that allows you to embed snippets of JavaScript in your NGINX configuration.  These snippets will be evaluated at run-time, in the context of each HTTP transaction, allowing you to create much more powerful conditional configuration, modify requests and responses, and control the internal operation of NGINX precisely for each request.

What nginScript is not
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

nginScript is not an application server.  We don’t plan to create an alternative to node.js, or any other application platforms.  nginScript is targeted very firmly at extending the NGINX configuration to give you more control over HTTP traffic.

nginScript is not a full, standards-compliant implementation of ECMAscript.  It shares the same syntax and behaviour, but for efficiency and pragmatic reasons, we don’t plan to support the more esoteric parts of the JavaScript language and built-in objects that believe are not necessary for our use cases.

nginScript is not intended to replace or marginalize the excellent and highly-regarded Lua suite of modules for NGINX, or any of the other embedded languages for NGINX.  We believe in choice, we believe there’s a gap and an opportunity for JavaScript as an alternative, and we welcome the broad community of extensions to NGINX. It’s community and choice that makes NGINX so strong.

Early-Access to nginScript
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

nginScript is available as an early-access project.  It’s an external module of alpha quality, and the features and configuration syntax are likely to change as we develop the implementation and incorporate your feedback.  Please use at your own risk, but we’d love to get your feedback and ideas.

Section 2: Installing the nginScript modules
-------------------------------------------------

Installation instructions:

.. code-block:: bash

  # Obtain the latest source for NGINX from http://nginx.org/en/download.html
  $ wget http://nginx.org/download/nginx-1.11.6.tar.gz
  $ tar -xzvf nginx-1.11.6.tar.gz

  # Obtain the development sources for nginScript
  $ hg clone http://hg.nginx.org/njs

  # Configure NGINX
  $ cd nginx-1.11.6
  $ ./configure --add-module=../njs/nginx --prefix=/your/installation/directory  # for static nginScript modules
  # or
  $ ./configure --add-dynamic-module=../njs/nginx  # for dynamic nginScript modules
      
  # Build and install NGINX
  $ make
  $ make install
  
  # Enable nginScript dynamic modules in NGINX configuration file
  
    load_module modules/ngx_http_js_module.so;
    load_module modules/ngx_stream_js_module.so;
  
For more details on compiling NGINX and third-party modules, please refer to http://nginx.org/en/docs/configure.html 

Section 3: Getting Started with nginScript
------------------------------------------------

In the early-access release of nginScript, you can do several things.
Note that all JavaScript code should be inside JavaScript function that takes a request object and a response object.

Variables
^^^^^^^^^

You can declare variables using ``js_set`` by writing a JavaScript function that returns a value:

.. code-block:: nginx

  js_set $msg "function msg(req, res) {
      var m = 'Hello ';
      m += 'world!';
      return m;
  }";


These variables can be used by NGINX configuration directives. The JavaScript code is evaluated when the variable is used:

.. code-block:: nginx

  location /hello {
      add_header Content-Type text/plain;
      return 200 $msg;
  }

Content generation
^^^^^^^^^^^^^^^^^^

The ``js_run`` directive is evaluated at the content-generation stage. It’s used to execute JavaScript natively and generate an HTTP response:

.. code-block:: nginx

  location /hello {
      js_run "function hello(req, res) {
          res.contentType = 'text/plain';
          res.status = 200;
          res.sendHeader();

          res.send( 'Hello, world!' );
          res.finish();
      }";
  }


The request object
^^^^^^^^^^^^^^^^^^

The nginScript environment provides a request object and passes it to your function as the first parameter.  You can read and set the properties of this object and use the methods it provides to access and modify the request.

.. code-block:: nginx

  js_set $summary "function summary(req, res) {
      var a, s, h;

      s = 'Request summary\n\n';

      s += 'Method: ' + req.method + '\n';
      s += 'HTTP version: ' + req.httpVersion + '\n';
      s += 'Host: ' + req.headers.host + '\n';
      s += 'Remote Address: ' + req.remoteAddress + '\n';
      s += 'URI: ' + req.uri + '\n';

      s += 'Headers:\n';
      for (h in req.headers) {
          s += '  header \"' + h + '\" is \"' + req.headers[h] + '\"\n';
      }

      s += 'Args:\n';
      for (a in req.args) {
          s += '  arg \"' + a + '\" is \"' + req.args[a] + '\"\n';
      }

      return s;
  }";

The response object
^^^^^^^^^^^^^^^^^^^

The nginScript environment passes a response object as the second function parameter.
You can use the response object to generate a response during variable evaluation or content generation:

.. code-block:: nginx

  js_run "function hello(req, res) {
      res.contentType = 'text/plain';
      res.status = 200;
      res.sendHeader();

      res.send( 'Hello, world!' );
      res.finish();
  }";

Bringing it all together
^^^^^^^^^^^^^^^^^^^^^^^^

The following example illustrates how to obtain a parameter from the query string of a request and generate a response.

.. code-block:: nginx

 location /fib {
     js_run "function f( n ) { return ( n < 2 ) ? 1: f( n-1 ) + f( n-2 ) ; }
          function fib(req, res) {
          
          var nn = req.args['n'];

          // nn++ is a hack to convert nn to an integer
          var n = nn++;

          var msg = 'Fibonacci( ' + n + ' ) = ' + f( n );

          res.contentType = 'text/plain';
          res.status = 200;
          res.sendHeader();

          res.send( msg );
          res.send( '\n' );
          res.finish();
      }";
  }

.. note:: If more than one JavaScript functions are defined, the nginScript environment will pass the request and response objects to the function that was defined last.

Section 4: Documentation
------------------------

Syntax and Execution
^^^^^^^^^^^^^^^^^^^^

Syntax for JavaScript variables
When are variables evaluated (and the JS code executed)

The Request object
^^^^^^^^^^^^^^^^^^

List of fields.  Mutable fields (values that can be changed) and Immutable fields (values that cannot be changed)

List of methods

Tuning and Configuration
^^^^^^^^^^^^^^^^^^^^^^^^

Any tunables or configuration for nJS

Caveats and Limitations
^^^^^^^^^^^^^^^^^^^^^^^

nginScript supports a subset of the JS language.

Specific exclusions (e.g. no closures, no eval, etc)

It’s not our goal to create a complete implementation of the JS/ECMAscript standard.  Implement sufficient functionality that users can create sophisticated rules in NGINX to control how requests and responses are processed.

Further examples
^^^^^^^^^^^^^^^^

Any other complete examples that we can share?


Feedback - what we want you to do
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Where to share feedback nginx-devel@nginx.org.

What sort of feedback we’re interested in:
suggestions for future features
recommendations on architectural improvements

.. note::
  This is an early-access release of nginScript.  Because the code is changing frequently, we’re not in a position to review and accept code contributions or patches at present.
