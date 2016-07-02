# serverless-sentry-plugin

[![serverless](http://public.serverless.com/badges/v3.svg)](http://www.serverless.com)

This plugin adds automatic forwarding of errors and exceptions to Sentry (getsentry.com) to Serverless 0.5.x.


## Overview
The plugin lets you forward errors and exceptions in your Lambda code to Sentry (getsentry.com) without
requiring any code changes.

**IMPORTANT:** Currently this plugin only supports the `nodejs` and `nodejs4.3` runtimes.
Any help to add Python support is appreciated.


## Installation

1. Install the plugin module.

   `npm install serverless-sentry-plugin --save` will install the latest version of the plugin.

   If you want to debug, you also can reference the source repository at a specific version or branch
   with `npm install https://github.com/arabold/serverless-sentry-plugin#<tag or branch name>`

2. Install the `raven` module to forward exceptions to Sentry.

   `npm install raven --save`

   If you use a different `package.json` for your source code than for your project itself, then make
   sure to add `raven` there. 

3. Set the `SENTRY_DSN` as well as the `SERVERLESS_*` environment variables.

   Open up your `s-function.json` and make sure to set the following environment variables. Without
   `SENTRY_DSN` set the plugin will not report any errors.
   ```
   {
     "name": "test-function",
     "runtime": "nodejs4.3",
     "handler": "handler.handler",
     "timeout": 6,
     "memorySize": 1024,
     "custom": {},
     "environment": {
       "SERVERLESS_PROJECT_NAME": "${project}",
       "SERVERLESS_STAGE": "${stage}",
       "SERVERLESS_REGION": "${region}",
       "SENTRY_DSN": "https://*****@app.getsentry.com/76507"
     }
   }
   ```

4. Activate the plugin in your Serverless project.

   Add `serverless-sentry-plugin` to the plugins array in your `s-project.json`.
   ```
   {
     "name": "my-project",
     "custom": {},
     "plugins": [
       "serverless-sentry-plugin"
     ]
   }
   ```


## Usage

The plugin automatically hooks into `serverless function deploy` and does not require
any manual invocation.


## How It Works
The plugin injects code into the generated `_serverless_handler.js` Lambda entry point and
extends it with automatic error reporting. Whenever your Lambda handler sets an error response, it is
forwarded to Sentry with additional context information.

### Capturing Custom Messages and Exceptions
When installed and enabled, the plugin exposes a new global object `sls_raven` that you can use
to capture messages and exceptions to Sentry.
```
  if (global.sls_raven) {
    global.sls_raven.captureMessage("Hello from Lambda", { level: "info" });
  }
```

This instance of the [Raven](https://github.com/getsentry/raven-node/) client is preconfigured and sets
some useful default tags. For further documentation on how to use it to capture your own messages refer to 
[docs.getsentry.com](https://docs.getsentry.com/hosted/clients/node/usage/).

### Local Development
This plugin will inject code into your Lambda functions during _deployment_ only. In other words the global
`sls_raven` object will _not_ be available and error responses will _not_ be captured while developing
locally. However, this is intended to not clutter your Sentry logs with unwanted debug information. If you
need `sls_raven` also during local development, you can instantiate your own version somewhere 
at the beginning of your code base:
```
global.sls_raven = global.sls_raven || new (require("raven").Client)();
```

### Detecting Slow Running Code
It's a good practice to specify the function timeout in `s-function.json` to be at last twice as large 
as the _expected maximum execution time_. If you specify a timeout of 6 seconds (the default), this plugin will
warn you if the function runs for 3 or more seconds. That means it's time to either review your code for
possible performance improvements or increase the timeout value slightly.

### Low Memory Warnings
The plugin will automatically generate a warning if the memory consuption of your Lambda function crosses
75% of the allocated memory limit. For this the plugin samples the amount of memory used by Node.js every 
250 milliseconds (using `process.memoryUsage()`), independently of any garbage collection. As with
all Node.js code, it is important to remember that JavaScript code runs single-threaded and the monitoring function
will only be able to sample memory usage, if your code is in a wait state, e.g. during database queries or 
when calling asynchronous functions with a callback.

Only one low memory warning will be generated per function invocation. You might want to increase the
memory limit step by step until your code runs without warnings.

### Turn Sentry Reporting On/Off
As stated before, the plugin only runs if the `SENTRY_DSN` environment variable is set. This is an easy
way to enable or disable reporting for specific functions through your `s-function.json`.

In some cases it might be desirable to disable only error reporting but keep the timeout and low memory 
warnings in place. This can be achieved via setting the flag `ignoreErrorResponses` to `true`
in your `s-function.json`:
```
{
  ...
  "custom": {
    "sentry": {
      "captureErrors": false,
      "captureUnhandledExceptions": true,
      "captureMemoryWarnings": true,
      "captureTimeoutWarnings": true
    }
  }
  ...
```

### Minified Code (e.g. Serverless Optimizer Plugin)
For minified projects with a large code base and many dependencies the `raven` reported might fail 
to publish events to Sentry. This happens due to an issue with serializing the call stack in `raven`. 
As a workaround please _disable minification for your project_. You can still use the optimizer plugin to 
compact all dependencies into a single JavaScript file and/or babelify your code as usual.


## Releases

### 0.2.0
* Properly capture error strings returned by your Lambda function.
* Improved timeout and out of memory tracking and reporting.
* Capture unhandled exceptions and forward them to Sentry.
* Allow more granular configuration of which errors and warnings should be tracked.
* Log Amazon Cognito identity (if available) in the Sentry user context.

### 0.1.4
* Low memory warning! The plugin now automatically raises a warning if your Lambda function uses 75%
  or more of the assigned memory limit. 

### 0.1.3
* Fixed timeout detection for Node 0.10.42 style use of `context.succeed` and `context.fail`

### 0.1.2
* Expose the raven client as global object `global.sls_raven`. This is only set if `SENTRY_DSN` is
  set for the current function.

### 0.1.1
* Emit a warning if your Lambda function runs for more than half the specified timeout value.

### 0.1.0
* Initial release

### To Dos
* Create releases in Sentry when functions are deployed.
* Support for minified code and source map files (might require the use of `raven-js` instead of `raven-node`).
* Warn if `raven-node` module isn't available; install it automatically using `npm`.
* Add support for Python runtime.
