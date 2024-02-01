# Bevy Cobweb (IN DEVELOPMENT)

**There is no code yet, only a draft of the proposed design in this document.**

Framework for building declarative reactive webs.

- Nodes are stateful reactive Bevy systems.
- Change detection prevents reinitializing and rerunning nodes unless needed.
- Nodes react to resource mutations, entity changes, reactive events, and node events; recursive reactions are allowed!
- Node outputs can be accessed throughout the web via node handles that synchronize with rebuilds.
- Root node error handling policy is customizable. Errors propagate to root nodes.
- Nodes may be detached and re-attached anywhere in the web.
- Nodes are automatically cleaned up when no longer used.
- Web mutations and node reactions have zero frame delay.



## Hello World

Here is a hypothetical example of writing `"Hello, World!"` to the screen. Note that `bevy_cobweb_ui` does not exist yet.

```rust
use bevy::prelude::*;
use bevy_cobweb::{CobwebPlugin, NodeHandle, SystemExt, Web};
use bevy_cobweb_ui::{Location, ScreenArea, TextNode, TextSize, WindowArea};

fn hello_world(
    mut web : Web,
    window  : Query<Entity, With<PrimaryWindow>>
) -> NodeResult<()>
{
    let area: NodeHandle<ScreenArea> = WindowArea::new(window.single())
        .build(&mut web)?;

    TextNode::new()
        .default_text("Hello, World!")
        .location(area, Location::TotallyCentered)
        .size(area, TextSize::RelativeHeight(10.))
        .build(&mut web)?;

    Ok(())
}

fn main()
{
    App::new()
        .add_plugins(DefaultPlugins::default())
        .add_plugins(CobwebPlugin::default())
        .add_systems(Setup, hello_world.webroot())
        .run();
}
```

The `WindowArea` and `TextNode` types seen here are custom node builders that internally use the `bevy_cobweb` API. The `hello_world` system is a root node owned by a system produced by the [`.webroot()`](bevy_cobweb::webroot) system adaptor, which packages and stores the node in a `Local`. If the `hello_world` node errors-out, then the error handling policy configured in [`CobwebPlugin`](bevy_cobweb::CobwebPlugin) will be used (panic by default).

If the window is resized, then `WindowArea` will rebuild because it is internally set up to react to changes in the window size. When the `WindowArea`'s output changes, its parent node `hello_world` will be rebuilt automatically. When `hello_world` rebuilds, the `TextNode` child will also rebuild if `area` has changed, thereby propagating the window size change to its dependents. Internally, `TextNode` will re-use its existing `Text` entity, avoiding string reallocation.



## Deep-Dive

A web is a structure analogous to a forest covered in cobwebs. Each 'tree' is a physical branching structure of nodes, and between all the nodes are reactive relationships (the 'web').

There are two kinds of reactive relationships. One is ECS reactivity, where nodes will rebuild in response to changes in tracked ECS elements (resource mutations, entity changes, etc.). The other is inter-node dependencies, where nodes can depend on the outputs of upstream nodes. When a referenced node output changes, any nodes dependent on that output will rebuild.

Any node in the web can be detached from its parent and sent to be reattached elsewhere in the web (root nodes can also be attached to other nodes).

As you might imagine, being able to reference the outputs of other nodes is both powerful and risky, especially given the ability to rearrange the web. We will discuss how to properly manage node references to ensure you avoid errors and bugs.


### Web Structure Overview

At the base of the web are [`RootNodes`](bevy_cobweb::RootNode), each representing the root of a node tree. Root nodes are simple wrappers around packaged [`BasicNodes`](bevy_cobweb::BasicNode), which means they can be created from normal nodes in the web. Root nodes are not allowed to depend on other nodes, and they do not have outputs (specifically, they may only wrap a `BasicNode<S, I, ()>`, where `S` and `I` implement `Hash`).

