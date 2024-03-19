# server-bots
Nothing here yet. Just an idea.

## pre alpha

APIs might change. The repository might move. The project might never be started.
The project might be renamed (I do not like the current name "server-bots" it is too generic)

## the idea

Have a well defined api for server side teeworlds bots:
- It should be easy to setup (less than 10 lines of code in mostly one place).
- It should be highly portable and work in basically any teeworlds or ddnet code base.
- It should be highly debuggable via logging.
- It should be highly debuggable via visual feedback (api for drawing text and lines at coordinates)
- It should be stable. As in versioned and old bots should still work in 10 years.
- It should be blazingly fast in production.
- It should be coverable by 100% unit tests.
- It should not depend on state.
- It should be hot reloadable.

The whole thing would be pure, without side effects. There is no networking. There is no IO.
There is no context needed of what happend before or after. It is basically turn based teeworlds.
Given the current inputs/velocity/world what is the best move for this tick.

Sample of a current implementation of a bot that should be outdated by this new api.

https://github.com/DDNetPP/DDNetPP/blob/72aa916964f4c1714463314ea9b6b5e894fda807/src/game/server/entities/dummy/blmapchill_police.cpp

## terms

There are two parties here. The implementations of both sides of the api call.
Each have different responsibilities. Those two parties should have catchy and clear names.

## implementation details user facing api

The general idea is that the whole api consists of one function call.
And it is being passed in a bunch of pointers. A few read only ones for the current world state.
And one output buffer for the current bot state given the world state. Consider the following pseudo code.

```C++

#include <ddnetpp/server-bots.h>

void CCharacter::OnTick()
{
  ServerBotState State;
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

## implementation details debuggability via logging

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

## implementation details debuggability via visual feedback (api for drawing text and lines at coordinates)

An implementation of a server bot should be able to give visual feedback in addition to logging for debugging.
There should be a well defined api similiar to the logging one that allows for the server bot code to draw lines.
And render text at certain locations. Or render colored simple shapes such as recentangles.
It is then up to the implementation consuming the server bot to actually do the drawing.
This requires writing some code but it is a optional feature if not implemented the bot will still work the same so that is fine.
There could be a sample implementation that does line drawing via lasers and text via broadcast or loltext.
And colored rectangles with different types of pickups or colored lasers from ddnet.

## draft of the required input

Ideally something that is easy to pass from a teeworlds code base.
Such as a pointer to a CCharacter or ICollision.
But it should also not require the bot library to link against an entire teeworlds code base.

Maybe a macro can be used to translate a CCharacter pointer into a struct defined in the library.
The macro could be called like this by the user:

```C++
void CCharacter::OnTick()
{
  // prep state
  ServerBotCharacter Char;
  FILL_STATE_CHARACTER(this, Char);

  // pass state to api
  ServerBotState State;
  server_bots(this, &State, &Char);
}
```

And the FILL_STATE_CHARACTER expands to something like this
```C++
  Char.m_Vel = m_Vel;
  Char.m_Weapon = m_Weapon;
  Char.m_Health = m_Health;
  // ..
```

And ServerBotCharacter could be defined as a struct in the library like so
```C++
struct ServerBotCharacter {
  int m_Vel;
  int m_Weapon;
  int m_Health;
  // ..
}
```

The draw back of not passing a CCharacter pointer directly but using that struct is that it requires much more byte copying. So it becomes a way bigger performance overhead than just passing a pointer to existing memory.

The bot has to know the following things (basically everything gameplay relevant).
All players in the current world and their state (position, velocity, weapons).
All projectiles in the world and their state.
The current map:
- map name as string
- map version as int if assumptions are made based on map name
- the actual map data of all gameplay relevant layers: game, front, tune, speedup, switch

## draft of the output struct

```C++
struct ServerBotState {
  // all controls walk/hook/jump/fire/emote/weapon...

  // either a capped array or vector for the debug statements (one per each control)
  // if it is an array then the control setting macros need to be a ring buffer
  // ensuring that it always contains the latest x sets
}
```
