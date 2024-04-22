## 手写一个 react-mini

_【react-mini】_ 最近想手写一下 react-mini 于是便搜集了一些资料 加上对于这段时间学习 react 的经验 ，开始手写一下，只有通过手写才能更加深入的去了解内部运行机制。也是给自己的查缺补漏和技术分享。

_笔者文章集合详见：_

- [GitHub 地址](Fengyain/fyb-react-mini "GitHub 地址")：https://github.com/Fengyain/fyb-react-mini

## createElement

接下来我们从最简单的 createElement 开始实现：
我们在使用 react 时，只有三行代码。第一个定义了一个 React 元素，下一个从 DOM 获得一个节点，最后一个函数将 React 元素呈现到容器中。

```
const element = <h1 title="foo">Hello</h1>
const container = document.getElementById("root")
ReactDOM.render(element, container)

```

在第一行，我们用 JSX 定义元素。它甚至不是有效的 JavaScript，因此为了用普通的 JS 替换它，首先我们需要用有效的 JS 替换它。通过诸如 Babel 之类的构建工具，JSX 被转换为 JS。
我们可以通过[babel 在线网站](https://www.babeljs.cn/repl#?browsers=defaults%2C%20not%20ie%2011%2C%20not%20ie_mob%2011&build=&builtIns=false&corejs=3.21&spec=false&loose=false&code_lz=DwCwjABALgllA2BTAvAIgGYHtOoHwAlF55NgB6cXIA&debug=false&forceAllTransforms=false&modules=false&shippedProposals=false&circleciRepo=&evaluate=false&fileSize=false&timeTravel=false&sourceType=module&lineWrap=true&presets=env%2Creact%2Cstage-2&prettier=false&targets=&version=7.24.4&externalPlugins=%40babel%2Fplugin-transform-react-jsx%407.23.4&assumptions=%7B%7D)查看被转换过后的代码：

```
React.createElement("h1", {
  title: "foo"
}, "Hello");
```

`React.createElement `除了做了一些校验本质上就是返回了一个对象，我们可以很安全的将他的返回结果-一个处理过后的对象作为最终的输出：

```
const element = React.createElement(
  "h1",
  { title: "foo" },
  "Hello"
)
//转换为：
const element = {
  type: "h1",
  props: {
    title: "foo",
    children: "Hello",
  },
}
```

那如何转换呢？我们一会聊。。
这就是一个元素，一个有两个属性的对象: (它有更多的属性，但是我们只关心这两个属性)。
于是我么可以通过 element 去创建元素并将 props 赋值给创建的元素：

```
onst node = document.createElement(element.type)
node["title"] = element.props.title
​
const text = document.createTextNode("")
text["nodeValue"] = element.props.children

node.appendChild(text)
container.appendChild(node)
```

以上是通过 createElement 转换为基本的保存节点信息的对象，再通过该对象去创建节点的过程，接下来回答如何转换的问题：

```
function createElement(type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map(child =>
        typeof child === "object"
          ? child
          : createTextElement(child)
      ),
    },
  }
}
function createTextElement(text) {
  return {
    type: "TEXT_ELEMENT",
    props: {
      nodeValue: text,
      children: [],
    },
  }
}
```

如果我们有一个像这样的注释，当 babel 传递 JSX 时，它将使用我们定义的函数：

```
const Didact = {
  createElement,
}
​
/** @jsx Didact.createElement */
const element = (
  <div id="foo">
    <a>bar</a>
    <b />
  </div>
)
const container = document.getElementById("root")
ReactDOM.render(element, container)
```

## 接下来完成 render 函数：

现在，我们只关心向 DOM 添加内容，稍后我们将处理更新和删除。
我们首先使用元素类型创建 DOM 节点，然后将新节点附加到容器中。

我们递归地对每个孩子执行相同的操作。

我们还需要处理文本元素，如果元素类型是我们创建的文本节点，而不是常规的节点，

这里我们需要做的最后一件事是将元素分配给节点。

```
function render(element, container) {
  const dom =
    element.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(element.type)
​
  const isProperty = key => key !== "children"
  Object.keys(element.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = element.props[name]
    })
​
  element.props.children.forEach(child =>
    render(child, dom)
  )
​
  container.appendChild(dom)
}
```

## 实现 Concurrent Mode

在上方`render`中`element.props.children.forEach`这个递归调用有一个问题。

一旦我们开始呈现，我们将不会停止，直到我们呈现完整的元素树。如果元素树很大，它可能会阻塞主线程太长时间。如果浏览器需要处理用户输入或保持动画流畅等高优先级的事情，它将不得不等待，直到渲染完成。

所以我们要把工作分成几个小单元，每个单元完成后，如果还有其他需要做的事情，我们会让浏览器中断渲染。

我们用来做一个循环,浏览器不会告诉它何时运行，而是在主线程空闲时运行回调。

_React 不再使用 requestIdleCallback。现在它使用调度程序包。但是对于这个用例，它在概念上是相同的。_

RequestIdleCallback 还为我们提供了一个截止日期参数。我们可以用它来检查我们有多少时间，直到浏览器需要再次控制。

要开始使用循环，我们需要设置第一个工作单元，然后编写一个函数，它不仅执行工作，而且还返回下一个工作单元。

```
let nextUnitOfWork = null
​
function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }
  requestIdleCallback(workLoop)
}
​
requestIdleCallback(workLoop)
```

## Fibers

为了组织工作单元，我们需要一个数据结构: fiber tree。

每个元素有一个 fiber ，每个 fiber 是一个工作单元。

我们将会对每一个 fiber 做三件事情：

- 将元素添加到 DOM
- 为元素的子元素创造 fiber
- 选择下一个工作单元

这种数据结构的目标之一是便于查找下一个工作单元。这就是为什么每一个 fiber 都与它的第一个孩子、下一个兄弟姐妹和它的父母有联系。

![](https://files.mdnice.com/user/63505/21b8ec86-8988-4caa-893a-59f6ce6318ed.png)

当我们完成对 fiber 的工作时，如果它有一个孩子，那么 fiber 将是下一个工作单元。

在我们的示例中，当我们完成对 div fiber 的处理时，下一个工作单元将是 h1 fiber。

如果 fiber 没有孩子，我们使用兄弟姐妹作为下一个工作单元。

例如，p fiber 没有孩子，所以我们在完成后移动到 a fiber。

如果 fiber 没有孩子也没有兄弟姐妹，我们就去找“叔叔”: 父母的兄弟姐妹。比如样本中的 a 和 h2 fiber。

此外，如果父母没有兄弟姐妹，我们继续通过父母，直到我们找到一个与兄弟姐妹或直到我们达到根。如果我们已经到达了根，这意味着我们已经完成了这个渲染的所有工作。
重写 render 方法：

```
​
function createDom(fiber) {
  const dom =
    fiber.type == "TEXT_ELEMENT"
      ? document.createTextNode("")
      : document.createElement(fiber.type)
​
  const isProperty = key => key !== "children"
  Object.keys(fiber.props)
    .filter(isProperty)
    .forEach(name => {
      dom[name] = fiber.props[name]
    })
​
  return dom
}
​
function render(element, container) {
  nextUnitOfWork = {
    dom: container,
    props: {
      children: [element],
    },
  }
}
​
let nextUnitOfWork = null
```

在`workLoop`函数中，当浏览器准备好时，它将调用我们的 workLoop，我们将开始处理 root。

首先，我们创建一个新节点并将其附加到 DOM。

我们在 fiber.DOM 属性中跟踪 DOM 节点。

然后我们为每个孩子创造一种新的 fiber。

我们把它添加到 fiber tree 中，根据它是否是第一个孩子，把它设置为一个孩子或者一个兄弟姐妹。

最后，我们寻找下一个工作单元。我们首先尝试与孩子，然后与兄弟姐妹，然后与叔叔，等等。

```
function performUnitOfWork(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
​
  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }
​
  const elements = fiber.props.children
  let index = 0
  let prevSibling = null
​
  while (index < elements.length) {
    const element = elements[index]
​
    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    }
​
    if (index === 0) {
      fiber.child = newFiber
    } else {
      prevSibling.sibling = newFiber
    }
​
    prevSibling = newFiber
    index++
  }
​
  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }
}
```

## Render and Commit

我们还有一个问题。

每次处理一个元素时，我们都要向 DOM 添加一个新节点。还有，浏览器可能会在我们绘制完整棵树之前中断我们的工作,在这种情况下，用户将看到一个不完整的 UI。

因此，我们需要从这里删除改变 DOM 的部分。
删除：

```
 if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom)
  }
```

相反，我们将跟踪 fiber tree 的根。我们将其称为“正在进行的工作”root 或 wipRoot。

一旦我们完成了所有的工作(我们知道这一点，因为没有下一个工作单元) ，我们将整个 fiber tree 提交给 DOM。

```
function commitRoot() {
  // TODO add nodes to dom
}
​
function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
  }
  nextUnitOfWork = wipRoot
}
​
let nextUnitOfWork = null
let wipRoot = null
​
function workLoop(deadline) {
  let shouldYield = false
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(
      nextUnitOfWork
    )
    shouldYield = deadline.timeRemaining() < 1
  }
​
  if (!nextUnitOfWork && wipRoot) {
    commitRoot()
  }
​
  requestIdleCallback(workLoop)
}
```

我们在 committee Root 函数中执行，在这里我们递归地将所有节点附加到 dom。

## Reconciliation

到目前为止，我们只向 DOM 添加了一些东西，但是更新或删除节点会怎么样呢？

这就是我们现在要做的，我们需要比较我们在渲染函数上接收到的元素和我们提交给 DOM 的最后一个 fiber tree。

因此，在完成提交之后，我们需要保存对“我们提交到 DOM 的最后一个 fiber tree”的引用。我们称之为 currentRoot。

我们还将替代属性添加到每个 fiber。这个属性是一个到旧 fiber 的链接，旧 fiber 是我们在前一个提交阶段提交给 DOM 的 fiber。

现在让我们从 PerformUnitOfWork 中提取代码来创建新的 fiber..

在这里我们将调和旧的 fiber 与新的元素。

我们同时迭代旧 fiber(wipFiber.Alternate)的子元素和我们想要调和的元素数组。

如果我们忽略同时迭代一个数组和一个链表所需的所有样板，那么在这段时间里我们只剩下最重要的东西: oldFiberandelement。元素是我们想要呈现给 DOM 的东西，而 old fiber 是我们上次呈现的东西。

我们需要比较它们，看看是否需要对 DOM 应用任何更改。

为了比较它们，我们使用类型:

- 如果旧的 fiber 和新的元素有相同的类型，我们可以保留 DOM 节点，只是更新它与新的 porps

- 如果类型不同并且有一个新元素，则意味着我们需要创建一个新的 DOM 节点

- 如果类型不同，有一个旧的 fiber，我们需要删除旧的节点

_在这里 react 也使用 key，这是一个更好的协调。例如，它检测子元素在元素数组中的位置何时发生更改。_

当旧 fiber 和单元具有相同类型时，我们创建一个新的 fiber，保持旧 fiber 中的 DOM 节点和单元中的 props。

我们还在 fiber 中添加了一个新属性: effectTag。

然后，对于元素需要一个新的 DOM 节点的情况，我们使用 PLACEMENT 效果标记来标记新的 fiber。

对于需要删除节点的情况，我们没有新的 fiber，所以我们将 effectTag 添加到旧 fiber。

但是当我们将 fiber tree 提交到 DOM 时，我们是从正在进行的工作根中提交的，根中没有旧的 fiber。

所以我们需要一个数组来跟踪我们想要删除的节点。

然后，当我们提交对 DOM 的更改时，我们也使用来自该数组的 fiber。

现在，让我们更改 committee Work 函数来处理新的 effectTags。

如果光纤有一个 PLACEMENT 效果标记，我们将执行与前面相同的操作，将 DOM 节点从父 fiber 追加到该节点。

如果是删除，删除孩子。

```
function commitRoot() {
  deletions.forEach(commitWork);
  commitWork(wipRoot.child);
  currentRoot = wipRoot;
  wipRoot = null;
}

function commitWork(fiber) {
  if (!fiber) {
    return;
  }

  let domParentFiber = fiber.parent;
  while (!domParentFiber.dom) {
    domParentFiber = domParentFiber.parent;
  }
  const domParent = domParentFiber.dom;

  if (fiber.effectTag === "PLACEMENT" && fiber.dom != null) {
    domParent.appendChild(fiber.dom);
  } else if (fiber.effectTag === "UPDATE" && fiber.dom != null) {
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  } else if (fiber.effectTag === "DELETION") {
    commitDeletion(fiber, domParent);
  }

  commitWork(fiber.child);
  commitWork(fiber.sibling);
}

function commitDeletion(fiber, domParent) {
  if (fiber.dom) {
    domParent.removeChild(fiber.dom);
  } else {
    commitDeletion(fiber.child, domParent);
  }
}

function render(element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
    alternate: currentRoot,
  };
  deletions = [];
  nextUnitOfWork = wipRoot;
}

let nextUnitOfWork = null;
let currentRoot = null;
let wipRoot = null;
let deletions = null;

function workLoop(deadline) {
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }

  if (!nextUnitOfWork && wipRoot) {
    commitRoot();
  }

  requestIdleCallback(workLoop);
}

requestIdleCallback(workLoop);

function performUnitOfWork(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber)
  }
​
  const elements = fiber.props.children
  reconcileChildren(fiber, elements)
​
  if (fiber.child) {
    return fiber.child
  }
  let nextFiber = fiber
  while (nextFiber) {
    if (nextFiber.sibling) {
      return nextFiber.sibling
    }
    nextFiber = nextFiber.parent
  }
}
​
function reconcileChildren(wipFiber, elements) {
  let index = 0;
  let oldFiber = wipFiber.alternate && wipFiber.alternate.child;
  let prevSibling = null;

  while (index < elements.length || oldFiber != null) {
    const element = elements[index];
    let newFiber = null;

    const sameType = oldFiber && element && element.type == oldFiber.type;

    if (sameType) {
      newFiber = {
        type: oldFiber.type,
        props: element.props,
        dom: oldFiber.dom,
        parent: wipFiber,
        alternate: oldFiber,
        effectTag: "UPDATE",
      };
    }
    if (element && !sameType) {
      newFiber = {
        type: element.type,
        props: element.props,
        dom: null,
        parent: wipFiber,
        alternate: null,
        effectTag: "PLACEMENT",
      };
    }
    if (oldFiber && !sameType) {
      oldFiber.effectTag = "DELETION";
      deletions.push(oldFiber);
    }

    if (oldFiber) {
      oldFiber = oldFiber.sibling;
    }

    if (index === 0) {
      wipFiber.child = newFiber;
    } else if (element) {
      prevSibling.sibling = newFiber;
    }

    prevSibling = newFiber;
    index++;
  }
}
```

如果它是一个 UPDATE，我们需要用更改过的 props 更新现有的 DOM 节点。

我们将在 updateDom 函数中执行此操作。

我们将旧 fiber 与新 fiber 的 props 进行比较，去掉不见的 props，设置新的或更换的 props。

我们需要更新的一种特殊的 props 是事件侦听器，因此如果道具名称以“ on”前缀开头，我们将对它们进行不同的处理。

如果事件处理程序发生更改，我们将其从节点中删除。

然后我们添加新的处理程序。

```
const isEvent = (key) => key.startsWith("on");
const isProperty = (key) => key !== "children" && !isEvent(key);
const isNew = (prev, next) => (key) => prev[key] !== next[key];
const isGone = (prev, next) => (key) => !(key in next);
function updateDom(dom, prevProps, nextProps) {
  //Remove old or changed event listeners
  Object.keys(prevProps)
    .filter(isEvent)
    .filter((key) => !(key in nextProps) || isNew(prevProps, nextProps)(key))
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.removeEventListener(eventType, prevProps[name]);
    });

  // Remove old properties
  Object.keys(prevProps)
    .filter(isProperty)
    .filter(isGone(prevProps, nextProps))
    .forEach((name) => {
      dom[name] = "";
    });

  // Set new or changed properties
  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      dom[name] = nextProps[name];
    });

  // Add event listeners
  Object.keys(nextProps)
    .filter(isEvent)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.addEventListener(eventType, nextProps[name]);
    });
}
```

## Function Components

Function Components 在两个方面有所不同:

来自函数组件的 fiber 没有 DOM 节点

孩子们来自运行函数，而不是直接从 props 得到他们.

我们检查 fiber 类型是否是一个函数，并根据这一点，我们去一个不同的更新函数。

在 updateHostComponent 中，我们执行与前面相同的操作。

在 updateFunctionComponent 中，我们运行函数来获取子元素。

对于我们的示例，这里 fiber.type 是 App 函数，当我们运行它时，它返回 h1 元素。

然后，一旦我们有了孩子，和解也会以同样的方式进行，我们不需要改变那里的任何东西。我们需要改变的是 committee Work 函数。

现在我们有了没有 DOM 节点的 fiber，我们需要改变两件事情。

首先，要找到 DOM 节点的父节点，我们需要沿着 fiber 向上查找，直到找到具有 DOM 节点的 fiber。

在删除一个节点时，我们还需要继续操作，直到找到一个具有 DOM 节点的子节点。