Every node in the tree can have child nodes. Child nodes come in two types, built-in nodes that are tracked implicitly, and object nodes that can be packaged and relocated ([`BasicNodes`](bevy_cobweb::BasicNode), [`ProducerNodes`](bevy_cobweb::BasicNode), and [`ReactorNodes`](bevy_cobweb::BasicNode)). Parents track their child nodes through 'node names', which are unique identifiers that allow the web to compare node metadata between successive rebuilds. Built-in child nodes can either be explicitly named by the user, or are assigned an anonymous name based on their index in the list of anonymous built-in child nodes. Object node names are derived from their unique node ids, which are global ids within the web.

Building a node involves specifying the node's initial state, input, reaction triggers, and internal system. Exactly how these are specified depends on the node type, which we will discuss in later sections. The first time a node is built, its internal system will be scheduled to run after the previous child node's system. After a node system runs, its output will be saved in the web for downstream nodes to read. We discuss the `bevy_cobweb` scheduling algorithm toward the end of this document.

When a parent node runs its node system, it will detect which of its children were built, and then destroy any children that were built in the past but fail to be rebuilt. It does this by comparing the child node name lists before and after the rebuild. Anonymous built-in, named built-in, and object nodes will all be destroyed if not rebuilt. Object-type child nodes cannot be rebuilt after being destroyed, but built-in node names can be reused for new nodes.

Building a child node uses change detection to avoid re-running the child node's internal node system unless necessary. The node state and node input passed to a node when building it are compared against hashes of state and input used the last time the node was built. If the hashes are the same, then the node system will not be scheduled and its output will stay the same.

As mentioned, child nodes can be assigned reaction triggers, which specify which ECS mutations will cause the node system to re-run automatically. When a node runs after being triggered, it will re-use its existing state and input, and will produce a new output. Node state is mutable and node inputs are immutable. Note that assigning a child node new reaction triggers will not cause the node to re-run, but it *will* cause pending reactions targeting that node to fail.

Child node outputs are *deferred*, which means they cannot be accessed by their parents. However, they *can* be safely accessed by downstream sibling nodes, and child node output handles can be returned by a parent node for use in cousin nodes (but not direct ancestors). In order to perform change detection on data (node state/inputs/outputs) that may contain handles to the deferred outputs of other nodes, the node scheduler carefully orders events so that node outputs will be fully resolved by the time they might be needed for performing change detection in dependents. We compute change hashes using a custom trait that allows inspecting the contents of handles (which are not themselves hashable). This is the 'memoization magic' of `bevy_cobweb`.

A triggered node can cause its parent to rebuild in two scenarios. One is if the node's output changes. The other is if the node errors out. We rebuild parents on error because it's possible that an error was caused by a failure to read an invalid node handle, and so we give the parent an opportunity to repair its children. Parents will recursively rebuild until the algorithm reaches an ancestor whose output doesn't change and that doesn't error out, or until the root node is rebuilt. Root nodes don't have outputs, and they consume propagated errors using their configured error handling policy.

A triggered node can error out either because it directly returned a [`NodeError`](bevy_cobweb::NodeError), or because a node error was propagated up by one of its children. Propagated errors are collected internally in [`WebErrors`](bevy_cobweb::WebError), which are only readable if they propagate to a root node and are consumed by its error handling policy.

Last but not least, child object nodes can be detached from their parents and reattached elsewhere in the web. This is done with [`Packaged`](bevy_cobweb::Packaged) node wrappers that can be sent through node events directly to other nodes (where they can be reattached and rebuilt). A node event is a special feature of `bevy_cobweb` that facilitates web mutations. One significant risk of moving a branch across the web is that if a node in the in-transit branch is triggered by an ECS mutation, then node references in the triggered node may be invalid and the reaction will error-out. To address this, the `bevy_cobweb` scheduler ensures that node events are fully resolved before processing ECS reactions. This allows branches transmitted by node events to safely reattach and then repair any internal node references by rebuilding, before they can be accessed erroneously. Of course, users can always shoot themselves in the foot by improperly handling node references, which is a weakness of this design.


### Plugin

The [`CobwebPlugin`](bevy_cobweb::CobwebPlugin) is the starting point for using `bevy_cobweb`. It provides two configuration options.

