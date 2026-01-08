# DOTS Framework – Entity Template & LinkedObject System

> ⚠️ **Work in Progress**
> This framework is actively under development. Some features are experimental, limited, or subject to change.
> Performance characteristics and APIs may evolve.

---

## Table of Contents

* [Overview](#overview)
* [Core Concepts](#core-concepts)
  * [Entity Templates](#entity-templates)
  * [Entity Instances](#entity-instances)
  * [Batched Execution](#batched-execution)
* [Editor Tooling & Code Generation](#editor-tooling--code-generation)
* [Entity Creation & Registration](#entity-creation--registration)
* [Runtime EntityBuilder (Experimental)](#runtime-entitybuilder-experimental)
* [Static Framework API](#static-framework-api)
* [LinkedObjects](#linkedobjects)
* [Template Attributes](#template-attributes)
* [Queries](#queries)
* [Supported Data Types](#supported-data-types)
* [Data Access (Getter / Setter API)](#data-access-getter--setter-api)
* [Performance Guidelines](#performance-guidelines)

---

## Overview

This framework provides a **template-based abstraction layer** on top of **Unity DOTS (Entities)**.

Key goals:

* Define **Entity Archetypes (Templates)** once
* Instantiate entities cheaply and consistently
* Control execution through **attribute-driven queries**
* Share data safely between **GameObject world and DOTS** using **LinkedObjects**
* Automatically batch structural changes at the end of the frame
* handles all scheduling for maximal parallelism

---

## Core Concepts
The framework simplifies the complexity of Unity DOTS into an intuitive workflow based on three pillars:

1. **Templates (Blueprints)**: Instead of manually assembling entities with an `EntityManager`, you register templates. These define the component layout (archetype) and allow you to manage instantiation consistently.


2. **LinkedObjects (The Bridge)**: This solves the primary DOTS challenge: communication with standard MonoBehaviours. You define static data structures that both GameObjects and DOTS jobs can access safely and performantly.


3. **Queries (Logic)**: You don't need to write complex `SystemBase` classes. Instead, you use static methods with attributes like `[UpdateTemplate]`. The framework automatically handles scheduling and parallelism as jobs.

### Entity Templates

An **Entity Template** represents an **archetype**, not an instance.

* Templates are registered once
* Components can be added before instantiation
* Instantiation clones the template archetype

Templates are identified by:

* `int ID`
* or `string name` (internally hashed)

---

### Entity Instances

When a template is instantiated:

* A new entity is created
* A **SubID (ulong)** is assigned
* The SubID is used to destroy the instance later

---

### Batched Execution

All framework operations are **queued** and executed **at the end of the frame**:

* Entity registration
* Component additions
* Instantiation
* Destruction

✅ This avoids structural changes during query execution  
⚠️ This means **results are not immediate** (runs in the LateUpdate)

---
Hier ist der bereinigte Part für die Editor-Tools, direkt integriert in ein Layout, das ohne Tabellen auskommt und stattdessen auf Scannbarkeit durch Bullet-Points und fette Markierungen setzt:

---

## Editor Tooling & Code Generation

The framework automates the boilerplate for Templates and LinkedObjects through background code generation. Manage these tools via the **DotsFramework** top-level menu.

### Menu Commands

* **Framework Compilation** The master switch. Enables or disables the entire code generation system.
* **Auto Compilation** Automatically triggers a recompile when changes are detected in your scripts.
  *Requires "Framework Compilation" to be active.*
* **Recompile** Force-cleans the cache and triggers a fresh compilation. Use this if your IDE (VS/Rider) doesn't recognize newly added LinkedObject fields.
* **Clear** Wipes all generated files and resets the internal framework state.

---

## Entity Creation & Registration

### Creating Templates

```csharp
Framework.RegisterEntity("Player", playerTemplate);
Framework.RegisterEntity(2, enemyTemplate);
```
> ⚠️ Strings will be hashed internally. When using mixed id types, it may **lead to duplicates**
---

### Adding Components to Templates

```csharp
Framework.AddComponent(1, new Health { Value = 100 });
Framework.AddComponent(2, new Speed  { Value = 5f });
```

> ⚠️ Adding components **changes the archetype** of the registered template  
> ⚠️ Already instantiated entities **won't change**

---

## Runtime EntityBuilder (Experimental)

```csharp
new EntityBuilder().BuildEntity(gameObject, entityID);
```

### Status

* ⚠️ **Experimental**
* ⚠️ Physics components do **not work correctly**
* ⚠️ **High performance cost**

### Recommendation

> ❌ **Do NOT use in performance-critical code**  
> ✅ Prefer **Bakers**

### Authoring & Baking
The `RegisterAsEntity` Authoring Component allows you to integrate directly with your Baker. By adding this component to your GameObject, the resulting baked entity is registered automatically, removing the need for manual registration in code.

---

## Static Framework API

All interactions go through the static `Framework` class.

---

### RegisterEntity

```csharp
Framework.RegisterEntity(int id, Entity entity);
Framework.RegisterEntity(string name, Entity entity);
```

Registers an entity archetype as a template.

---

### AddComponent

```csharp
Framework.AddComponent<T>(int id, T component);
Framework.AddComponent<T>(string name, T component);
```

* Adds a component to a **registered template**
* Changes its archetype
* `T` must be `unmanaged` and `IComponentData`

---

### Instantiate

```csharp
Framework.Instantiate(int id);
Framework.Instantiate(string name);

Framework.Instantiate(int id, LocalTransform transform);
Framework.Instantiate(string name, LocalTransform transform);
```

* Creates an instance of the template
* Optional transform initialization
* returns unique identifier of instance (**subID**)

⚠️SubID is only unique in same session. Resets when the framework resets.

---

### Destroy

```csharp
Framework.Destroy(ulong subID);
```

Destroys a previously instantiated entity **by subID**.

---

## LinkedObjects

LinkedObjects allow **data sharing between GameObject world and DOTS**.

They are:

* Declared as public static fields
* Accessed in Burst-compatible DOTS queries
* Automatically synchronized every frame in the LateUpdate

---

### Declaration (GameObject World)

```csharp
[LinkedObject(1)]
public static PlayerDataStruct PlayerData;
```

```csharp
[LinkedObject("EnemiesInRange")]
public static List<ulong> EnemiesInRange;
```

> ⚠️ LinkedObjects **must be defined before the first Update**  
> ⚠️ Strings will be hashed internally. When using mixed id types, it may **comes to duplicates**


---

## Template Attributes

These attributes control **when to run functionality for which templates**.
* Runs in the LateUpdate
* Runs only for the specified template IDs
* If **no IDs are provided**, runs for **all entities matching the query**
* ✅Fewer IDs → less exclusion → better performance
---

### `[UpdateTemplate]`

```csharp
[UpdateTemplate(1, 2)]
```

* Called **every frame**
* **First** template that runs

---

### `[StartTemplate]`

```csharp
[StartTemplate("Player")]
```

* Called **once** when an entity instance is created
* Calls in the same frame, the instance is created
* **Last** template that runs

---

### `[EndTemplate]`

```csharp
[EndTemplate(2)]
```

* Called when an entity instance is destroyed
* Calls in the same frame, the instance is destroyed
* **Second** template that runs

---

## Queries

Queries are the core logic units of your application. They define which code should run for specific entities and provide access to both DOTS component data and global `LinkedObjects`.

### Query Structure

Every query is defined as a `static void` method. The framework automatically detects these methods and schedules them within the Unity Job System.

```csharp
[BurstCompile]
[UpdateTemplate(PlayerID)] // Filters execution to this specific template
public static void PlayerMovement(
    ref EntityID entityID,                                              // Metadata (contains SubID and TemplateID)
    ref LocalTransform transform,                                       // Standard DOTS component (Read/Write)
    [LinkedObject("PlayerData")] ref LinkedObjectData playerData,       // Read-only global data (Burst-compatible)
    [LinkedObject("DeltaTime", true)] ref LinkedObjectData deltaTime    // Read-only deltaTime
)
{
    // Access DOTS components directly

    // Access LinkedObject data via the API

    // Perform your logic here...
}
```


### Attributes:
### `[BurstCompile]`
* Enables the Unity Burst Compiler for high-performance machine code execution.
### `[UpdateTemplate(ID)]`
* Restricts the query to entities belonging to specific templates. If no ID is provided, the query runs on all entities that possess the required components.
* **`[StartTemplate]` / `[EndTemplate]`**: Use these instead of `UpdateTemplate` to run logic only when an entity is created or destroyed.

### Parameter Types (all optional):

### Components (`ref T`)
* Direct access to standard DOTS `IComponentData`. You can include as many components as needed.
### `ref EntityID`
* Provides the unique `SubID` of the instance. Available on every instance.
### `[LinkedObject]`
* Bridges the gap to the GameObject world. It allows you to pull in global data or shared structures.

### Parallelism:

The framework automatically handles job scheduling and dependency management:

* **Thread Safety**: If multiple queries try to write to the same `LinkedObject`, they are scheduled sequentially to prevent data corruption.
* **ReadOnly Optimization**: By setting the second parameter of `[LinkedObject]` to `true`, you mark the data as read-only. This allows the framework to run multiple queries in parallel.

> ⚠️ **Parallel Execution**: A query is only scheduled in parallel if **all** LinkedObjects in its parameters are marked as **ReadOnly**

---

## Supported Data Types

### Primitive Types

* `float`, `double`
* `short`, `int`, `long`
* `ushort`, `uint`, `ulong`
* `byte`

---

### Math Types

* `Vector2` / `float2`
* `Vector3` / `float3`
* `Vector4` / `float4`

---

### Strings

* `string`
* `FixedStringXXBytes`

⚠️Strings are all turned to size of 128 bytes internally

---

### Containers

* `Array`
* `List`

⚠️Containers can be nested, but depending on datatype it may introduce to some extra overhead


---

### Unsupported Types

Any unsupported type is:

* Automatically **flattened**
* Only **public fields** are considered

⚠️ Nested reference types are discouraged

---

## Data Access (Getter / Setter API)

All LinkedObject data is accessed via generic functions.

---

### Get

```csharp
playerData.Get(out int value, key, index);
```

---

### Set

```csharp
playerData.Set(value, key, index);
```

---

### Add (Lists)

```csharp
playerData.Add(value, key, index);
```

---

### Remove

```csharp
playerData.Remove(at, key, index);
```

---

### Length

```csharp
playerData.Length(key, index);
```

---

### Parameters

* `key` → field name (`FixedString32Bytes`)
* `index` → nested access path (`FixedList64Bytes<int>`)

```csharp
public struct PlayerData
{
    public int Health;
    public Skill[] Skills
}
public struct Skill
{
    public string Name;
}

playerSpeed.Get(out float speed); // no need for key if access is clearly defined (all but struct/class)

playerData.Get(out int health, "Health");

playerData.Get(out FixedString128Bytes skillName, "Skills.Name", new FixedList64Bytes<int>() {0}); // Gets first Index of Skills
```
---

## Performance Guidelines

### LinkedObjects

* Complex LinkedObjects → more overhead
* Data is synced **twice per frame**
* LinkedObject sync cost scales with data size
---

### Lists

❌ Avoid:

```csharp
List<Class>
```

✅ Prefer:

```csharp
List<int>
List<float3>
```

---

### Queries & Parallelism

* Queries with **no overlapping write access** run in parallel
* ReadOnly LinkedObjects are ignored for conflict checks
* Fewer query parameters → higher chance of parallel execution

---

### Method Count

* Each method has scheduling overhead
* Merge logic where possible

---

### LinkedObject Data Layout

* Use many **different** data types
* Many **identical** data types are slower.
* Multiple same data types cause **switch-based access**
* single data type has **direct access**

❌ Avoid:

```csharp
public struct Data
{
    public float Length
    public float Height
}
```

✅ Prefer:

```csharp
public struct Data
{
    public float Length
    public double Height
}
```
⚠️ Only noticeable in **larger** structures

---

### Advanced Optimization

You may bypass Getter/Setter by directly accessing generated fields.  

* Faster
* More flexible
* ⚠️ Code generation may change interfaces
```csharp
playerData.Get(out FixedString128Bytes skillName, "Skills.Name", new FixedList64Bytes<int>() {0}); 

playerData.Skills[0].Name; //⚠️Only valid with generated code
```

You also can look into the generated files yourself to see what happens.
