# RFC: node-reactions

- Status: **Draft**
- Start Date: 2019-07-15
- RFC PR: (leave this empty)
- Lightning Web Component Issue: (leave this empty)

# Summary

node-reactions provides a way to react to lifecycle of a DOM node. The library accepts lifecycle hooks per DOM node and invokes them synchronously when the lifecycle event occurs. The primary focus of this RFC is the connectedness of a node.

# Basic example

*Proposal 1*: One simple API that accepts hooks for various lifecycle events
```js
/**
 * @param {Node} node - Node that should be observed
 * @param {String} reactionEventType - one of connected, disconnected, adopted etc
 * @param {Function} callback
 */
function reactTo(node, reactionEventType, callback);
```

*Proposal 2*: One API per lifecycle event
```js
addConnectedCallback(node, callback);
addDisconnectedCallback(node, callback);
```

# Motivation
The browser provides ways to react to lifecycle events in the window or document. For example, [`DOMContentLoaded`](https://developer.mozilla.org/en-US/docs/Web/API/Window/DOMContentLoaded_event) event signals that the initial HTML document has been loaded and parsed. The [`load`](https://developer.mozilla.org/en-US/docs/Web/API/Window/load_event) event signals that the page is full loaded. Similarly, [`beforeunload`](https://developer.mozilla.org/en-US/docs/Web/API/WindowEventHandlers/onbeforeunload) allows reactions to the resources being unloaded.

Web Components supports reacting to the lifecycle of a custom element. Hooks for  lifecycle events are part of the custom component definition. Read the spec [here](https://developer.mozilla.org/en-US/docs/Web/Web_Components/Using_custom_elements#Using_the_lifecycle_callbacks) for more detail. 

Lightning Web Components(LWC) has support for similar lifecycle events. Of particular interest to this RFC is the `connectedCallback` and `disconnectedCallback`. In LWC, the responsibility of invoking the hooks is distributed between the engine itself and snabdom. The goal of this RFC is to abstract the responsibility of managing the lifecycle into a library and make LWC subscribe to lifecycle events from the node-reactions library.

Further, there are inconsistencies in the current implementation. For example, connectedCallback should only be invoked if a custom element is connected to the document. In LWC, connectedCallback is invoked anytime a custom element is connected to a parent, regardless of wether the parent is connected to the document or not.

# Detailed design

## When does a node qualify as connected?

For custom elements, a node that is appended to the document or one of its connected discendents is considered as connected. [Node.isConnected](https://dom.spec.whatwg.org/#dom-node-isconnected) offers the most reliable way to determine if a node is connected.

## Invariants
1. A node can be connected and disconnected from the DOM more than once. Every time the event happens, the qualifying hook should be invoked.
2. For a given node, more than one service can be registered to react to the same lifecycle event. In which case, each hook should be invoked in the order of registration(FIFO). 
    2.1 Duplicate hooks for the same event type, for the same node will be ignored. For example, if a service registers the same connectedCallback more than once for the same node, the callback is invoked only once.
3. Hooks are invoked synchronously.
4. In case of an event affecting a subtree, nodes are processed in [preorder(or treeorder)](https://dom.spec.whatwg.org/#concept-tree-order).

## Proposal: Patch DOM mutations API to sniff activity

### Inbound
On the inbound side, the library will accept requests for a DOM node. A map will maintain the relationship between the node and all of its hooks per lifecycle event.
```js
WeakMap<Node, { [key: string]: Array<keyOf Function> }> 
```
Following constraints govern the API usage:
    1. node param is required and has to be a Node
    2. node cannot be a shared node(document, body, head) - This is just an optimization. Shared nodes are mostly managed by the browser and do not go through lifecycle events as frequently as regular DOM nodes. 

*The references to the Node will be weak so as to not interfere with garbage collection. The hooks are garbage collectible only when all the nodes they are associated with are garbage collected.*

#### Callback
- Option 1
    ```js
    type ReactionCallback = (
        node: Node,
        reactionEventType: String,
        parentNode: Node
    ) => void;
    ```
- Option 2
    ```js
    interface ReactionMessage {
        node: Node,
        reactionEventType: String,
        parentNode: Node
    };

    type ReactionCallback = ( message: ReactionMessage) => void;
    ```

### Outbound
Post registration, the library is responsible for invoking the hook when a qualifying event occurs in the node being observed. 

The library will identify the DOM APIs that can trigger mutations in a node. When an event occurs, the library will take action to invoke all the eligible lifecycle hooks.

### Order of processing

```
1. Consider a node where the mutation has occured as currentNode, parentNode is the node's parent.
2. Identify the actors
    2.1 If currentNode has been added without affecting other nodes in the parent's subtree, then the actors are currentNode and its subtree.
    2.2 If currentNode is replacing an existing child node of the parent, then the actors are the existing child node and its subtree, the currentNode and its subtree.
    2.3 If currentNode's subtree is being mutated without changing the placement of the current node itself, then the actors are all the child nodes of currentNode and their subtree.
3. Identify the order of traversal
    3.1 Predetermine the order in which the hooks will be invoked.
    3.2 Once a hook is qualified, it will be invoked regardless of any mutations that occur during invocation of prior hooks.
4. If the same node qualifies for more than one lifecycle event e.g, appendChild(existingConnectedNode)
    4.1 Then process all lifecycle events for the given node
    4.2 The order of lifecycle events is: disconnectedCallback, connectedCallback
    4.3 After all hooks for all events have been invoked, move on to other nodes as per order of traversal.
5. If an existing node is being replaced by a new node
    5.1 Process lifecycle events for existing node(s)
    5.2 Next, process lifecycle events for new node(s)
6. If the currentNode is a custom element, then the order of processing is as follows(TODO - provide link to native custom elements example)
    6.1 The host element is processed first
    6.2 shadowRoot and nodes in the shadow tree next
    6.3 Nodes in the light DOM
```

### What browser APIs control a node's lifecycle?

#### DOM APIs that connect a node

```js
Node.appendChild
Node.insertBefore
Node.replaceChild

ParentNode.prepend
ParentNode.append

Element.insertAdjacentElement

ChildNode.after
ChildNode.before
ChildNode.replaceWith

Range.insertNode

```

#### DOM APIs that disconnect a node

```js
ChildNode.after
ChildNode.before
ChildNode.remove
ChildNode.replaceWith

Element.insertAdjacentElement
Element.innerHTML
Element.outerHTML

Node.appendChild
Node.insertBefore
Node.nodeValue
Node.replaceChild
Node.removeChild
Node.textContent

Range.insertNode
Range.deleteContents
Range.extractContents
Range.surroundContents
```

#### Reactions for various APIs
Here are some sample reactions for the APIs listed above.
- Node.appendChild: 
    - If the childNode has an existing parent and the parent is connected, invoke disconnectedCallback for the childNode and its subtree.
    - If new parent is connected, invoke connectedCallback for the childNode and its subtree

- Node.insertBefore: Same steps as Node.appendChild

- Node.replaceChild
    - if the parent node is connected
        - invoke disconnectedCallback for the existing child and its subtree
        - invoke disconnectedCallback for the new child and its subtree

# Drawbacks

Why should we *not* do this? Please consider:

- Performance: Traversing the dom is not cheap. We need to think about memoization techniques to prevent repetitive traversal. This is of significant importance since the invocations are synchronous in nature.
- Security: Can a malicious actor sniff and get access to hooks that they do not own? If they did, could utilize that hook and attack the subscriber by simulating an event.
- Native web components: When we switch to native web components, will this library become obsolete?

# Alternatives

## MutationObserver
 MutationObserver is a browser API to monitor mutations in a node and its subtree. MutationObserver is not a viable option because of the following limitations:
 1. Callbacks are invoked asynchronously. This is a major limitation because lifecycle callbacks are to be invoked synchronously as per spec. Asynchronous reactions are also a limitation for applying styling to shadow dom in case of portal elements(TODO: clarify).

# Adoption strategy

The usage of this library in LWC will be an implementation detail of the engine. Component developers should notive no difference in usage.

How should this feature be taught to existing Lightning Web Components developers?

# Unresolved questions

- How will traversal work for native custom elements running in closed shadow mode? In web components, Lifecycle hooks behave the same regardless of mode of shadow DOM. For example: https://jsbin.com/misofod/edit?html,js,console,output
- Are there any browser APIs that are faster than manual DOM traversal?

Notes:
1. isConnected works for regular dom and shadow DOM: https://jsbin.com/femohaqoku/edit?html,js,console,output
2. isConnected works for shadowRoot node and other documentFragments
3. When a node is moved to a different subtree, invoke its disconnectedCallback first and then the connectedCallback: https://jsbin.com/qawacil/edit?html,js,output
4. All hooks for a given node is processed first before moving down the subtree: https://jsbin.com/qawacil/edit?html,js,output
5. Hooks in subtree are invoked even in closed shadow mode.

TODOs:
1. Usage of the word `hooks`, `callbacks` interchageably
2. `Lifecycle events`, `mutations` are used interchageably