- **[NodeErrorPolicy](bevy_cobweb::NodeErrorPolicy)**: The default error handling policy for root nodes (e.g. panic, log and drop, etc.). Root nodes can override the plugin's policy with a different policy (e.g. via `.webroot_with(NodeErrorPolicy::LogAndDrop)`). When an error propagates to a root node, its configured error handling policy will consume the error.
- **[NodeCleanupPolicy](bevy_cobweb::NodeCleanupPolicy)**: The pre-configured node cleanup policy for [`RootNodes`](bevy_cobweb::RootNode) and [`Packaged`](bevy_cobweb::Packaged) nodes that are dropped. When a root or packaged node is dropped, it will be garbage collected, then cleaned up using the configured cleanup policy. Note that root and packaged nodes can never be completely detached to live in the background. You must store them, attach them to other nodes, destroy them, or send them back to the garbage collector.


### Nodes

Every node is a stateful Bevy system that is operated by the web. Nodes have five pieces:

- **Node state**: This is mutable state tied to a specific node. Every time a node runs, it can use this state freely. Node state is initialized by the node builder, and can be overwritten or updated with successive rebuilds. Node state comes in two flavors:
    - **Built-in**: Node state is moved into the node system, which is defined as a closure. This is easy to use, but node state can only be reset.
    - **External**: Node state is defined outside the node system and can be accessed via the [`NodeState<S>`](bevy_cobweb::NodeState) system parameter (which panics on deref if the state is missing). This is less ergonomic, but enables node state merging (see the section on that below) and allows non-closure systems to be used.
- **Node input**: This is immutable data sent to a node by a node builder, and can be changed with successive rebuilds. Every time a node runs, it can read this data via the [`NodeInput<I>`](bevy_cobweb::NodeInput) system parameter (which panics on deref if the input is missing).
- **Node triggers**: These are ECS reaction triggers associated with the node. When a trigger is detected in the `World`, the node will automatically rebuild. Examples: resource mutation, entity component insertion, entity despawn, etc. The node builder specifies node triggers, and can change the node triggers with successive rebuilds (changing triggers does not force the node to run, but it does invalidate all pending reactions targeting that node).
- **Node system**: This is the node's Bevy system. Running a node means running the node system on the Bevy `World`.
- **Node output**: This is the node system's output. Node outputs come in two flavors:
    - **Read-only**: Immutable data that can be read by anyone with a [`NodeHandle`](bevy_cobweb::NodeHandle) to that node. Node handles are produced by built-in basic nodes and object-type [`BasicNodes`](bevy_cobweb::BasicNode).
    - **Consumable**: Single-use data in a [`NodeLink`](bevy_cobweb::NodeLink) that can be use to initialize a chained partner node (produced by producer nodes). Node links are produced by built-in producer nodes and object-type [`ProducerNodes`](bevy_cobweb::ProducerNode).

Node systems are re-runnable 'constructors'. Every time a node runs, it needs to 'reconstruct' all of its child nodes. Children are tracked by `bevy_cobweb` and destroyed if not reconstructed when their parent runs. Node builders use change detection to avoid reconstructing a node if its state initialization or inputs have not changed, which minimizes the work needed to rebuild any node in the web.

#### Node State Example

TODO

#### Node Input Example

TODO

#### Node Triggers Example

TODO

#### Inter-Node Dependencies via [NodeHandle\<O\>](bevy_cobweb::NodeHandle)

Node handles represent a reference to a specific node's output that is stored in the web. They also contain a node id that can be used to send node events to the referenced node.

The data in a node handle is not readable while the node is building. This means node handles are only readable within sibling and cousin nodes that are built downstream of the handle origins. The are *not* readable by parents and direct ancestors, which always finish running before their children.

TODO

#### Node Chaining via [NodeLink\<O\>](bevy_cobweb::NodeLink)

If you need node state initialization to be computed with a complex process, rather than embed that logic in the node that uses the state, you can use node chaining. The producer node will output a [`NodeLink<O>`](bevy_cobweb::NodeLink), which is a consumable handle. Another node can then 'connect' to the producer by consuming the `NodeLink` when updating their node state.

Producer nodes can only connect to sibling nodes with the same parent. Once a producer is connected to another node, it cannot be connected to any other node until the connected node is destroyed or detached.

