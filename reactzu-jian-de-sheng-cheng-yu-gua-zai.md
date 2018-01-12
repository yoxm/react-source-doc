## React组件的构成

### 一、React组件是什么样的

先看下面的这段代码

```
import React, { Component} from 'react';

class App extends Component {
  constructor(props) {
    super(props);
  }

  render() {
    return (
      <div>
        我是一个React组件
      </div>

    );
  }
}

export default App;
```

这样，一个简单通常的React App组件就写好了，但是这个App组件是什么样的呢？没事，我们有万能的

console.log\(\);打印一下看看！

![](/assets/App.png)可以发现，我们的&lt;App /&gt;组件不是一个DOM，而是一个JS对象，拥有以上所示的属性，这时我们疑惑了，为什么不是DOM呢，我们再打印下&lt;App&gt;&lt;div&gt;我是子组件&lt;/div&gt;&lt;/App&gt;，看看会输出什么呢?![](/assets/import.png) 我们发现上面最明显的变化无疑是prop属性，在&lt;App /&gt;组件中嵌套了一个div,div中还有文字，所以props对象增加了一个children属性，这时候我们明白了，原来我们的组件嵌套，就是在其props对象下的children属性增加对应的值，这就是React组件的嵌套。

但是，我们的JSX语法，也就是&lt;App /&gt;这种形式的的对象是怎么构成的呢？

在react的github上，packages/events/react/src/React.js文件如下

```
/**
 * Copyright (c) 2013-present, Facebook, Inc.
 *
 * This source code is licensed under the MIT license found in the
 * LICENSE file in the root directory of this source tree.
 */

import assign from 'object-assign';
import ReactVersion from 'shared/ReactVersion';
import {REACT_FRAGMENT_TYPE} from 'shared/ReactSymbols';

import {Component, PureComponent, AsyncComponent} from './ReactBaseClasses';
import {forEach, map, count, toArray, only} from './ReactChildren';
import ReactCurrentOwner from './ReactCurrentOwner';
import {
  createElement,
  createFactory,
  cloneElement,
  isValidElement,
} from './ReactElement';
import {
  createElementWithValidation,
  createFactoryWithValidation,
  cloneElementWithValidation,
} from './ReactElementValidator';
import ReactDebugCurrentFrame from './ReactDebugCurrentFrame';

var React = {
  Children: {
    map,
    forEach,
    count,
    toArray,
    only,
  },

  Component,
  PureComponent,
  unstable_AsyncComponent: AsyncComponent,

  Fragment: REACT_FRAGMENT_TYPE,

  createElement: __DEV__ ? createElementWithValidation : createElement,
  cloneElement: __DEV__ ? cloneElementWithValidation : cloneElement,
  createFactory: __DEV__ ? createFactoryWithValidation : createFactory,
  isValidElement: isValidElement,

  version: ReactVersion,

  __SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED: {
    ReactCurrentOwner,
    // Used by renderers to avoid bundling object-assign twice in UMD bundles:
    assign,
  },
};

if (__DEV__) {
  Object.assign(React.__SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED, {
    // These should not be included in production.
    ReactDebugCurrentFrame,
    // Shim for React DOM 16.0.0 which still destructured (but not used) this.
    // TODO: remove in React 17.0.
    ReactComponentTreeHook: {},
  });
}

export default React;
```

发现React对外暴露的React对象拥有Children， Component， PureComponent， unstable\_AsyncComponent，Fragment，createElement，cloneElement，createFactory，isValidElement及\_\_SECRET\_INTERNALS\_DO\_NOT\_USE\_OR\_YOU\_WILL\_BE\_FIRED这几个方法，其中的Component就是我们最常使用的方法，PureComponent是傀儡（受控\)组件的实现，顺着上面import的ReactBaseClasses来看看这个文件里有什么。

```
import emptyObject from 'fbjs/lib/emptyObject';
import invariant from 'fbjs/lib/invariant';
import lowPriorityWarning from 'shared/lowPriorityWarning';

import ReactNoopUpdateQueue from './ReactNoopUpdateQueue';


function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}

Component.prototype.isReactComponent = {};


Component.prototype.setState = function(partialState, callback) {
  invariant(
    typeof partialState === 'object' ||
      typeof partialState === 'function' ||
      partialState == null,
    'setState(...): takes an object of state variables to update or a ' +
      'function which returns an object of state variables.',
  );
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};


Component.prototype.forceUpdate = function(callback) {
  this.updater.enqueueForceUpdate(this, callback, 'forceUpdate');
};

if (__DEV__) {
  const deprecatedAPIs = {
    isMounted: [
      'isMounted',
      'Instead, make sure to clean up subscriptions and pending requests in ' +
        'componentWillUnmount to prevent memory leaks.',
    ],
    replaceState: [
      'replaceState',
      'Refactor your code to use setState instead (see ' +
        'https://github.com/facebook/react/issues/3236).',
    ],
  };
  const defineDeprecationWarning = function(methodName, info) {
    Object.defineProperty(Component.prototype, methodName, {
      get: function() {
        lowPriorityWarning(
          false,
          '%s(...) is deprecated in plain JavaScript React classes. %s',
          info[0],
          info[1],
        );
        return undefined;
      },
    });
  };
  for (const fnName in deprecatedAPIs) {
    if (deprecatedAPIs.hasOwnProperty(fnName)) {
      defineDeprecationWarning(fnName, deprecatedAPIs[fnName]);
    }
  }
}


function PureComponent(props, context, updater) {
  // Duplicated from Component.
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}

function ComponentDummy() {}
ComponentDummy.prototype = Component.prototype;
const pureComponentPrototype = (PureComponent.prototype = new ComponentDummy());
pureComponentPrototype.constructor = PureComponent;
// Avoid an extra prototype jump for these methods.
Object.assign(pureComponentPrototype, Component.prototype);
pureComponentPrototype.isPureReactComponent = true;

function AsyncComponent(props, context, updater) {
  // Duplicated from Component.
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}

const asyncComponentPrototype = (AsyncComponent.prototype = new ComponentDummy());
asyncComponentPrototype.constructor = AsyncComponent;

Object.assign(asyncComponentPrototype, Component.prototype);
asyncComponentPrototype.unstable_isAsyncReactComponent = true;
asyncComponentPrototype.render = function() {
  return this.props.children;
};

export {Component, PureComponent, AsyncComponent};
```

为方便观看起见，删除了部分注释，我们可以看到Component就是一个构造函数，内部的props，refs，context，updater属性，以及setState，forceUpdate，并且，我们应该注意到的是，setState是接受两个参数的，第一个是整个应用部分的state，第二个是回调函数。其他的PureComponent，asynComponent同理。这样，我们的&lt;App /&gt;就有了最开始的样子。

## 二、React组件是怎么挂载的

生成了React组件后，便可以在其内部自定义方法，并且使用React自带的生命周期函数，但是他是怎么挂载到真实的DOM树上的呢？

我们可以吧&lt;App /&gt;组件的代码放到Babel中看一下。

```
var App = function (_Component) {
  _inherits(App, _Component);

  function App() {
    _classCallCheck(this, App);

    return _possibleConstructorReturn(this, (App.__proto__ || Object.getPrototypeOf(App)).apply(this, arguments));
  }

  _createClass(App, [{
    key: 'render',
    value: function render() {
      return _react2.default.createElement(
        'p',
        null,
        '\u6211\u662F\u4E00\u4E2AReact\u7EC4\u4EF6'
      );
    }
  }]);

  return App;
}(_react.Component);

exports.default = App;
```



