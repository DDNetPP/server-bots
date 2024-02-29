# server-bots
Nothing here yet. Just an idea.

## pre alpha

APIs might change. The repository might move. The project might never be started.
The project might be renamed (I do not like the current name "server-bots" it is too generic)

## the idea

Have a well defined api for server side teeworlds bots:
- It should be easy to setup (less than 10 lines of code in mostly one place).
- It should be highly portable and work in basically any teeworlds or ddnet code base.
- It should be highly debuggable.
- It should be stable. As in versioned and old bots should still work in 10 years.
- It should be blazingly fast in production.
- It should be coverable by 100% unit tests.
- It should not depend on state.
- It should be hot reloadable.

## implementation details user facing api

The general idea is that the whole api consists of one function call.
And it is being passed in a bunch of pointers. A few read only ones for the current world state.
And one output buffer for the current bot state given the world state. Consider the following pseudo code.

```C++

#include <ddnetpp/server-bots.h>

void CCharacter::OnTick()
{
  CServerBotState State;
  server_bots(this, &State);

  m_Direction = State.m_Direction;
  m_Hook = State.m_Hook;
  m_Fire = State.m_Fire;
  m_Jump = State.m_Jump;
}
```

The api promises that it does never keep any internal state accross ticks.
All the state it needs can be extracted from the world.
The api operates on a per tick level. With one input and one output.
There is no IO or any other side effects happening in the api.

These limitations make developing bots a bit more tricky. But it allows to fullfill all the goals.
It allows for better portability. It allows for clean unit test setups. And it allows to hot reload without breaking state.

## implementation details debuggability

While writing such bot it can happen that for example m_Direction is set in one if statement.
And then overwritten by another one. In the end there is only ever one value that is the output of the current tick.
If it is not the expected value it is hard to figure out who set that value.
So all sets of those output variables should be wrapped in a macro.
And the macro needs a reason that will be added to some list that can be dumped for debugging.
In release mode the macro should just compile to a raw integer set.

Consider the following pseudo code of how it has been done so far:

```C++
m_Direction = 0;
if(true) {
  m_Direction = -1;
}
// condition nobody expected to be true
if(true) {
  m_Direction = 1;
}
```

And how the code should be from now on:

```C++
DIR(0, "initial value");
if(true) {
  DIR(-1, "go left because xyz");
}
// condition nobody expected to be true
if(true) {
  DIR(1, "go right because super edge case");
}
```

Which in the end compiles to the old code in release mode and the comments will be ignored.
But in debug mode those comments should either be printed or added as a list to the output struct for inspection.

This ensures that you can always obtain the "why" for the current state. And also see which movement values got overwritten.
