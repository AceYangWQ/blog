# 学习 sentry 源码整体架构，打造属于自己的前端异常监控SDK

## 前言

>这是学习源码整体架构第四篇。整体架构这词语好像有点大，姑且就算是源码整体结构吧，主要就是学习是代码整体结构，不深究其他不是主线的具体函数的实现。文章学习的是打包整合后的代码，不是实际仓库中的拆分的代码。

其余三篇分别是：
>1.[学习 jQuery 源码整体架构，打造属于自己的 js 类库](https://juejin.im/post/5d39d2cbf265da1bc23fbd42)

>2.[学习 underscore 源码整体架构，打造属于自己的函数式编程类库](https://juejin.im/post/5d4bf94de51d453bb13b65dc)

>3.[学习 lodash 源码整体架构，打造属于自己的函数式编程类库](https://juejin.im/post/5d767e1d6fb9a06b032025ea)

感兴趣的读者可以点击阅读。

开发微信小程序，想着搭建小程序错误监控方案。最近用了丁香园 开源的`Sentry` 小程序 `SDK`[sentry-miniapp](https://github.com/lizhiyao/sentry-miniapp)。
顺便研究下[`sentry-javascript`](https://github.com/getsentry/sentry-javascript) 的源码整体架构，于是有了这篇文章。

本文分析的是打包后未压缩的源码，源码总行数五千余行，链接地址是：[https://browser.sentry-cdn.com/5.7.1/bundle.js](https://browser.sentry-cdn.com/5.7.1/bundle.js)， 版本是`v5.7.1`。

本文示例等源代码在这个[github blog](https://github.com/lxchuan12/blog/docs/sentry)，需要的读者可以点击查看，觉得不错，可以顺便`star`一下。

看源码前先来梳理下前端错误监控的知识。

## 前端错误监控知识

摘抄自某视频教程。

### 前端错误的分类

>**1.即时运行错误：代码错误**

`try...catch`

`window.onerror` (也可以用`DOM2`事件监听)

>**2.资源加载错误**

 `object.onerror`: `dom`对象的`onerror`事件

`performance.getEntries()`

`Error`事件捕获

>3.**使用`performance.getEntries()`获取网页图片加载错误**

`var allImgs = document.getElementsByTagName('image')`

`var loadedImgs = performance.getEntries().filter(i => i.initiatorType === 'img')`

最后`allIms`和`loadedImgs`对比即可找出图片资源未加载项目

### Error事件捕获代码示例

```js
window.addEventListener('error', function(e) {
  console.log('捕获', e)
}, true) // 这里只有捕获才能触发事件，冒泡是不能触发
```

### 上报错误的基本原理

1.采用`Ajax`通信的方式上报

2.利用`Image`对象上报 (主流方式)

`Image`上报错误方式：
` (new Image()).src = 'https://lxchuan12.cn/error?name=若川'`

## Sentry 前端异常监控原理

>1.重写 `window.onerror` 方法

```js
window.onerror = function (msg, url, line, column, error) {
	console.log('msg, url, line, column, error', msg, url, line, column, error);
}
```
`Sentry` 源码可以搜索 `global.onerror` 定位到具体位置
```js
 GlobalHandlers.prototype._installGlobalOnErrorHandler = function () {
            if (this._onErrorHandlerInstalled) {
                return;
            }
			var self = this; // tslint:disable-line:no-this-assignment

			// 这里的 this._global 在浏览器中就是 window
            this._oldOnErrorHandler = this._global.onerror;
			this._global.onerror = function (msg, url, line, column, error) {}
			// code ...
 }
```

>2.采用`Ajax`上传

支持 `fetch` 使用 `fetch`，否则使用　`XHR`。

```js
BrowserBackend.prototype._setupTransport = function () {
	// code ...
	if (supportsFetch()) {
		return new FetchTransport(transportOptions);
	}
	return new XHRTransport(transportOptions);
};
```

>2.1 `fetch`

```js
FetchTransport.prototype.sendEvent = function (event) {
	var defaultOptions = {
		body: JSON.stringify(event),
		method: 'POST',
		referrerPolicy: (supportsReferrerPolicy() ? 'origin' : ''),
	};
	return this._buffer.add(global$2.fetch(this.url, defaultOptions).then(function (response) { return ({
		status: exports.Status.fromHttpCode(response.status),
	}); }));
};
```

>2.2 `XMLHttpRequest`

```js
 XHRTransport.prototype.sendEvent = function (event) {
	 var _this = this;
	return this._buffer.add(new SyncPromise(function (resolve, reject) {
		var request = new XMLHttpRequest();
		// code ...
	};
 }
```

本文主要通过如何`Ajax上报`和`window.onerror`两条主线来学习源码。

## Sentry 源码入口和出口

```js
var Sentry = (function(exports){
	// code ...

	// code ...
	// 省略了导出的Sentry的若干个方法和属性
	// 只列出了如下两个
	// 重点关注 captureMessage
    exports.captureMessage = captureMessage;
	// 重点关注 init
    exports.init = init;

    return exports;
}({}));
```

## Sentry.init 初始化 init 函数

初始化

```js
Sentry.init({ dsn: 'https://34e29c93475c43a58bee89a5530f2b9f@sentry.io/1784519' });
```

```js
// options 是 {dsn: '...'}
function init(options) {
	// 如果options 是undefined，则赋值为 空对象
	if (options === void 0) { options = {}; }
	// 如果没传 defaultIntegrations 则赋值默认的
	if (options.defaultIntegrations === undefined) {
		options.defaultIntegrations = defaultIntegrations;
	}
	// 初始化语句
	if (options.release === undefined) {
		var window_1 = getGlobalObject();
		// 这是给  sentry-webpack-plugin 插件提供的，webpack插件注入的变量。这里没用这个插件，所以这里不深究。
		// This supports the variable that sentry-webpack-plugin injects
		if (window_1.SENTRY_RELEASE && window_1.SENTRY_RELEASE.id) {
			options.release = window_1.SENTRY_RELEASE.id;
		}
	}
	// 初始化并且绑定
	initAndBind(BrowserClient, options);
}
```

### getGlobalObject、inNodeEnv 函数

很多地方用到这个函数，解释一下，其实做的事情也比较简单，就是获取全局对象。浏览器中是`window`。

```js
/**
 	* 判断是否是node环境
	* Checks whether we're in the Node.js or Browser environment
	*
	* @returns Answer to given question
	*/
function isNodeEnv() {
	// tslint:disable:strict-type-predicates
	return Object.prototype.toString.call(typeof process !== 'undefined' ? process : 0) === '[object process]';
}
var fallbackGlobalObject = {};
/**
	* Safely get global scope object
	*
	* @returns Global scope object
	*/
function getGlobalObject() {
	return (isNodeEnv()
	// 是 node 环境 赋值给 global
		? global
		: typeof window !== 'undefined'
			? window
			// 不是 window self 不是undefined 说明是 web Socket 环境
			: typeof self !== 'undefined'
				? self
				// 都不是，赋值给空对象。
				: fallbackGlobalObject);
```

继续看 `initAndBind` 函数

## initAndBind 函数 之 new BrowserClient(options)

```js
function initAndBind(clientClass, options) {
	//
	if (options.debug === true) {
		logger.enable();
	}
	getCurrentHub().bindClient(new clientClass(options));
}
```

可以看出 initAndBind()，第一个参数是 `BrowserClient` 构造函数，第二个参数是初始化后的`options`
接着先看 构造函数 `BrowserClient`。
TODO: 另一条线 `getCurrentHub().bindClient()` 先不看。

### BrowserClient 构造函数

```js
var BrowserClient = /** @class */ (function (_super) {
	__extends(BrowserClient, _super);
	/**
	 * Creates a new Browser SDK instance.
	 *
	 * @param options Configuration options for this SDK.
	 */
	function BrowserClient(options) {
		if (options === void 0) { options = {}; }
		return _super.call(this, BrowserBackend, options) || this;
	}
	return BrowserClient;
}(BaseClient));
```

```js
__extends(BrowserClient, _super);
function BrowserClient(options) {
	if (options === void 0) { options = {}; }
	return _super.call(this, BrowserBackend, options) || this;
}
```
`从代码中可以看出`
`BrowserClient` 继承至`BaseClient`，并且把`BrowserBackend`，`options`传参给`BaseClient`调用。

先看 `BrowserBackend`
TODO: 这里的`BaseClient`，暂时不看。

### __extends、extendStatics

这里提一下继承、继承静态属性和方法。

未打包的源码是使用`ES6 extends`实现的。这是打包后的对`ES6`的`extends`的一种实现。

如果对继承还不是很熟悉的读者，可以参考我之前写的文章。[面试官问：JS的继承](https://lxchuan12.github.io/js-extend/)

```js
var extendStatics = function(d, b) {
	extendStatics = Object.setPrototypeOf ||
		({ __proto__: [] } instanceof Array && function (d, b) { d.__proto__ = b; }) ||
		function (d, b) { for (var p in b) if (b.hasOwnProperty(p)) d[p] = b[p]; };
	return extendStatics(d, b);
};

function __extends(d, b) {
	extendStatics(d, b);
	function __() { this.constructor = d; }
	d.prototype = b === null ? Object.create(b) : (__.prototype = b.prototype, new __());
}
```

### BrowserBackend  构造函数 浏览器后端

```js
var BrowserBackend = /** @class */ (function (_super) {
    __extends(BrowserBackend, _super);
	function BrowserBackend() {
		return _super !== null && _super.apply(this, arguments) || this;
	}
	/**
	 * @inheritDoc
	 */
	BrowserBackend.prototype._setupTransport = function () {
		if (!this._options.dsn) {
			// We return the noop transport here in case there is no Dsn.
			return _super.prototype._setupTransport.call(this);
		}
		var transportOptions = __assign({}, this._options.transportOptions, { dsn: this._options.dsn });
		if (this._options.transport) {
			return new this._options.transport(transportOptions);
		}
		if (supportsFetch()) {
			return new FetchTransport(transportOptions);
		}
		return new XHRTransport(transportOptions);
	};
	// code ...
	return BrowserBackend;
}(BaseBackend));
```

#### BaseBackend  构造函数 （基础后端）

```js
/**
 * This is the base implemention of a Backend.
 * @hidden
 */
var BaseBackend = /** @class */ (function () {
	/** Creates a new backend instance. */
	function BaseBackend(options) {
		this._options = options;
		if (!this._options.dsn) {
			logger.warn('No DSN provided, backend will not do anything.');
		}
		this._transport = this._setupTransport();
	}
	/**
	 * Sets up the transport so it can be used later to send requests.
	 */
	BaseBackend.prototype._setupTransport = function () {
		return new NoopTransport();
	};
	/**
	 * @inheritDoc
	 */
	BaseBackend.prototype.eventFromException = function (_exception, _hint) {
		throw new SentryError('Backend has to implement `eventFromException` method');
	};
	/**
	 * @inheritDoc
	 */
	BaseBackend.prototype.eventFromMessage = function (_message, _level, _hint) {
		throw new SentryError('Backend has to implement `eventFromMessage` method');
	};
	/**
	 * @inheritDoc
	 */
	BaseBackend.prototype.sendEvent = function (event) {
		this._transport.sendEvent(event).then(null, function (reason) {
			logger.error("Error while sending event: " + reason);
		});
	};
	/**
	 * @inheritDoc
	 */
	BaseBackend.prototype.getTransport = function () {
		return this._transport;
	};
	return BaseBackend;
}());
```


#### BaseClient class

```js
var BaseClient = /** @class */ (function () {
	/**
	 * Initializes this client instance.
	 *
	 * @param backendClass A constructor function to create the backend.
	 * @param options Options for the client.
	 */
	function BaseClient(backendClass, options) {
		/** Array of used integrations. */
		this._integrations = {};
		/** Is the client still processing a call? */
		this._processing = false;
		this._backend = new backendClass(options);
		this._options = options;
		if (options.dsn) {
			this._dsn = new Dsn(options.dsn);
		}
		if (this._isEnabled()) {
			this._integrations = setupIntegrations(this._options);
		}
	}
	// code ...
	return BaseClient;
}());
```

### new BrowerClient 经过一系列的继承和初始化

最终得到这样的数据

![new BrowserClient(options)](./images/sentry-new-BrowserClient(options).png)

## initAndBind 函数 之 getCurrentHub().bindClient()

继续看initAndBind 的另一条线

```js
function initAndBind(clientClass, options) {
	if (options.debug === true) {
		logger.enable();
	}
	getCurrentHub().bindClient(new clientClass(options));
}
```

获取当前的控制中心 `Hub`，bindClient。

### getCurrentHub()

```js
function getCurrentHub() {
	// Get main carrier (global for every environment)
	var registry = getMainCarrier();
	// If there's no hub, or its an old API, assign a new one
	if (!hasHubOnCarrier(registry) || getHubFromCarrier(registry).isOlderThan(API_VERSION)) {
		setHubOnCarrier(registry, new Hub());
	}
	// node 才执行
	// Prefer domains over global if they are there (applicable only to Node environment)
	if (isNodeEnv()) {
		return getHubFromActiveDomain(registry);
	}
	// Return hub that lives on a global object
	return getHubFromCarrier(registry);
}
```

### getMainCarrier、getHubFromCarrier

<!-- 获取主载体 -->
```js
function getMainCarrier() {
	// 载体 这里是window
	// 通过一系列new BrowerClient() 一系列的初始化
	// 挂载在  carrier.__SENTRY__ 已经有了三个属性，globalEventProcessors, hub, logger
	var carrier = getGlobalObject();
	carrier.__SENTRY__ = carrier.__SENTRY__ || {
		hub: undefined,
	};
	return carrier;
}
```


```js
// 获取控制中心 hub 从载体上
function getHubFromCarrier(carrier) {
	// 已经有了则返回，没有则new Hub
	if (carrier && carrier.__SENTRY__ && carrier.__SENTRY__.hub) {
		return carrier.__SENTRY__.hub;
	}
	carrier.__SENTRY__ = carrier.__SENTRY__ || {};
	carrier.__SENTRY__.hub = new Hub();
	return carrier.__SENTRY__.hub;
}
```

### bindClient

获取最后一个

```js
Hub.prototype.bindClient = function (client) {
	var top = this.getStackTop();
	top.client = client;
};
```

```js
Hub.prototype.getStackTop = function () {
	return this._stack[this._stack.length - 1];
};
```

### 经过一系列的继承和初始化

最终得到这样的数据

```js
function initAndBind(clientClass, options) {
	if (options.debug === true) {
		logger.enable();
	}
	getCurrentHub().bindClient(new clientClass(options));
}
```

### hub

```js
{
_stack: Array(1)
0:
client: BrowserClient {_integrations: {…}, _processing: false, _backend: BrowserBackend, _options: {…}, _dsn: Dsn}
scope: Scope {_notifyingListeners: false, _scopeListeners: Array(0), _eventProcessors: Array(0), _breadcrumbs: Array(0), _user: {…}, …}
__proto__: Object
length: 1
__proto__: Array(0)
_version: 3
__proto__:
		addBreadcrumb: ƒ (breadcrumb, hint)
		bindClient: ƒ (client)
		captureEvent: ƒ (event, hint)
		captureException: ƒ (exception, hint)
		captureMessage: ƒ (message, level, hint)
		configureScope: ƒ (callback)
		getClient: ƒ ()
		getIntegration: ƒ (integration)
		getScope: ƒ ()
		getStack: ƒ ()
		getStackTop: ƒ ()
		isOlderThan: ƒ (version)
		lastEventId: ƒ ()
		popScope: ƒ ()
		pushScope: ƒ ()
		run: ƒ (callback)
		setContext: ƒ (name, context)
		setExtra: ƒ (key, extra)
		setExtras: ƒ (extras)
		setTag: ƒ (key, value)
		setTags: ƒ (tags)
		setUser: ƒ (user)
		traceHeaders: ƒ ()
		withScope: ƒ (callback)
		_invokeClient: ƒ (method)
		constructor: ƒ Hub(client, scope, _version)
		__proto__: Object
}
```

初始化完成后，再来看例子
具体 `captureMessage` 函数。

```js
Sentry.captureMessage('Hello, 若川!');
```

## captureMessage 函数


```js
function captureMessage(message, level) {
	var syntheticException;
	try {
		throw new Error(message);
	}
	catch (exception) {
		syntheticException = exception;
	}
	return callOnHub('captureMessage', message, level, {
		originalException: message,
		syntheticException: syntheticException,
	});
}
```

```js
/**
 * This calls a function on the current hub.
 * @param method function to call on hub.
 * @param args to pass to function.
 */
function callOnHub(method) {
	var args = [];
	for (var _i = 1; _i < arguments.length; _i++) {
		args[_i - 1] = arguments[_i];
	}
	var hub = getCurrentHub();
	if (hub && hub[method]) {
		// tslint:disable-next-line:no-unsafe-any
		return hub[method].apply(hub, __spread(args));
	}
	throw new Error("No hub defined or " + method + " was not found on the hub, please open a bug report.");
}
```

```js
/**
 * @inheritDoc
 */
Hub.prototype.captureMessage = function (message, level, hint) {
	var eventId = (this._lastEventId = uuid4());
	var finalHint = hint;
	// If there's no explicit hint provided, mimick the same thing that would happen
	// in the minimal itself to create a consistent behavior.
	// We don't do this in the client, as it's the lowest level API, and doing this,
	// would prevent user from having full control over direct calls.
	if (!hint) {
		var syntheticException = void 0;
		try {
			throw new Error(message);
		}
		catch (exception) {
			syntheticException = exception;
		}
		finalHint = {
			originalException: message,
			syntheticException: syntheticException,
		};
	}
	this._invokeClient('captureMessage', message, level, __assign({}, finalHint, { event_id: eventId }));
	return eventId;
};
```

```js
/**
 * Internal helper function to call a method on the top client if it exists.
 *
 * @param method The method to call on the client.
 * @param args Arguments to pass to the client function.
 */
Hub.prototype._invokeClient = function (method) {
	var _a;
	var args = [];
	for (var _i = 1; _i < arguments.length; _i++) {
		args[_i - 1] = arguments[_i];
	}
	var top = this.getStackTop();
	if (top && top.client && top.client[method]) {
		(_a = top.client)[method].apply(_a, __spread(args, [top.scope]));
	}
};
```


### BaseClient.prototype.captureMessage

### BaseClient.prototype._processEvent

### _this._getBackend().sendEvent(finalEvent);

### BaseBackend.prototype.sendEvent

```js
/**
	* @inheritDoc
	*/
BaseBackend.prototype.sendEvent = function (event) {
	this._transport.sendEvent(event).then(null, function (reason) {
		logger.error("Error while sending event: " + reason);
	});
};
```

### FetchTransport.prototype.sendEvent

```js
/**
	* @inheritDoc
	*/
FetchTransport.prototype.sendEvent = function (event) {
	var defaultOptions = {
		body: JSON.stringify(event),
		method: 'POST',
		// Despite all stars in the sky saying that Edge supports old draft syntax, aka 'never', 'always', 'origin' and 'default
		// https://caniuse.com/#feat=referrer-policy
		// It doesn't. And it throw exception instead of ignoring this parameter...
		// REF: https://github.com/getsentry/raven-js/issues/1233
		referrerPolicy: (supportsReferrerPolicy() ? 'origin' : ''),
	};
	return this._buffer.add(global$2.fetch(this.url, defaultOptions).then(function (response) { return ({
		status: exports.Status.fromHttpCode(response.status),
	}); }));
};
```

### PromiseBuffer

TODO:
可不写

看完 `Ajax 上报` 主线，再看本文的另外一条主线 `window.onerror` 捕获。

## window.onerror 捕获 错误

例子：调用一个未申明的变量。

```js
func();
```

### captureEvent

```js
/** JSDoc */
GlobalHandlers.prototype._installGlobalOnErrorHandler = function () {
	if (this._onErrorHandlerInstalled) {
		return;
	}
	var self = this; // tslint:disable-line:no-this-assignment
	this._oldOnErrorHandler = this._global.onerror;
	this._global.onerror = function (msg, url, line, column, error) {
		var currentHub = getCurrentHub();
		var hasIntegration = currentHub.getIntegration(GlobalHandlers);
		var isFailedOwnDelivery = error && error.__sentry_own_request__ === true;
		if (!hasIntegration || shouldIgnoreOnError() || isFailedOwnDelivery) {
			if (self._oldOnErrorHandler) {
				return self._oldOnErrorHandler.apply(this, arguments);
			}
			return false;
		}
		var client = currentHub.getClient();
		var event = isPrimitive(error)
			? self._eventFromIncompleteOnError(msg, url, line, column)
			: self._enhanceEventWithInitialFrame(eventFromUnknownInput(error, undefined, {
				attachStacktrace: client && client.getOptions().attachStacktrace,
				rejection: false,
			}), url, line, column);
		addExceptionMechanism(event, {
			handled: false,
			type: 'onerror',
		});
		currentHub.captureEvent(event, {
			originalException: error,
		});
		if (self._oldOnErrorHandler) {
			return self._oldOnErrorHandler.apply(this, arguments);
		}
		return false;
	};
	this._onErrorHandlerInstalled = true;
};
```

BaseClient.prototype.captureEvent

BaseClient.prototype._processEvent

_this._getBackend().sendEvent(finalEvent);


```js
BaseBackend.prototype.sendEvent = function (event) {
		this._transport.sendEvent(event).then(null, function (reason) {
			logger.error("Error while sending event: " + reason);
		});
};
```

```js
FetchTransport.prototype.sendEvent = function (event) {
	var defaultOptions = {
		body: JSON.stringify(event),
		method: 'POST',
		// Despite all stars in the sky saying that Edge supports old draft syntax, aka 'never', 'always', 'origin' and 'default
		// https://caniuse.com/#feat=referrer-policy
		// It doesn't. And it throw exception instead of ignoring this parameter...
		// REF: https://github.com/getsentry/raven-js/issues/1233
		referrerPolicy: (supportsReferrerPolicy() ? 'origin' : ''),
	};
	return this._buffer.add(global$2.fetch(this.url, defaultOptions).then(function (response) { return ({
		status: exports.Status.fromHttpCode(response.status),
	}); }));
};
```

![new BrowserClient(options)](./images/sentry-new-BrowserClient(options).png)

## 总结

可谓是惊艳。

## 推荐阅读

[JavaScript集成Sentry](https://juejin.im/post/5b7f63c96fb9a019f709b14b)

未完待续 ...

TODO:
