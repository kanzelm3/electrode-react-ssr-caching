# electrode-react-ssr-caching [![NPM version][npm-image]][npm-url] [![Build Status][travis-image]][travis-url] [![Dependency Status][daviddm-image]][daviddm-url]

Support profiling React Server Side Rendering time and component caching to help you speed up SSR.

# Installing

```
npm i electrode-react-ssr-caching
```

# Usage

Note that since this module patches React's source code to inject the caching logic, it must be loaded before the React module.

For example:

```js
import SSRCaching from "electrode-react-ssr-caching";
import React from 'react';
import ReactDOM from 'react-dom/server';
```


## Profiling

You can use this module to inspect the time each component took to render.

```js
import SSRCaching from "electrode-react-ssr-caching";
import { renderToString } from "react-dom/server";
import MyComponent from "mycomponent";

// First you should render your component in a loop to prime the JS engine (i.e: V8 for NodeJS)
for( let i = 0; i < 10; i ++ ) {
    renderToString(<MyComponent />);
}

SSRCaching.clearProfileData();
SSRCaching.enableProfiling();
const html = renderToString(<MyComponent />);
SSRCaching.enableProfiling(false);
console.log(JSON.stringify(SSRCaching.profileData, null, 2));
```

## Caching

Once you determined the most expensive components with profiling, you can enable component caching this module provides to speed up SSR performance.

The basic steps to enabling caching are:

```js
import SSRCaching from "electrode-react-ssr-caching";

SSRCaching.enableCaching();
SSRCaching.setCachingConfig(cacheConfig);
```

Where `cacheConfig` contains information on what component to apply caching.  See below for details.

In order for the `enableCaching()` method to work, you'll also need `NODE_ENV` set to `production`, or else it will throw an error.

### cacheConfig

SSR component caching was first demonstrated in [Sasha Aickin's talk].

His demo requires each component to provide a function for generating the cache key.

Here we implemented two cache key generation strategies: `simple` and `template`.

You are required to pass in the `cacheConfig` to tell this module what component to apply caching.

For example:

```js
const cacheConfig = {
    components: {
        "Component1": {
            strategy: "simple",
            enable: true
        },
        "Component2": {
            strategy: "template",
            enable: true
        }
    }
}

SSRCaching.setCachingConfig(cacheConfig);
```

### Caching Strategies

#### simple

The `simple` caching strategy is basically doing a `JSON.stringify` on the component's props.  You can also specify a callback in `cacheConfig` to return the key.

For example:

```js
const cacheConfig = {
    components: {
        Component1: {
            strategy: "simple",
            enable: true,
            genCacheKey: (props) => JSON.stringify(props)
        }
    }
};
```

This strategy is not very flexible.  You need a cache entry for each different props.  However it requires very little processing time.

#### template

The `template` caching strategy is more complex but flexible.  

The idea is akin to generating logic-less handlebars template from your React components and then use string replace to process the template with different props.

If you have this component:

```js
class Hello extends Component {
    render() {
        return <div>Hello, {this.props.name}.  {this.props.message}</div>
    }
}
```

And you render it with props:
```js
const props = { name: "Bob", message: "How're you?" }
```

You get back HTML string:
```html
<div>Hello, <span>Bob</span>.  <span>How&#x27;re you?</span></div>
```

Now if you replace values in props with tokens, and you remember that `@0@` refers to `props.name` and `@1@` refers to `props.message`:
```js
const tokenProps = { name: "@0@", message: "@1@" }
```

You get back HTML string that could be akin to a handlebars template:
```html
<div>Hello, <span>@0@</span>.  <span>@1@</span></div>
```

We cache this template html using the tokenized props as cache key.  When we need to render the same component with a different props later, we can just lookup the template from cache and use string replace to apply the values:
```js
cachedTemplateHtml.replace( /@0@/g, props.name ).replace( /@1@/g, props.message );
```

That's the gist of the template strategy.  Of course there are many small details such as handling the encoding of special characters, preserving props that can't be tokenized, avoid tokenizing non-string props, or preserving `data-reactid` and `data-react-checksum`.

To specify a component to be cached with the `template` strategy:

```js
const cacheConfig = {
    components: {
        Hello: {
            strategy: "template",
            enable: true,
            preserveKeys: [ "key1", "key2" ],
            preserveEmptyKeys: [ "key3", "key4" ],
            ignoreKeys: [ "key5", "key6" ],
            whiteListNonStringKeys: [ "key7", "key8" ]
        }
    }
};
```

   - `preserveKeys` - List of keys that should not be tokenized.
   - `preserveEmptyKeys` - List of keys that should not be tokenized if they are empty string `""`
   - `ignoreKeys` - List of keys that should be completely ignored as part of the template cache key.
   - `whiteListNonStringKeys` - List of non-string keys that should be tokenized.

# API

### [`enableProfiling(flag)`](#enableprofilingflag)

Enable profiling according to flag

   - `undefined` or `true` -  enable profiling
   - `false` - disable profiling

### [`enableCaching(flag)`](#enablecachingflag)

Enable cache according to flag

   - `undefined` or `true` - enable caching
   - `false` - disable caching

### [`enableCachingDebug(flag)`](#enablecachingdebugflag)

Enable cache debugging according to flag.

> Caching must be enabled for this to have any effect.

   - `undefined` or `true` - enable cache debugging
   - `false` - disable cache debugging

### [`setCachingConfig(config)`](#setcachingconfigconfig)

Set caching config to `config`.

### [`stripUrlProtocol(flag)`](#stripurlprotocolflag)

Remove `http:` or `https:` from prop values that are URLs according to flag.

> Caching must be enabled for this to have any effect.

   - `undefined` or `true` - strip URL protocol
   - `false` - don't strip

### [`shouldHashKeys(flag, [hashFn])`](#shouldhashkeysflaghashfn)

Set whether the `template` strategy should hash the cache key and use that instead.

> Caching must be enabled for this to have any effect.

  - `flag`
    - `undefined` or `true` - use a hash value of the cache key
    - `false` - don't use a hash valueo f the cache key
  - `hashFn` - optional, a custom callback to generate the hash from the cache key, which is passed in as a string
    - i.e. `function customHashFn(key) { return hash(key); }`

If no `hashFn` is provided, then [farmhash] is used if it's available, otherwise hashing is turned off.

### [`clearProfileData()`](#clearprofiledata)

Clear profiling data

### [`clearCache()`](#clearcache)

Clear caching data

### [`cacheEntries()`](#cacheentries)

Get total number of cache entries

### [`cacheHitReport()`](#cachehitreport)

Returns an object with information about cache entry hits

Built with :heart: by [Team Electrode](https://github.com/orgs/electrode-io/people) @WalmartLabs.

[Sasha Aickin's talk]: https://www.youtube.com/watch?v=PnpfGy7q96U
[farmhash]: https://github.com/google/farmhash
[npm-image]: https://badge.fury.io/js/electrode-react-ssr-caching.svg
[npm-url]: https://npmjs.org/package/electrode-react-ssr-caching
[travis-image]: https://travis-ci.org/electrode-io/electrode-react-ssr-caching.svg?branch=master
[travis-url]: https://travis-ci.org/electrode-io/electrode-react-ssr-caching
[daviddm-image]: https://david-dm.org/electrode-io/electrode-react-ssr-caching.svg?theme=shields.io
[daviddm-url]: https://david-dm.org/electrode-io/electrode-react-ssr-caching
