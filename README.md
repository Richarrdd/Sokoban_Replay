As an example, consider the following arguments to the main method:

```
java -jar Sokoban.jar 3 map02.map FREE_RACE 60 actions0.txt actions1.txt
```
It means that the game will take `map02.map` as `GameMap`.
Suppose there are two players in the game map, `actions0.txt` and `actions1.txt` files specifies the actions of the two players, respectively.
The number `3` means that the game will be repeatedly replayed for 3 times in parallel.
The remaining two arguments `FREE_RACE` and `60` are the game replay mode and the frameRate (frames per second) of rendering, respectively.
They are to be explained in [Multithreading Architecture](#Multithreading in ReplaySokobanGame).

The `map02.map` may look like this:
```text
2
########
#..@..@#
#A..a..#
#..a#@.#
#@.a#..#
#.b..B.#
########
```
The `actions0.txt` may look like this:
```text
0
J
K
L
L
U
L
H
H
J
E
```
The `actions1.txt` may look like this:
```text
1
J
K
L
U
L
U
K
E
```

#### Action File
The first line of an action file specifies the player ID.
All actions in the action file are performed by this player.
The following lines specify the actions of the player.
The actions are represented by the following characters:
- `H`: move the player left
- `J`: move the player down
- `K`: move the player up
- `L`: move the player right
- `U`: undo
- `E`: exit

### Multithreading Architecture

#### Thread-Safe `ReplaySokobanGame`

The `ReplaySokobanGame` class should be [thread-safe](https://en.wikipedia.org/wiki/Thread_safety).
Each game instance should be able to run in parallel with other game instances in different threads.
The `ReplaySokobanGame` implements `Runnable`, and it will execute in a thread as implemented in `hk.ust.ust.comp3021.SokobanGame#replayGame`.
In the example [above](#Functionality Design), there will be three game instances replayed in parallel in three threads.

**Correctness Requirement**:
- For an arbitrary list of games, running them in parallel should achieve the same result as running them in sequence.

#### Multithreading in ReplaySokobanGame

When instantiating a `ReplaySokobanGame`, it takes an array of `InputEngine` instances and a `RenderingEngine` instance, in addition to `GameState`.
Each `InputEngine` instance corresponds to an action file specified in the arguments of the main method.
Each `InputEngine` instance and `RenderingEngine` instance should be executed in a separate thread.
In the example [above](#Functionality Design), each game instance will have two `InputEngine` threads and one `RenderingEngine` thread.
In the `ReplaySokobanGame` class, we provide two inner wrapper classes `InputEngineRunnable` and `RenderingEngineRunnable` to wrap the `InputEngine` and `RenderingEngine` instances, respectively, so that they can be executed in a thread.
You need to implement them and perform proper thread synchronization to ensure the correctness of the game.

**Replay Mode**:
Multiple `InputEngine` threads perform actions concurrently.
There are two modes of scheduling between `InputEngine` threads: `FREE_RACE` and `ROUND_ROBIN`, which should be supported by `ReplaySokobanGame`.
- `ROUND_ROBIN`: all `InputEngine` threads perform actions in a round-robin fashion (turn by turn).
  In the example [above](#Functionality Design), the actions in two action files are scheduled to be processed in the following order: `PlayerA J`, `PlayerB J`, `PlayerA K`, `PlayerB K`, `PlayerA L`, `PlayerB L`, `PlayerA L`, `PlayerB U`, `PlayerA U`, `PlayerB L`, ...
- `FREE_RACE`: the `InputEngine` threads perform actions concurrently without any scheduling in the order.
  In this mode, the final order of processed actions are arbitrary and may be different across different runs.

**Frame Rate**:
The `RenderingEngine` thread renders the game state in a specified frameRate (frames per second).
In the example [above](#Functionality Design), the frameRate is set to 60, which means the `RenderingEngine` thread will render the game state (invoke `render` method) 60 times per second.

**Requirements**:
- The game must be rendered at least once before the first action is performed (i.e., the initial state of the game must be rendered).
- The game must be rendered at least once after the last action is performed (i.e., the final state of the game must be rendered).
  The trailing `Exit` action does not count.
- Method `run` starts the game by spawn threads for each `InputEngine` and `RenderingEngine` instance.
  When `run` method returns, all spawned threads should already terminate.
- For each action file (and the corresponding `InputEngine`), all actions before (inclusive) the first `Exit` (`E`) should be processed (i.e., fed to the `processAction` method).
- After the first `Exit` (`E`) is processed, all other actions in the action file should be ignored (i.e., not fed to the `processAction` method).
- Actions in the same action file should be processed in the same order as they appear in the action file.
- The game ends when either:
  - The winning condition is satisfied (i.e., all boxes are placed on the destinations).
  - All actions in all action files (before the first `Exit`) have been processed.

**Assumption (Those already implemented, and you should not modify)**:
- The last action in an action file is always `Exit` (`E`).
- Each action file corresponds to one `InputEngine` instance, and they are passed in the same order as an array to `ReplaySokobanGame`.
- The `InputEngine` passed to `ReplaySokobanGame` is an instance of `StreamInputEngine` and `fetchAction` method will return the next action in the action file no matter whether there are `Exit` in the middle.
  If there are no more actions, `Exit` will be returned.

java --enable-preview -jar Sokoban-proguard.jar 3 src/main/resources/map02.map ROUND_ROBIN 100 src/main/resources/actions/actions0.txt src/main/resources/actions/actions1.txt
```
