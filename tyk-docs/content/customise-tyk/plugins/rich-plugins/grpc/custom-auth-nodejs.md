---
date: 2017-03-24T13:28:45Z
title: Create Custom Authentication Plugin with NodeJS
menu:
  main:
    parent: "gRPC"
weight: 3 
---

## <a name="introduction"></a>Introduction

This tutorial will guide you through the creation of a custom authentication plugin for Tyk with a gRPC based plugin written in NodeJS.

For additional information about gRPC, check the official documentation [here](https://grpc.io/docs/guides/index.html).

## <a name="Requirements"></a>Requirements

* Tyk Gateway: This can be installed using standard package management tools like Yum or APT, or from source code. See [here][1] for more installation options.
* The Tyk CLI utility, which is bundled with our RPM and DEB packages, and can be installed separately from [https://github.com/TykTechnologies/tyk-cli][2]
* * NodeJS v6.x.x https://nodejs.org/en/download/ 


## <a name="what-is-grpc"></a>What is gRPC?

gRPC is a very powerful framework for RPC communication across different languages. It was created by Google and makes heavy use of HTTP2 capabilities and the Protocol Buffers serialization mechanism.

## <a name="why-use-it"></a>Why Use it for Plugins?
When it comes to built-in plugins, we have been able to integrate several languages like Python, JavaScript & Lua in a native way: this means the middleware you write using any of these languages runs in the same process. For supporting additional languages we have decided to integrate gRPC connections and perform the middleware operations outside of the Tyk process. The flow of this approach is as follows: 

* Tyk receives a HTTP request.
* Your gRPC server performs the middleware operations (for example, any modification of the request object).
* Your gRPC server sends the request back to Tyk.
* Tyk proxies the request to your upstream API.

The sample code that we'll use implements a very simple authentication layer using NodeJS and the proper gRPC bindings generated from our Protocol Buffers definition files.

## <a name="create"></a>Create the Plugin

### Setting up the NodeJS Project

We will use the NPM tool to initialize our project, follow the steps provided by the `init` command:

```{.copyWrapper}
cd ~
mkdir tyk-plugin
cd tyk-plugin
npm init
```

Now we'll add the gRPC package for this project:

```{.copyWrapper}
npm install --save grpc
```

### gRPC Tools and Bindings Generation

Typically to use gRPC and Protocol Buffers you need to use a code generator and generate bindings for the target language that you're using. For this tutorial we'll skip this step and use the dynamic loader that's provided by the NodeJS gRPC library, this mechanism allows a program to load Protocol Buffers definitions directly from `.proto` files, see [this section](https://grpc.io/docs/tutorials/basic/node.html#loading-service-descriptors-from-proto-files) in the gRPC documentation for more details.

To fetch the required `.proto` files, you may use a official repository where we keep the Tyk Protocol Buffers definition files:

```{.copyWrapper}
cd ~/tyk-plugin
git clone https://github.com/TykTechnologies/tyk-protobuf
```


### Server Implementation

Now we're ready to implement our gRPC server, create a file called `main.js` in the project's directory

Add the following code to `main.js`.

```{javascript}
const grpc = require('grpc'),
    resolve = require('path').resolve

const tyk = grpc.load({
    file: 'coprocess_object.proto',
    root: resolve(__dirname, 'tyk-protobuf/proto')
  }).coprocess

const listenAddr = '127.0.0.1:5555',
      authHeader = 'Authorization'
      validToken = '71f6ac3385ce284152a64208521c592b'

// The dispatch function is called for every hook:
const dispatch = (call, callback) => {
    var obj = call.request
    // We dispatch the request based on the hook name, we pass obj.request which is the coprocess.Object:
    switch (obj.hook_name) {
        case 'MyPreMiddleware':
            preMiddleware(obj, callback)
            break
        case 'MyAuthMiddleware':
            authMiddleware(obj, callback)
            break
        default:
            callback(null, obj)
            break
    }
}

const preMiddleware = (obj, callback) => {
    var req = obj.request

    // req is the coprocess.MiniRequestObject, we inject a header using the "set_headers" field:
    req.set_headers = {
        'mycustomheader': 'mycustomvalue'
    }

    // Use this callback to finish the operation, sending back the modified object:
    callback(null, obj)
}

const authMiddleware = (obj, callback) => {
    var req = obj.request

    // We take the value from the "Authorization" header:
    var token = req.headers[authHeader]

    // The token should be attached to the object metadata, this is used internally for key management:
    obj.metadata = {
        token: token
    }

    // If the request token doesn't match the  "validToken" constant we return the call:
    if (token != validToken) {
        callback(null, obj)
        return
    }

    // At this point the token is valid and a session state object is initialized and attached to the coprocess.Object:
    var session = new tyk.SessionState()
    session.id_extractor_deadline = Date.now() + 100000000000
    obj.session = session
    callback(null, obj)
}

main = function() {
    server = new grpc.Server()
    server.addService(tyk.Dispatcher.service, {
        dispatch: dispatch
    })
    server.bind(listenAddr, grpc.ServerCredentials.createInsecure())
    server.start()
}

main()
```


To run the gRPC server run:

```{.copyWrapper}
node main.js
```

The gRPC server will listen on port `5555` (see the `listenAddr` constant). In the next steps we'll setup the plugin bundle and modify Tyk to connect to our gRPC server.


## <a name="bundle"></a>Setting up the Plugin Bundle

We need to create a manifest file within the `tyk-plugin` directory. This file contains information about our plugin and how we expect it to interact with the API that will load it. This file should be named `manifest.json` and needs to contain the following:

```{json}
{
    "custom_middleware": {
        "driver": "grpc",
        "auth_check": {
            "name": "MyAuthMiddleware"
        }
    }
}
```

* The `custom_middleware` block contains the middleware settings like the plugin driver we want to use (`driver`) and the hooks that our plugin will expose. We use the `auth_check` hook for this tutorial. For other hooks see [here](https://tyk.io/docs/customise-tyk/plugins/rich-plugins/rich-plugins-work/#coprocess-dispatcher-hooks).
* The `name` field references the name of the function that we implement in our plugin code - `MyAuthMiddleware`. The implemented dispatcher uses a switch statement to handle this hook, and calls the `authMiddleware` function in `main.js`.

To bundle our plugin run the following command in the `tyk-plugin` directory. Check your tyk-cli install path first:

```{.copyWrapper}
/opt/tyk-gateway/utils/tyk-cli bundle build -y
```


A plugin bundle is a packaged version of the plugin. It may also contain a cryptographic signature of its contents. The `-y` flag tells the Tyk CLI tool to skip the signing process in order to simplify the flow of this tutorial. 

For more information on the Tyk CLI tool, see [here](https://tyk.io/docs/customise-tyk/plugins/rich-plugins/plugin-bundles/#using-the-bundler-tool).

You should now have a `bundle.zip` file in the `tyk-plugin` directory.

## <a name="publish"></a>Publish the Plugin

To publish the plugin, copy or upload `bundle.zip` to a local web server like Nginx, Apache or storage like Amazon S3. For this tutorial we'll assume you have a web server listening on `localhost` and accessible through `http://localhost`.

## <a name="configure-tyk"></a>Configure Tyk

You will need to modify the Tyk global configuration file `tyk.conf` to use gRPC plugins. The following block should be present in this file:

```{.copyWrapper}
"coprocess_options": {
    "enable_coprocess": true,
    "coprocess_grpc_server": "tcp://localhost:5555"
},
"enable_bundle_downloader": true,
"bundle_base_url": "http://localhost/bundles/",
"public_key_path": ""
```


### tyk.conf Options

* `enable_coprocess`: This enables the plugin.
* `coprocess_grpc_server`: This is the URL of our gRPC server.
* `enable_bundle_downloader`: This enables the bundle downloader.
* `bundle_base_url`: This is a base URL that will be used to download the bundle. You should replace the bundle_base_url with the appropriate URL of the web server that's serving your plugin bundles. For now HTTP and HTTPS are supported but we plan to add more options in the future (like pulling directly from S3 buckets).
* `public_key_path`: Modify `public_key_path` in case you want to enforce the cryptographic check of the plugin bundle signatures. If the `public_key_path` isn't set, the verification process will be skipped and unsigned plugin bundles will be loaded normally.


### Configure an API Definition

There are two important parameters that we need to add or modify in the API definition.
The first one is `custom_middleware_bundle` which must match the name of the plugin bundle file. If we keep this with the default name that the Tyk CLI tool uses, it will be `bundle.zip`:

```{json}
"custom_middleware_bundle": "bundle.zip"
```

Assuming the `bundle_base_url` is `http://localhost/bundles/`, Tyk will use the following URL to download our file:

`http://localhost/bundles/bundle.zip`

The second parameter is specific to this tutorial, and should be used in combination with `use_keyless` to allow an API to authenticate against our plugin:

```{json}
"use_keyless": false,
"enable_coprocess_auth": true
```


`enable_coprocess_auth` will instruct the Tyk gateway to authenticate this API using the associated custom authentication function that's implemented by our plugin.

### Configuration via the Tyk Dashboard

To attach the plugin to an API, from the **Advanced Options** tab in the **API Designer** enter `bundle.zip` in the **Plugin Bundle ID** field.

![Plugin Options][3]

We also need to modify the authentication mechanism that's used by the API.
From the **Core Settings** tab in the **API Designer** select **Use Custom Auth (plugin)** from the **Target Details - Authentication Mode** drop-down list. 

![Advanced Options][4]

## <a name="testing"></a>Testing the Plugin


At this point we have our test HTTP server ready to serve the plugin bundle and the configuration with all the required parameters.
The final step is to start or restart the **Tyk Gateway** (this may vary depending on how you set up Tyk):

```{.copyWrapper}
service tyk-gateway start
```


A simple CURL request will be enough for testing our custom authentication middleware.

This request will trigger an authentication error:

```{.copyWrapper}
curl http://localhost:8080/my-api/my-path -H 'Authorization: badtoken'
```

This will trigger a successful authentication. We're using the token that's specified in our server implementation (see line 57 in `Server.cs`):


```{.copyWrapper}
curl http://localhost:8080/my-api/my-path -H 'Authorization: abc123'
```

## <a name="next"></a>What's Next?

In this tutorial we learned how Tyk gRPC plugins work. For a production-level setup we suggest the following:

* Configure an appropriate web server and path to serve your plugin bundles.











[1]: https://tyk.io/docs/get-started/with-tyk-on-premise/installation/
[2]: https://github.com/TykTechnologies/tyk-cli
[3]: /docs/img/dashboard/system-management/plugin_options.png
[4]: /docs/img/dashboard/system-management/plugin_auth_mode.png




