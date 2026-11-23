---
isCurrent: false
---
## The Mental Model: The Central Whiteboard

Think of a Godot project like a high school classroom:

- **Local Scripts (`Player.gd`, `Coin.gd`, `HUD.gd`)** are like individual students holding their own notebooks. They only easily see what’s on their own desk.
    
- **The Global Script (`Global.gd`)** is the big whiteboard at the front of the classroom. Any student can walk up, read a number off the board, or write an updated number for everyone else to see
## Data Flow Architecture

Here is how data moves between scene nodes using a Global Autoload script as the central mediator.

![[data flow diagram example.jpg]]

