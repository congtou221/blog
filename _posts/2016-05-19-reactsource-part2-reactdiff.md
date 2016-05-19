---
layout: post
title: "蕙泉斋"
description: ""
category: 
tags: []
---
{% include JB/setup %}

## React源码剖析——react diff

React最核心的内容在于virtual Dom与diff的结合。用户操作时，可通过React diff计算出virtual Dom中发生变化的部分，并且对发生变化的这部分执行实际的Dom操作。

事实上，diff算法由来已久，但传统的diff算法复杂度很高：O(n^3)。(具体为啥算法复杂度这么高，还不太明白，先挖个坑。参考文献http://grfia.dlsi.ua.es/ml/algorithms/references/editsurvey_bille.pdf)

而React diff算法的复杂度只有O(n)。React diff从三个方面对传统diff算法进行了优化：

*tree diff
*component diff
*element diff

###### element diff
对同一层级的Dom节点，React diff提供三种操作：INSERT_MARKUP(插入)、MOVE_EXISTING(移动)、REMOVE_NODE(删除)。

**key的引入**

如果新的节点集合与老的节点集合是完全一样的，只有节点的位置发生了改变，此时对老的节点集合进行增删以实现更新，效率是很低的。因此React对同一层级的每一个节点都添加了唯一的key，用于在新老集合中比较是否存在相同的节点，如果存在相同的节点，则只会进行节点的位置移动，不再删除和创建节点。

**节点是怎样移动的？**

1.通过key判断在新老集合中是否存在相同的节点，若存在，则执行移动操作。

2.lastIndex：访问过的所有节点(新集合中/virtual Dom)，在老集合中(真实Dom)最右的位置。
child._mountIndex：当前访问的节点(新集合中/virtual Dom)，在老集合中(真实Dom)的位置。

3.如果child._mountIndex < lastIndex，说明在老集合中，当前访问的节点(child._mountIndex)在访问过的所有节点前面(lastIndex)；而在新集合中则相反，当前访问的节点(child._mountIndex)在访问过的所有节点(lastIndex)后面，因此需要移动节点。

![move existing](http://o7bm68198.bkt.clouddn.com/element_diff.png)

eg.以上图为例详细了解移动节点的步骤

*从新集合中取得B，判断老集合中也存在B，因此执行节点移动。

*B在老集合中的child._mountIndex为1，而此时lastIndex为0，child._mountIndex > lastIndex，不对B进行移动。

*进入下一个节点的判断前，更新lastIndex为1(访问过的节点在老集合中最右的位置)。

*从新集合中取得E，判断老集合中不存在E，因此创建新节点E。

*进入下一个节点的判断前，更新lastIndex为1。

*(重复第1到3步)从新集合中取得C，判断老集合中也存在C，因此执行节点移动。

*C在老集合中的child._mountIndex为2，而此时lastIndex为1，child._mountIndex > lastIndex，不对C进行移动(enqueueMove(this,child_mountIndex,toIndex)，toIndex为A在新集合中的位置)。

*进入下一个节点的判断前，更新lastIndex为2。

*(重复第1到3步)从新集合中取得A，判断老集合中也存在A，因此执行节点移动。

*A在老集合中的child._mountIndex为0，而此时lastIndex为2，child._mountIndex < lastIndex，对A进行移动(enqueueMove(this,child_mountIndex,toIndex)，toIndex为A在新集合中的位置)。

*进入下一个节点的判断前，更新lastIndex为2。

*对新集合的所有节点判断完成后，还要对老集合进行遍历，发现在新集合中不存在的节点D，要删除D。至此，diff完成。

	_updateChildren: function(nextNestedChildrenElements, transaction, context) {
	  var prevChildren = this._renderedChildren;
	  var nextChildren = this._reconcilerUpdateChildren(
	    prevChildren, nextNestedChildrenElements, transaction, context
	  );
	  if (!nextChildren && !prevChildren) {
	    return;
	  }
	  var name;
	  var lastIndex = 0;
	  var nextIndex = 0;
	  for (name in nextChildren) {
	    if (!nextChildren.hasOwnProperty(name)) {
	      continue;
	    }
	    var prevChild = prevChildren && prevChildren[name];
	    var nextChild = nextChildren[name];
	    if (prevChild === nextChild) {
	      // 移动节点
	      this.moveChild(prevChild, nextIndex, lastIndex);
	      lastIndex = Math.max(prevChild._mountIndex, lastIndex);
	      prevChild._mountIndex = nextIndex;
	    } else {
	      if (prevChild) {
	        lastIndex = Math.max(prevChild._mountIndex, lastIndex);
	        // 删除节点
	        this._unmountChild(prevChild);
	      }
	      // 初始化并创建节点
	      this._mountChildAtIndex(
	        nextChild, nextIndex, transaction, context
	      );
	    }
	    nextIndex++;
	  }
	  for (name in prevChildren) {
	    if (prevChildren.hasOwnProperty(name) &&
	        !(nextChildren && nextChildren.hasOwnProperty(name))) {
	      this._unmountChild(prevChildren[name]);
	    }
	  }
	  this._renderedChildren = nextChildren;
	},
	// 移动节点
	moveChild: function(child, toIndex, lastIndex) {
	  if (child._mountIndex < lastIndex) {
	    this.prepareToManageChildren();
	    enqueueMove(this, child._mountIndex, toIndex);
	  }
	},
	// 创建节点
	createChild: function(child, mountImage) {
	  this.prepareToManageChildren();
	  enqueueInsertMarkup(this, mountImage, child._mountIndex);
	},
	// 删除节点
	removeChild: function(child) {
	  this.prepareToManageChildren();
	  enqueueRemove(this, child._mountIndex);
	},

	_unmountChild: function(child) {
	  this.removeChild(child);
	  child._mountIndex = null;
	},

	_mountChildAtIndex: function(
	  child,
	  index,
	  transaction,
	  context) {
	  var mountImage = ReactReconciler.mountComponent(
	    child,
	    transaction,
	    this,
	    this._nativeContainerInfo,
	    context
	  );
	  child._mountIndex = index;
	  this.createChild(child, mountImage);
	},

###### tree diff
由于Dom节点跨层级的操作很少，因此React对Virtual Dom树进行了层级控制(通过updateDepth)，只会对同一层级的节点进行比较。这样，只需要对树进行一次遍历，就能完成整个Dom树的比较。

>如下图所示，React只会对相同颜色方框内的Dom节点进行比较。

![tree diff](http://o7bm68198.bkt.clouddn.com/tree_diff.png)

而如果真的出现了跨Dom层级的操作(只可能是节点移动)，只能在同级比较的过程中，依次在不同的层级上添加/删除Dom节点，从而间接实现跨Dom层级移动节点的效果。如下图所示。因此，跨层级的Dom操作是对React性能有影响的，官方建议不要进行跨层级的Dom操作。

![tree diff cross level](http://o7bm68198.bkt.clouddn.com/tree_diff_cross_level.png)

###### component diff

React所采用的组件间的比较策略为：

*同一类型的组件，判断其virtual Dom是否发生了变化(通过shouldComponentUpdate()判断)。如果有变化，则按照上述策略依次进行tree diff、element diff。否则，不执行diff。

*不同类型的组件，直接将该组件(真实Dom中)判断为dirty component，并替换该组件下的所有子节点。

>如下图所示，React判断D和G是不同类型的组件，直接删除D，并重新创建G及其子节点。

![component diff](http://o7bm68198.bkt.clouddn.com/component_diff.png)
