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

在node\_module/react/umd/react.development.js中相关代码如下

```
var React = {
  Children: {
    map: mapChildren,
    forEach: forEachChildren,
    count: countChildren,
    toArray: toArray,
    only: onlyChild
  },

  Component: Component,
  PureComponent: PureComponent,
  unstable_AsyncComponent: AsyncComponent,

  Fragment: REACT_FRAGMENT_TYPE,

  createElement: createElementWithValidation,
  cloneElement: cloneElementWithValidation,
  createFactory: createFactoryWithValidation,
  isValidElement: isValidElement,

  version: ReactVersion,

  __SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED: {
    ReactCurrentOwner: ReactCurrentOwner,
    // Used by renderers to avoid bundling object-assign twice in UMD bundles:
    assign: objectAssign
  }
};
```

发现React对外暴露的React对象拥有Children， Component， PureComponent， unstable\_AsyncComponent，Fragment，createElement，cloneElement，createFactory，isValidElement及\_\_SECRET\_INTERNALS\_DO\_NOT\_USE\_OR\_YOU\_WILL\_BE\_FIRED这几个方法，其中的Component就是我们最常使用的方法，PureComponent是傀儡（受控\)组件的实现，顺着上面import的ReactBaseClasses来看看这个文件里有什么。

```
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

## 二、React组件是怎么初始化的

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

其中的inherits是es6 extends的实现，表示继承，我们可以关注下createClass 和 createElement,我们可以打开与React.js同级目录下看一下。

```
    function createElement(type, config, children) {
        var propName;

        // Reserved names are extracted
        var props = {};

        var key = null;
        var ref = null;
        var self = null;
        var source = null;

        if (config != null) {
            if (hasValidRef(config)) {
                ref = config.ref;
            }
            if (hasValidKey(config)) {
                key = '' + config.key;
            }

            self = config.__self === undefined ? null : config.__self;
            source = config.__source === undefined ? null : config.__source;
            // Remaining properties are added to a new props object
            for (propName in config) {
                if (hasOwnProperty$1.call(config, propName) && !RESERVED_PROPS.hasOwnProperty(propName)) {
                    props[propName] = config[propName];
                }
            }
        }

        // Children can be more than one argument, and those are transferred onto
        // the newly allocated props object.
        var childrenLength = arguments.length - 2;
        if (childrenLength === 1) {
            props.children = children;
        } else if (childrenLength > 1) {
            var childArray = Array(childrenLength);
            for (var i = 0; i < childrenLength; i++) {
                childArray[i] = arguments[i + 2];
            }
            {
                if (Object.freeze) {
                    Object.freeze(childArray);
                }
            }
            props.children = childArray;
        }

        // Resolve default props
        if (type && type.defaultProps) {
            var defaultProps = type.defaultProps;
            for (propName in defaultProps) {
                if (props[propName] === undefined) {
                    props[propName] = defaultProps[propName];
                }
            }
        }
        {
            if (key || ref) {
                if (typeof props.$$typeof === 'undefined' || props.$$typeof !== REACT_ELEMENT_TYPE) {
                    var displayName = typeof type === 'function' ? type.displayName || type.name || 'Unknown' : type;
                    if (key) {
                        defineKeyPropWarningGetter(props, displayName);
                    }
                    if (ref) {
                        defineRefPropWarningGetter(props, displayName);
                    }
                }
            }
        }
        return ReactElement(type, key, ref, self, source, ReactCurrentOwner.current, props);
    }
    
    ...
    function createElementWithValidation(type, props, children) {
        var validType = typeof type === 'string' || typeof type === 'function' || typeof type === 'symbol' || typeof type === 'number';
        // We warn in this case but don't throw. We expect the element creation to
        // succeed and there will likely be errors in render.
        if (!validType) {
            var info = '';
            if (type === undefined || typeof type === 'object' && type !== null && Object.keys(type).length === 0) {
                info += ' You likely forgot to export your component from the file ' + "it's defined in, or you might have mixed up default and named imports.";
            }

            var sourceInfo = getSourceInfoErrorAddendum(props);
            if (sourceInfo) {
                info += sourceInfo;
            } else {
                info += getDeclarationErrorAddendum();
            }

            info += getStackAddendum() || '';

            warning_1(false, 'React.createElement: type is invalid -- expected a string (for ' + 'built-in components) or a class/function (for composite ' + 'components) but got: %s.%s', type == null ? type : typeof type, info);
        }

        var element = createElement.apply(this, arguments);

        // The result can be nullish if a mock or a custom function is used.
        // TODO: Drop this when these are no longer allowed as the type argument.
        if (element == null) {
            return element;
        }

        // Skip key warning if the type isn't valid since our key validation logic
        // doesn't expect a non-string/function type and can throw confusing errors.
        // We don't want exception behavior to differ between dev and prod.
        // (Rendering will throw with a helpful message and as soon as the type is
        // fixed, the key warnings will appear.)
        if (validType) {
            for (var i = 2; i < arguments.length; i++) {
                validateChildKeys(arguments[i], type);
            }
        }

        if (typeof type === 'symbol' && type === REACT_FRAGMENT_TYPE) {
            validateFragmentProps(element);
        } else {
            validatePropTypes(element);
        }

        return element;
    }
