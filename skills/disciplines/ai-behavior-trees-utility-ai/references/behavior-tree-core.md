# Behavior Tree core — Blackboard, nodes, leaves, composites, decorators

Depth for `ai-behavior-trees-utility-ai`. A complete, allocation-conscious C# behavior-tree
runtime. The code targets plain C# (no engine types) so it drops into Unity 6 or Godot 4 C#
unchanged — bind the leaves to your engine's transform/navigation in the concrete actions.

Contract in one line: every node's `Tick` returns `Success`, `Failure`, or `Running`, and a
parent decides what that means. Build the tree once at spawn; keep per-tick work allocation-free.

## Blackboard — the shared state system

The **Blackboard** is the agent's working memory. Leaves read and write it; no node holds a
reference to another. That indirection is what lets the same `MoveTo` action serve chase, patrol,
and flee subtrees, and what makes subtrees reusable across enemy types.

```csharp
using System.Collections.Generic;

// A typed key/value store. Keys are strings (or use enum/int keys to avoid hashing cost).
public sealed class Blackboard
{
    private readonly Dictionary<string, object> _values = new();

    public void Set<T>(string key, T value) => _values[key] = value!;

    public T Get<T>(string key, T fallback = default!)
        => _values.TryGetValue(key, out var v) && v is T t ? t : fallback;

    public bool TryGet<T>(string key, out T value)
    {
        if (_values.TryGetValue(key, out var v) && v is T t) { value = t; return true; }
        value = default!;
        return false;
    }

    public bool Has(string key) => _values.ContainsKey(key);
    public void Remove(string key) => _values.Remove(key);
}
```

For hot agents, prefer a **struct-of-fields blackboard** (public fields on a class) over a string
dictionary — it removes hashing and boxing entirely. Use the dictionary form when designers add
keys at runtime or you serialize the board.

```csharp
// Zero-allocation alternative: a plain data object shared by every node on this agent.
public sealed class AgentContext
{
    public Vector2 Position;
    public Transform Target;      // null when no target
    public Vector2 Home;
    public float LastSeenTime;
    public readonly List<Vector2> Path = new();
}
```

## The node base and status

```csharp
public enum Status { Success, Failure, Running }

// Every node in the tree derives from Node. dt is the decision-step delta (may differ from frame dt).
public abstract class Node
{
    public abstract Status Tick(Blackboard bb, float dt);

    // Called when a parent stops running this node before it finished (aborted branch).
    // Override to release timers, animations, or reservations.
    public virtual void Reset() { }
}
```

`Reset()` is the half of the contract people forget. When a `Selector` switches from a running
low-priority branch to a higher-priority one, it must `Reset()` the abandoned branch so a
half-finished action (a playing attack animation, a reserved cover point) is cleaned up.

## Base classes: leaf, composite, decorator

Three structural node kinds cover every tree:

```csharp
// A leaf does the actual work; it has no children.
public abstract class Leaf : Node { }

// A composite has many children and defines how their statuses combine (see Composites below).
public abstract class Composite : Node
{
    protected readonly List<Node> Children = new();
    protected int Current;                       // resume index for Running composites

    public Composite Add(Node child) { Children.Add(child); return this; }

    public override void Reset()
    {
        Current = 0;
        foreach (var c in Children) c.Reset();
    }
}

// A decorator wraps exactly one child and transforms its status or gates it (see Decorators below).
public abstract class Decorator : Node
{
    protected Node Child = default!;
    public Decorator Wrap(Node child) { Child = child; return this; }
    public override void Reset() => Child.Reset();
}
```

## Condition leaves — instantaneous predicates

A **condition** reads the blackboard and returns `Success` or `Failure` in the same tick. It never
returns `Running` and never mutates game state. Express reusable predicates as one small class with
an injected test, or subclass for named conditions.

```csharp
// Generic condition: succeed when a predicate over the blackboard holds.
public sealed class Condition : Leaf
{
    private readonly System.Func<Blackboard, bool> _predicate;
    public Condition(System.Func<Blackboard, bool> predicate) => _predicate = predicate;

    public override Status Tick(Blackboard bb, float dt)
        => _predicate(bb) ? Status.Success : Status.Failure;
}

// Named condition when the check is non-trivial or reused across trees.
public sealed class CanSeeTarget : Leaf
{
    private readonly float _sightRange;
    public CanSeeTarget(float sightRange) => _sightRange = sightRange;

    public override Status Tick(Blackboard bb, float dt)
    {
        if (!bb.TryGet<Vector2>("targetPos", out var target)) return Status.Failure;
        var self = bb.Get<Vector2>("position");
        // Real games also raycast for line of sight; keep the check cheap and cache the result.
        return (target - self).sqrMagnitude <= _sightRange * _sightRange
            ? Status.Success : Status.Failure;
    }
}
```

Note the pre-allocated `System.Func` delegate: create conditions **once** when the tree is built,
not per tick, so no closure is allocated during evaluation.

## Action leaves — multi-frame work that returns Running

An **action** performs work and returns `Running` until it completes, then `Success` (or `Failure`
if it can't). Returning `Running` is what lets a walk, an animation, or a timed wait span many
ticks without the parent restarting it.

```csharp
// Move toward a blackboard target; Running until within arrive radius, then Success.
public sealed class MoveToTarget : Leaf
{
    private readonly float _speed, _arriveRadius;
    public MoveToTarget(float speed, float arriveRadius)
    { _speed = speed; _arriveRadius = arriveRadius; }

    public override Status Tick(Blackboard bb, float dt)
    {
        if (!bb.TryGet<Vector2>("targetPos", out var target)) return Status.Failure;
        var pos = bb.Get<Vector2>("position");
        var offset = target - pos;
        if (offset.magnitude <= _arriveRadius) return Status.Success;   // arrived

        pos += offset.normalized * _speed * dt;
        bb.Set("position", pos);          // in an engine, drive the NavMeshAgent / CharacterBody here
        return Status.Running;            // keep going next tick
    }
}

// A timed wait — the canonical Running action, useful for patrol pauses and cooldown holds.
public sealed class Wait : Leaf
{
    private readonly float _duration;
    private float _elapsed;
    public Wait(float duration) => _duration = duration;

    public override Status Tick(Blackboard bb, float dt)
    {
        _elapsed += dt;
        if (_elapsed < _duration) return Status.Running;
        return Status.Success;
    }

    public override void Reset() => _elapsed = 0f;   // restart cleanly if the branch is re-entered
}
```

`Wait` shows why `Reset()` matters: its `_elapsed` accumulator must be cleared when the branch is
abandoned and later re-entered, or the second wait finishes instantly.

The `Sequence`/`Selector`/`Parallel` execution logic and the decorator library build directly on
these base classes — see the composites and decorators sections below.
