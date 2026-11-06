# FC Syntax Reference

This is an agent-facing syntax reference derived from `FCGLexer.g4` and `FCGParser.g4`.

Use this file only for FC grammar and common syntax forms. For APIs, types, events, enums, components, generated project symbols, imports, and package symbols, use `fc-symbol-lookup`. For assetIds, entityIds, UI objects, script attachments, button bindings, and current editor state, use `fc-asset-check`.

## File Forms

`.fcg` graph files:

```fcg
import "StdLibrary.fcc" as StdLib
import OtherScript as Other from "./OtherScript.fcg"

graph MyGraph {
    count int

    func Start() {
    }
}
```

`.fcc` header files contain imports followed by declarations:

```fcg
import "StdLibrary.fcc" as StdLib

define MyType: int
event MyEvent(value int)
func Helper(value int) int
```

## Imports

Library import:

```fcg
import "StdLibrary.fcc" as StdLib
```

Script reference import (`as Alias` is optional):

```fcg
import ScriptName as Alias from "./ScriptName.fcg"
import ScriptName from "./ScriptName.fcg"
```

## Header Declarations

Type definition:

```fcg
define Health: int
```

Function point type:

```fcg
define Callback: func(player entity<Player>)
```

Alias:

```fcg
alias NumberLike = int | float
alias EntityAndTransform = entity<Player> & Transform
```

Enum:

```fcg
enum Team {
    Red = 1
    Blue = 2
}
```

Tuple:

```fcg
tuple SpawnInfo {
    position Vector3
    yaw float
}
```

Component:

```fcg
component CustomState {
    hp int
    name string
}
```

Event declaration:

```fcg
event PlayerScored(player entity<Player>, score int)
```

Function declaration:

```fcg
func ClampScore(score int) int
```

Declare graph:

```fcg
declare graph EnemyAI {
    target entity<Player>
    func Reset()
}
```

## Decorators

Decorator forms:

```fcg
[readonly]
speed float

[accept entity<Player>]
define PlayerEntity: entity<Player>

[element int]
enum ScoreType {
    Normal = 1
}

[bridging]
component BridgeComponent {
}

[combine Transform, Player]
component PlayerView {
}
```

Custom decorators use bracketed identifiers:

```fcg
[platform_client]
graph ClientGraph {
}
```

## Graph Body

Graph declaration:

```fcg
graph MyGraph {
    score int
    enabled bool = true

    func Start() {
    }

    event Game.Start() {
    }
}
```

Graph members are properties, functions, or event listeners.

Property forms:

```fcg
score int
name string = "player"
team = Team.Red
```

## Function Signatures

Function syntax uses `name Type` parameters and return type after the parameter list:

```fcg
func Add(a int, b int) int {
    return a + b
}
```

Async function:

```fcg
async func LoadData(player entity<Player>) {
}
```

Out parameter:

```fcg
func TryGetScore(out var score int) bool {
    return true
}
```

Anonymous function:

```fcg
var callback = func(player entity<Player>) {
}
```

## Statements

Local variable:

```fcg
var count int
var name string = "abc"
var pos = Vector3{0, 0, 0}
```

Assignment and self modification:

```fcg
count = 1
count += 2
count++
count--
```

Function call:

```fcg
DoSomething(player)
```

Async call:

```fcg
start DoSomething(player)
wait DoSomething(player)
```

Out argument:

```fcg
TryGetScore(out var score)
```

If / else:

```fcg
if count > 0 {
    score += count
} else if count == 0 {
    score = 0
} else {
    score = -1
}
```

Index for loop:

```fcg
for i = 0, count, 1 {
}
```

Range for loop:

```fcg
for i, item in items {
}
```

While:

```fcg
while count > 0 {
    count--
}
```

Return / break / continue:

```fcg
return value
break
continue
```

## Expressions

Cast:

```fcg
value as int
```

Call expression:

```fcg
Math.Min(a, b)
```

Generic or component access:

```fcg
thisEntity<Transform>.Position
player<Player>.Name
```

Field access:

```fcg
position.X
```

Index access:

```fcg
items[i]
```

Type expression:

```fcg
typeof(entity<Player>)
```

Operators supported by grammar:

```fcg
!flag
~mask
-value
a * b / c % d
a + b - c
a << b
a >> b
a & b
a ^ b
a | b
a == b
a != b
a < b
a <= b
a > b
a >= b
a && b
a || b
```

## Types

Primitive and built-in types:

```fcg
object
bool
int
int64
float
string
Vector2
Vector3
Quaternion
LocString
```

Generic types:

```fcg
entity<Player>
List<int>
Map<string, int>
List<entity<Player>>
```

Namespaced type:

```fcg
StdLib.SomeType
```

Function point type:

```fcg
func(player entity<Player>) bool
```

## Literals

Basic literals:

```fcg
nil
true
false
123
0xFF
1.5
"text"
`raw text`
#FF00FF
```

Vector literals:

```fcg
Vector2{1, 2}
Vector3{1, 2, 3}
Quaternion{0, 0, 0, 1}
```

List literal:

```fcg
List<int>{1, 2, 3}
```

Map literal:

```fcg
Map<string, int>{"a": 1, "b": 2}
```

Tuple literal:

```fcg
SpawnInfo{position = Vector3{0, 0, 0}, yaw = 0}
```

LocString literal:

```fcg
LocString{"key.name", List<object>{playerName}}
```

## Common Wrong Patterns

Do not use TypeScript or C# parameter syntax:

```fcg
// Wrong
func Move(player: entity<Player>) {
}

// Correct
func Move(player entity<Player>) {
}
```

Do not put the return type before the function name:

```fcg
// Wrong
int GetCount() {
    return 1
}

// Correct
func GetCount() int {
    return 1
}
```

Do not write graph fields as local variables:

```fcg
// Wrong
var score int

// Correct
score int
```

Do not use constructor-call syntax for vectors:

```fcg
// Wrong
Vector3(1, 2, 3)

// Correct
Vector3{1, 2, 3}
```

Do not call async syntax after the function name:

```fcg
// Wrong
DoSomething(player).wait()

// Correct
wait DoSomething(player)
```

Do not put platform decorators on graph-body event listeners. For client-only or server-only event handling, put the platform decorator on the graph:

```fcg
// Wrong
graph ClientGraph {
    [platform_client]
    event UI.ButtonClicked(button entity<UIWidgetButton>, player entity<Player>) {
    }
}

// Correct
[platform_client]
graph ClientGraph {
    event UI.ButtonClicked(button entity<UIWidgetButton>, player entity<Player>) {
    }
}
```

Do not use package cache paths as imports:

```fcg
// Wrong
import "Temp/UGCLanguage/packages/libId/Some.fcc" as Some

// Correct
import "Some.fcc" as Some
```