```

这是ReactElement的实现代码，可以看到，createElement的函数签名有3个参数，分别是type,config,children，返回了一个用ReactELement构造函数初始化后的组件。其中还有对defaultProps的处理

| 参数 | 功能 |
| :---: | :---: |
| type | typebabel在解析jsx时会判断标签首字母大小写，大写为自定义组件，小写为原生HTML标签\(string/ReactClass\) |
| config | refs,key等属性都在config中 |
| children | 如果render的为单个组件，那么children参数为空，porps为空，如果为嵌套组件，内部组件作为children放入props中。 |

而ReactELement的构造函数的$$typeof,key, props,ref,\_ower参数释义如下：

| 参数 | 功能 |
| :---: | :---: |
| $$typeof | 组件的标识，通常为REACT\_ELEMENT\_TYPE |
| key | DOM结构的标识id，在diff算法中有用。 |
| props | 嵌套组件有用 |
| ref | 对应真实DOM树的引用 |
| \_ower | \_ower\_owner === ReactCurrentOwner.current\(ReactCurrentOwner.js\),值为创建当前组件的对象，默认值为null。\_owner === ReactCurrentOwner.current\(ReactCurrentOwner.js\),值为创建当前组件的对象，默认值为null。 |

看完上述内容相信大家已经对React组件的实质有了一定的了解。通过执行`React.createElement`创建出的`ReactElement`

类型的js对象，就是组件。进一步说，如果我们通过`class`关键字声明两个组件,那么他们在真正被挂载之前一直是`ReactElement`

类型的js对象。

## 三、组件是怎么挂载到DOM上的

我们知道可以通过`ReactDOM.render(component,mountNode)`的形式对自定义组件/原生DOM/字符串进行挂载，例如

```
ReactDOM.render(<App />, document.getElementById('root'));
```

但是其中是怎么挂载到实际DOM树上的呢？我们可以打开packages/events/react/src/react-dom/src/client/ReactDOM.js看一下

下面是相关代码:

```
function ReactRoot(container: Container, isAsync: boolean, hydrate: boolean) {
  const root = DOMRenderer.createContainer(container, isAsync, hydrate);
  this._internalRoot = root;
}
ReactRoot.prototype.render = function(
  children: ReactNodeList,
  callback: ?() => mixed,
): Work {
  const root = this._internalRoot;
  const work = new ReactWork();
  callback = callback === undefined ? null : callback;
  if (__DEV__) {
    warnOnInvalidCallback(callback, 'render');
  }
  if (callback !== null) {
    work.then(callback);
  }
  DOMRenderer.updateContainer(children, root, null, work._onCommit);
  return work;
};
...
function renderSubtreeIntoContainer(
  parentComponent: ?React$Component<any, any>,
  children: ReactNodeList,
  container: DOMContainer,
  forceHydrate: boolean,
  callback: ?Function,
) {
  invariant(
    isValidContainer(container),
    'Target container is not a DOM element.',
  );

    const isRootRenderedBySomeReact = !!container._reactRootContainer;
    const rootEl = getReactRootElementInContainer(container);
    const hasNonRootReactChild = !!(
      rootEl && ReactDOMComponentTree.getInstanceFromNode(rootEl)
    );

  let root = container._reactRootContainer;
  if (!root) {
    const shouldHydrate =
      forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
    // First clear any existing content.
    if (!shouldHydrate) {
      let warned = false;
      let rootSibling;
      while ((rootSibling = container.lastChild)) {
        
        container.removeChild(rootSibling);
      }
    }
   
    const newRoot = DOMRenderer.createContainer(container, shouldHydrate);
    root = container._reactRootContainer = newRoot;
    // Initial mount should not be batched.
    DOMRenderer.unbatchedUpdates(() => {
      DOMRenderer.updateContainer(children, newRoot, parentComponent, callback);
    });
  } else {
    DOMRenderer.updateContainer(children, root, parentComponent, callback);
  }
  return DOMRenderer.getPublicRootInstance(root);
}
```

这



