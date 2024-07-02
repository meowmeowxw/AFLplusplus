IJON FRIDA
=======

This is a fork of the <a href="https://github.com/RUB-SysSec/ijon">IJON project</a>, modified to be a binary-only stateful fuzzer. The original IJON project is a compiler-based stateful fuzzer that uses an annotation mechanism to guide fuzzers like AFL++, enabling them to solve complex challenges with minimal annotations.

## Methods

### Primitives

#### `IJON.set(context, val)`

Sets a value in the coverage map of AFL++ by XORing the hashed state value (`val`) with the current instruction pointer. This marks the corresponding entry in the coverage map as true.

- **Parameters:**
  - `context` (object): Execution context.
  - `val` (number): Hashed value.

#### `IJON.inc(context, val)`

Increments a value in the coverage map of AFL++. The hashed state value (`val`) is XORed with the current instruction pointer to locate the map entry, which is then incremented instead of being set.

- **Parameters:**
  - `context` (object): Execution context.
  - `val` (number): Hashed value.

#### `IJON.max(addr, val)`

Implements a maximization strategy by splitting the coverage map of AFL++ and maximizing variables (`val`) independently. This involves marking entries in a queue based on execution results that increase tracked variables, adjusting input mutations accordingly.

- **Parameters:**
  - `addr` (number): Number of variable to track the value of (up to 512).
  - `val` (number): Current value of the tracked variable.

#### `IJON.min(context, val)`

Implements a minimization strategy similar to `IJON.max`.

- **Parameters:**
  - `addr` (number): Number of variable to track the value of (up to 512).
  - `val` (number): Current value of the tracked variable.

### State
Interacting with IJON's internal state variable is useful when you want a state that has already been explored to be considered for new coverage. For instance, within a super mario game if the x and y coordinates are considered a state, the value tracked by this variable could be seen as the different levels within the game.

#### `IJON.xor_state(val)`

Performs a XOR operation for the given value against the current internal state tracked by IJON. 
- **Parameters:**
  - `val` (number): Value to XOR with current state.

#### `IJON.push_state(val)`

Pushes a new state into the current internal state tracked by IJON. The difference between this function and `IJON.xor_state` is that calling this function would gradually add the values to the state instead of using the complete value for the XOR operation. 

- **Parameters:**
  - `val` (number): Value to push onto the state stack.

### Hashing

Using hashes instead of direct values is highly recommanded as it reduces the chance of collisions on the coverage map of AFL++. The hasing algorithm used is Splitmix64.

#### `IJON.hashint(old, val)`

Hashes an integer value.

- **Parameters:**
  - `old` (number): Previous hash value.
  - `val` (number): New integer value to hash.
- **Returns:** 
  - `number`: Updated hash value.

#### `IJON.hashstr(old, val)`

Hashes a string value by hashing each character individually.

- **Parameters:**
  - `old` (number): Previous hash value.
  - `val` (string): New string value to hash.
- **Returns:** 
  - `number`: Updated hash value.

#### `IJON.hashmem(old, val, len)`

Hashes a memory region by hashing each byte individually.

- **Parameters:**
  - `old` (number): Previous hash value.
  - `val` (pointer): Memory pointer.
  - `len` (number): Length of memory to hash.
- **Returns:** 
  - `number`: Updated hash value.

## Examples

### Maze
The following Frida script intercepts execution at the `0x0040127f` of the target Maze game. At this address the coordinates of the player have updated values, and thus, if a new state has been reached the fuzzer would increase the state exploration coverage with the `AFL.IJON.set` function call. Every time a new state is discovered the fuzzer prioritizes the input that lead to that discovery, eventually finding the end of the maze.

```js
const main = DebugSymbol.fromName('main').address;

Interceptor.attach(ptr('0x0040127f'), function(args) {
    var rbp = this.context.rbp;

    // Calculate the address for rbp - 0x224 and rbp - 0x220
    var x_addr = rbp.sub(0x4);
    var y_addr = rbp.sub(0x8);

    var x = Memory.readS32(x_addr);
    var y = Memory.readS32(y_addr);

    var hash = Afl.IJON.hashint(y, x);
    Afl.IJON.set(this.context, hash);
});

Afl.done();
```

### Super Mario
The following Frida script intercepts execution at the `0x004012e0` of the target Super Mario game. At this address the coordinates of the player have updated values, however instead of aiming for new coverage, we use instead `Afl.IJON.max` to find inputs that lead to the highest x value eventually reaching the end of the mario level.

```js
const main = DebugSymbol.fromName('main').address;

Interceptor.attach(ptr('0x004012e0'), function(args) {
    var rbp = this.context.rbp;

    // Calculate the address for rbp - 0x224 and rbp - 0x220
    var x_addr = rbp.sub(0x4);
    var y_addr = rbp.sub(0x8);

    var x = Memory.readS32(x_addr);
    var y = Memory.readS32(y_addr);

    Afl.IJON.max(y, x);
});

Afl.done();
```

## Usage
First, build the AFLplusplus by going into the root directory and running `make` as well as `cd frida_mode && make`. Then, inside this directory, set the `AFL_PATH` variable to the destination of the cloned fork, make sure to not include the trailing slash: `export AFL_PATH=/path/to/AFLplusplus`.

After this is done, starting fuzzing by executing your Frida script with a target binary as such:

```
AFL_FRIDA_JS_SCRIPT=./maze.js $AFL_PATH/afl-fuzz -O -i ./in -o ./out -- ./maze
```

