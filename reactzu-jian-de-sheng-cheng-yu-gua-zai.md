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

我们可以发现，对外暴露的React组件拥有Children，Component，PureComponent，unstable\_AsyncComponent，createElement，cloneElement，createFactory，isValidElement，version

