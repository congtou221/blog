---
layout: post
title: "蕙泉斋"
description: ""
category: 
tags: []
---
{% include JB/setup %}

## React源码剖析——生命周期

*React 源码剖析系列 － 生命周期的管理艺术 https://zhuanlan.zhihu.com/p/20312691 读书笔记*

#### 组件和有限状态机

组件，通过状态渲染界面。组件具有自己的生命周期，在其生命周期的特定阶段，组件状态会发生改变，组件方法会被执行。

有限状态机(FSM)，即组件。表示具有有限个状态，并能在这些状态之间转移的模型。 

#### React的生命周期

自定义组件的生命周期，有三个状态：MOUNTING、RECEIVE_PROPS、UNMOUNTING。

**下图为React生命周期的三个状态**

![React生命周期](http://o7bm68198.bkt.clouddn.com/life_circle.png)

#### React的生命周期方法

React的生命周期中的三个状态，分别对应三个方法：mountComponent、updateComponent、unmountComponent。

这三个方法分别提供了两种处理方法：will方法在进入状态前调用；did方法在进入状态后调用。

将react-lifecycle mixin添加到需要观察的组件中，可以在控制台中观察到不同生命周期状态下，生命周期方法的调用顺序。反复实验后发现，首次装载组件时(First Render)、卸载组件时(Unmont)、重新装载组件时(Second Render)、组件状态更新时(Props Change/State Change)，生命周期方法的调用顺序如下。

![生命周期方法的调用顺序](http://o7bm68198.bkt.clouddn.com/life_circle_method.png)

###### 状态一：MOUNTING
Constructor负责管理getDefaultProps。getDefaultProps在整个生命周期的最开始执行，在一个生命周期中只执行1次。

mountComponent负责管理getInitialState、componentWillMount、render、componentDidMount。

通过mountComponent装载组件的过程如下：

1.首先，利用getInitialState获取初始化state。

2.在componentWillMount中调用setState，进行state合并(不触发reRender)。(状态MOUNTING结束，状态NULL开始)

3.state更新，render(可以)获取更新后的this.state数据，进行渲染。(递归渲染：父组件的componentWillMount在子组件的componentWillMount之前调用，且父组件的componentDidMount在子组件的componentDidMount之后调用)

4.渲染完成后，触发componentDidMount。

![mountComponent](http://o7bm68198.bkt.clouddn.com/mountComponent.png)

>instantiateReactComponent创建了自定义元素实例。（在学习React Virtual DOM时，将继续深入了解自定义元素实例是如何被创建的）

下面这段代码映射了前文所述的mountComponent过程：

	// 装载组件
	mountComponent: function(rootID, transaction, mountDepth) {
	// 当前状态为 MOUNTING
	this._compositeLifeCycleState = CompositeLifeCycle.MOUNTING;

	// 当前元素对应的上下文
	this.context = this._processContext(this._currentElement._context);

	// 当前元素对应的 props
	this.props = this._processProps(this.props);

	// 获取初始化 state
	this.state = this.getInitialState();

	// 初始化更新队列
	this._pendingState = null;
	this._pendingForceUpdate = false;

	// componentWillMount 调用setstate，不会触发rerender而是自动提前合并
	if (this.componentWillMount) {
	this.componentWillMount();
	if (this._pendingState) {
	  this.state = this._pendingState;
	  this._pendingState = null;
	}
	}

	// 得到 _currentElement 对应的 component 类实例
	this._renderedComponent = instantiateReactComponent(
	this._renderValidatedComponent(),
	this._currentElement.type
	);

	// 完成 MOUNTING，更新 state
	this._compositeLifeCycleState = null;

	// render 递归渲染
	var markup = this._renderedComponent.mountComponent(
	rootID,
	transaction,
	mountDepth + 1
	);

	// 如果存在 this.componentDidMount，则渲染完成后触发
	if (this.componentDidMount) {
	transaction.getReactMountReady().enqueue(this.componentDidMount, this);
	}

	return markup;
	}

###### 状态二：RECEIVING_PROPS
updateComponent负责管理componentWillReceiveProps、 shouldComponentUpdate、componentWillUpdate、render、componentDidUpdate.

通过updateComponent更新组件的过程如下：

1.若当前组件状态与之前不一致，说明组件需要更新。(此时状态被设为RECEIVING_PROPS)

2.如果存在componentWillReceiveProps，则在componentWillReceiveProps中调用setState，进行state合并(不触发reRender)。(状态更新为NULL)

3.state更新，可通过this.state获取更新后的数据。(然而此时，更新后的this.state只能在源码中使用。在componentWillReceiveProps、shouldComponentUpdate、componentWillUpdate中不能获取更新后的this.state，只有在render及componentDidUpdate中才能获取)

4.执行shouldComponentUpdate，判断是否进行组件更新。

5.如果存在componentDidUpdate，则执行componentDidUpdate。(同上，为递归渲染： 父组件的componentWillUpdate在子组件的componentWillUpdate之前调用，父组件的componentDidUpdate在子组件的componentDidUpdate之后调用)

6.渲染完成后，触发componentDidUpdate。

![updateComponent](http://o7bm68198.bkt.clouddn.com/updateComponent.png)

>在shouldComponentUpdate和componentWillUpdate中禁止调用setState，会造成循环引用，耗光浏览器内存。

	// 更新组件
	updateComponent: function(transaction, prevParentElement, nextParentElement) {
	  var prevContext = this.context;
	  var prevProps = this.props;
	  var nextContext = prevContext;
	  var nextProps = prevProps;

	  if (prevParentElement !== nextParentElement) {
	    nextContext = this._processContext(nextParentElement._context);
	    nextProps = this._processProps(nextParentElement.props);
	    // 当前状态为 RECEIVING_PROPS
	    this._compositeLifeCycleState = CompositeLifeCycle.RECEIVING_PROPS;

	    // 如果存在 componentWillReceiveProps，则执行
	    if (this.componentWillReceiveProps) {
	      this.componentWillReceiveProps(nextProps, nextContext);
	    }
	  }

	  // 设置状态为 null，更新 state
	  this._compositeLifeCycleState = null;
	  var nextState = this._pendingState || this.state;
	  this._pendingState = null;
	  var shouldUpdate =
	    this._pendingForceUpdate ||
	    !this.shouldComponentUpdate ||
	    this.shouldComponentUpdate(nextProps, nextState, nextContext);
	  if (!shouldUpdate) {
	    // 如果确定组件不更新，仍然要设置 props 和 state
	    this._currentElement = nextParentElement;
	    this.props = nextProps;
	    this.state = nextState;
	    this.context = nextContext;
	    this._owner = nextParentElement._owner;
	    return;
	  }
	  this._pendingForceUpdate = false;

	  ......

	  // 如果存在 componentWillUpdate，则触发
	  if (this.componentWillUpdate) {
	    this.componentWillUpdate(nextProps, nextState, nextContext);
	  }

	  // render 递归渲染
	  var nextMarkup = this._renderedComponent.mountComponent(
	    thisID,
	    transaction,
	    this._mountDepth + 1
	  );

	  // 如果存在 componentDidUpdate，则触发
	  if (this.componentDidUpdate) {
	    transaction.getReactMountReady().enqueue(
	      this.componentDidUpdate.bind(this, prevProps, prevState, prevContext),
	      this
	    );
	  }
	}

###### 状态三：UNMOUNTING
unmountComponent负责管理componentWillUnmount。

通过unmountComponent卸载组件的过程如下：

1.状态设置为UNMOUNTING。

2.若存在componentWillUnmount，则执行componentWillUnmount。(在此过程中不会触发render)

3.状态更新为NULL，完成组件卸载。

	// 卸载组件
	unmountComponent: function() {
	  // 设置状态为 UNMOUNTING
	  this._compositeLifeCycleState = CompositeLifeCycle.UNMOUNTING;

	  // 如果存在 componentWillUnmount，则触发
	  if (this.componentWillUnmount) {
	    this.componentWillUnmount();
	  }

	  // 更新状态为 null
	  this._compositeLifeCycleState = null;
	  this._renderedComponent.unmountComponent();
	  this._renderedComponent = null;

	  ReactComponent.Mixin.unmountComponent.call(this);
	}

*答疑：为什么禁止在shouldComponentUpdate/componentWillUpdate中调用setState？*

调用setState时，会对state及_pendingState更新、合并。在这一过程中，会通过replaceState判断当前状态。当状态不是MOUNTING/RECEIVING_PROPS时，proformUpdate会调用updateComponent更新组件。但updateComponent又会反过来调用shouldComponentUpdate/componentWillUpdate，最终造成循环引用，导致内存占满，浏览器崩溃。

![setStateDie](http://o7bm68198.bkt.clouddn.com/setStateDie.png)

	// 更新 state
	setState: function(partialState, callback) {
	  // 合并 _pendingState
	  this.replaceState(
	    assign({}, this._pendingState || this.state, partialState),
	    callback
	  );
	},

	// 更新 state
	replaceState: function(completeState, callback) {
	  validateLifeCycleOnReplaceState(this);

	  // 更新队列
	  this._pendingState = completeState;

	  // 判断状态是否为 MOUNTING，如果不是，即可执行更新
	  if (this._compositeLifeCycleState !== CompositeLifeCycle.MOUNTING) {
	    ReactUpdates.enqueueUpdate(this, callback);
	  }
	},

	// 如果存在 _pendingElement、_pendingState、_pendingForceUpdate，则更新组件
	performUpdateIfNecessary: function(transaction) {
	  var compositeLifeCycleState = this._compositeLifeCycleState;

	  // 当状态为 MOUNTING 或 RECEIVING_PROPS时，则不更新
	  if (compositeLifeCycleState === CompositeLifeCycle.MOUNTING ||
	      compositeLifeCycleState === CompositeLifeCycle.RECEIVING_PROPS) {
	    return;
	  }

	  var prevElement = this._currentElement;
	  var nextElement = prevElement;
	  if (this._pendingElement != null) {
	    nextElement = this._pendingElement;
	    this._pendingElement = null;
	  }

	  // 调用 updateComponent
	  this.updateComponent(
	    transaction,
	    prevElement,
	    nextElement
	  );
	}

如下图所示，不建议在getDefaultProps、getInitialState、shouldComponentUpdate、componentWillUpdate、render 和 componentWillUnmount中调用setState。禁止在shouldComponentUpdate、componentWillUpdate中调用setState，会导致循环调用。

![dont setState](http://o7bm68198.bkt.clouddn.com/dontSetState.png)