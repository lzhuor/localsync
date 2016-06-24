[![NPM](https://raw.githubusercontent.com/noderaider/localsync/master/public/images/localsync.gif)](https://npmjs.com/packages/localsync)

**a lightweight module to sync JS objects in realtime across tabs / windows of a browser.**

**:exclamation: v1.1.x: Now a modular lerna repo! Synchronization strategies have been split out into separate packages.**

#### Features

* Uses local storage event emitters to sync objects in realtime across tabs.
* Never calls the tab that the event occurred on.
* Falls back to cookie polling internally if using an unsupported browser (IE 9+ / Edge).
* Isomorphic.
* Tested with mocha.

[![Build Status](https://travis-ci.org/noderaider/localsync.svg?branch=master)](https://travis-ci.org/noderaider/localsync)
[![codecov](https://codecov.io/gh/noderaider/localsync/branch/master/graph/badge.svg)](https://codecov.io/gh/noderaider/localsync)

[![NPM](https://nodei.co/npm/localsync.png?stars=true&downloads=true)](https://nodei.co/npm/localsync/)


## Install

`npm install -S localsync`


## How to use

```js
import localsync from 'localsync'

/** Create an action that will trigger a sync to other tabs. */
const action = (userID, first, last) => ({ userID, first, last })

/** Create a handler that will run on all tabs that did not trigger the sync. */
const handler = (value, old, url) => {
  console.info(`Another tab at url ${url} switched user from "${old.first} ${old.last}" to "${value.first} ${value.last}".`)
  // do something with value.userID
}

/** Create a synchronizer. localsync supports N number of synchronizers for different things across your app. The key 'user' defines a localsync synchronization channel. */
const usersync = localsync('user', action, handler)

/** Start synchronizing. */
usersync.start()

/** IE / Edge do not support local storage across multiple tabs. localsync will automatically fallback to a cookie polling mechanism here. You don't need to do anything else. */
if(usersync.isFallback)
  console.warn('browser doesnt support local storage synchronization, falling back to cookie synchronization.')

/** Trigger an action that will get handled on other tabs. */
usersync.trigger(1, 'jimmy', 'john')

console.info(usersync.mechanism) /** => 'storagesync' on chrome, 'cookiesync' on IE */

setTimeout(() => {
  /** Trigger another action in 5 seconds. */
  usersync.trigger(2, 'jane', 'wonka')
}, 5000)

setTimeout(() => {
  /** If its still running, stop syncing in 10 seconds. */
  if(usersync.isRunning)
    usersync.stop()
}, 10000)
```

## Structure

**localsync has a singular purpose: to synchronize events from one client to many using a common interface and the least invasive mechanism for the current browsing medium.**

Internally, localsync is comprised of several small 'sync' packages that all adhere to the common localsync interface. The main localsync package does no actual synchronization on its own but rather determines the most appropriate synchronization strategy and calls upon the necessary packages to invoke it. All the packages with brief descriptions are listed here:

##### 1.x.x - Guaranteed synchronization between clients of the same browser (Chrome <=> Chrome, Firefox <=> Firefox, or IE <=> IE)

* **[localsync](https://npmjs.com/packages/localsync)** - Determines synchronization mechanism and invokes it.

**Mechanisms**

* **[:bullettrain_front: storagesync](https://npmjs.com/packages/storagesync)** - Synchronizes data in a push fashion using local storage 'storage' event for a given browser.
* **[:cookie: cookiesync](https://npmjs.com/packages/cookiesync)** - Synchronizes data via cookie polling mechanism for a given browser.
* **[:computer: serversync](https://npmjs.com/packages/serversync)** - Mocks the localsync interface on server environments but does no actual synchronization (for now).

##### 2.x.x (In Progress) - The primary goal of 2.0 is to enable localsync *cross-browser* (e.g. Chrome to FireFox synchronization) for a single localsync client channel. In addition to the above mechanisms, the following mechanisms are being implemented for 2.0.

* **[:rocket: webrtcsync](https://npmjs.com/packages/webrtcsync)** - Synchronizes data across any supporting browser using WebRTC technology.
* **[:airplane: socketsync](https://npmjs.com/packages/socketsync)** - Synchronizes data across any supporting browser using web sockets technology (fallback for browsers not supporting WebRTC).


## Interface

```js
localsync(key: string, action: (...args) => payload, handler: payload => {}, [opts: Object]): { start, stop, trigger, isRunning, isFallback }
```

##### Input

**key**: a string that is used for this synchronization instance (you may have multiple instances of localsync each with different keys to sync different types of data).

**action**: a function that will be called when this client's trigger function is invoked. The action will be passed any arguments provided to the trigger function and should return the payload to be delivered to other clients for the given localsync key.

**handler**: a function that will be invoked on this client when any other client's trigger function is invoked. *NOTE: This handler will **NEVER** be called due to this clients trigger function being called, only other clients.*

**opts**: An optional object argument that may be specified to control how localsync operates. Supported values are shown in the table below.

**name**    | **type**    | **default**   | **description**
--------    | --------    | -----------   | ---------------
`tracing`   | `boolean`   | `false`       | toggles tracing for debugging purposes
`logger`    | `Object`    | `console`     | the logger object to trace to
`loglevel`  | `string`    | `'info'`      | the log level to use when tracing (`error`, `warn`, `info`, `trace`)

**IE / Edge fallback props for `cookiesync`**

**name**        | **type**      | **default**   | **description**
--------        | --------      | -----------   | ---------------
`pollFrequency` | `number`      | `3000`        | the number in milliseconds that should be used for cookie polling
`idLength`      | `number`      | `8`           | the number of characters to use for tracking the current instance (tab)
`path`          | `string`      | `'/'`         | The path to use for cookies
`secure`        | `boolean`     | `false`       | Whether to set the secure flag on cookies or not (not recommended)
`httpOnly`      | `boolean`     | `false`       | Whether to set the http only flag on cookies or not


##### Output

**Interface of returned localsync object**

**name**        | **type**      | **defaults**                    | **description**
--------        | --------      | -----------                     | ---------------
`start`         | `function`    | `N/A`                           | Call to start syncing
`stop`          | `function`    | `N/A`                           | Call to stop syncing
`trigger`       | `function`    | `N/A`                           | Call to trigger a sync to occur to all other clients
`mechanism`     | `string`      | `(storage|cookie|server)sync`   | The underlying mechanism that was selected for synchronization
`isRunning`     | `boolean`     | `false`                         | Is synchronization currently enabled
`isFallback`    | `boolean`     | `false`                         | Is the selected mechanism a fallback strategy
`isServer`      | `boolean`     | `false`                         | Is the current client running in a server environment