A producer only sets a new value to its `NodeLink` when rebuilt. If a producer is disconnected and has an empty `NodeLink` (i.e. because its previous connection consumed the link, and the link was not refreshed by a rebuild), then the next time its link handle is used to connect another node it will be force-rebuilt. This makes it easy to seamlessly rearrange node chains.

A consumer node can only consume one `NodeLink` and no other values when updating their node state. Unlike other aspects of the web design, `NodeLinks` do not use change detection because they can transmit non-hashable data (e.g. [`Packaged`](bevy_cobweb::Packaged) nodes). Whenever a non-empty node link is consumed, the consumer will always rebuild.

TODO

#### Node State Merging

Often it's useful to incrementally update a node's state rather than completely reset it. In that case instead of setting node state in the node builder, you can merge existing state with initialization data (which may be data from the parent node, or a [`NodeLink`](bevy_cobweb::NodeLink)).

Nodes that allow state merging must use the [`NodeState<S>`](bevy_cobweb::Node) system parameter to access the node state.

TODO

#### Change Detection

- node building as deferred mutation, change detection as carefully ordered data hashing

- state initializers (normal, merged, normal/merged + NodeLink)
- node triggers
- node input
- node output

TODO


### [NodeBuilder](bevy_cobweb::NodeBuilder)

All nodes are constructed using the [NodeBuilder](bevy_cobweb::NodeBuilder).

TODO

#### Built-in Nodes

TODO

**Reinitialization**

TODO

#### [BasicNode](bevy_cobweb::BasicNode)

TODO

#### [ProducerNode](bevy_cobweb::ProducerNode)

TODO

#### [ReactorNode](bevy_cobweb::ReactorNode)

TODO

#### [Packaged](bevy_cobweb::Packaged) Nodes

A [`PackagedNode`](bevy_cobweb::PackagedNode) is a node in the web with no parent. Packaged nodes cannot be reinitialized, but they can be moved anywhere.

A `PackagedNode` can be scheduled to rebuild at any time (if you give it changed inputs), but does *not* produce a [`NodeHandle`](bevy_cobweb::NodeHandle), since handle access scoping is relative to a specific position within the web (whereas `PackagedNodes` have a 'floating' position). If you want to get a handle, then attach the node to another node to get a [`AttachedNode`](bevy_cobweb::AttachedNode) or [`ProducerNode`](bevy_cobweb::ProducerNode) before building it.

If a `PackagedNode` is dropped, then it will be sent to the `bevy_cobweb` garbage collector, where the preconfigured [`NodeCleanupPolicy`](bevy_cobweb::NodeCleanupPolicy) will decide what to do with it. Using a garbage collector makes it relatively safe to transfer `PackagedNodes` around your application (e.g. sending them between parts of the web through node events), since you won't be at risk of dangling nodes.

TODO


### Node Events

TODO


### ECS Reactivity

In `bevy_cobweb`, ECS reactivity is implemented through [`ReactCommands`](bevy_cobweb::ReactCommands). We use custom reactivity instead of Bevy change detection in order to achieve precise, responsive, recursive reactions with an ergonomic API that correctly integrates with `bevy_cobweb`'s node building protocol. In an ideal world `bevy_cobweb` would be upstreamed to Bevy, which would eliminate the ergonomic limitations of custom reactive elements (i.e. `ReactRes<>` resources and `React<>` components).

See the [docs](bevy_cobweb::react) for more details (WILL BE MIGRATED FROM `BEVY_KOT`, SEE [THE DOCS](https://github.com/UkoeHB/bevy_kot/tree/master/bevy_kot_ecs) THERE).

#### Example

TODO

#### Scheduler: Four-Tier Commands Framework

A foundational component of `bevy_cobweb` is a four-tier commands framework (aka the scheduler) that enables recursive rebuilds and reactions.

TODO

- bevy commands
- system commands
- system events
- reaction commands


### Error Propagation

- node errors vs accumulated web errors
- system command: ignore failing children (clear error queue)

TODO



## `bevy` compatability

| `bevy` | `bevy_cobweb` |
|-------|----------------|
| 0.12  | 0.1 - master   |
