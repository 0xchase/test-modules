Below is a **comprehensive Cmajor Language Reference**—condensed into a single guide for Large Language Models that need to generate valid Cmajor (Cmaj) code. It includes all essential rules, syntax, and best practices, along with illustrative examples.

---

# **Cmajor Language Reference**

## **1. Introduction**

**Cmajor (Cmaj)** is a language designed for writing high-performance, safe, and composable DSP (Digital Signal Processing) code. Key goals:

1. **Graph Composability:** Simple procedural DSP code should be easy to combine into graph structures.  
2. **High Performance:** Achieve or exceed C/C++ performance.  
3. **Safety:** Make it impossible to write code that can crash or break real-time safety rules.  
4. **Familiar Syntax:** Borrow from common languages like C/C++, JavaScript, etc., making it quick to learn.

Most DSP code you write in Cmaj will reside in **processors** and **graphs**, with a top-level `main()` or `[ [ main ] ]` annotation typically indicating an entry point.

---

## **2. Lexical Rules**

### 2.1 Whitespace
- Whitespace is ignored except as needed to separate tokens.
- Tabs and spaces are treated equivalently. (Use spaces for indentation!)

### 2.2 Comments
- **C-style multi-line:**
  ```c
  /* Multi-line comments
     use the slash-star syntax. */
  ```
- **C++-style single-line:**
  ```c
  // Single-line comments use double-slashes
  ```

### 2.3 Identifiers
- Allowed characters: `[A-Za-z0-9_]`.
- Must **begin** with a letter (`[A-Za-z]`).
- Case-sensitive.

#### 2.3.1 Suggested Naming Styles
- **Variables**: `lowerCamelCase`
- **Types, processors, structs**: `UpperCamelCase`
- **Namespaces**: `snake_case`

### 2.4 Reserved Keywords
Cannot be used as user-defined names:
```
bool, break, case, catch, class, complex, complex32, complex64, connection, const, 
continue, default, do, double, else, enum, event, external, false, fixed, float, 
float32, float64, for, graph, if, import, input, int, int32, int64, let, loop, 
namespace, node, operator, output, private, processor, public, return, string, 
struct, switch, throw, true, try, using, var, void, while
```

---

## **3. Literals**

### 3.1 Integer Literals
- **32-bit signed integers**: no suffix  
  Examples:
  ```c
  123    // 32-bit decimal
  -456   // 32-bit signed decimal
  0x123  // 32-bit hex
  0b1011 // 32-bit binary
  ```
- **64-bit integers**: Must use `L`, `_L`, `i64`, or `_i64`.  
  Examples:
  ```c
  123456L
  0b1010_i64
  0x1234_L
  ```
- Currently only **signed** integers supported, two’s-complement representation.
- Integer overflow does **not** throw errors; behavior depends on CPU/runtime.

### 3.2 Boolean Literals
- **`true`** and **`false`** (lower-case).

### 3.3 Floating-Point Literals
- Must contain a decimal point.
- **64-bit**: no suffix or `f64` / `_f64`.  
  E.g. `123.0`, `123.0_f64`
- **32-bit**: `f`, `f32`, or `_f32`.  
  E.g. `123.0f`, `123.0_f32`
- **Imaginary literals**: add `i` (e.g. `2.5i`)  
  Combine with integer or float to form **complex** numbers, e.g. `(2.5 + 0.5i)`.

### 3.4 String Literals
- Written in double-quotes.  
  E.g. `"Hello\nWorld\n\uD83D\uDE00"`

### 3.5 Aggregate Literals
- For initializing structs or arrays, use parentheses:
  ```c
  int[5] x = (2, 3, 4, 5, 6);
  MyStruct y = (3, 6.5f, "hello", (3, 4, false));
  var z = bool<4> (true, false, false, true);
  ```

### 3.6 Null Literals
- An empty pair of parentheses `()` can represent a zero or null value of any type:
  ```c
  var x = int[5](); // array of 5 zeros
  x = ();           // sets all 5 elements to 0

  int[] slice = (1, 2, 3);
  slice = ();  // becomes an empty slice of size 0

  float64 i = 123.0;
  i = (); // resets i to 0
  ```

---

## **4. Types**

### 4.1 Primitive Types
1. **int32** (or **int**)
2. **int64**
3. **float32** (or **float**)
4. **float64**
5. **complex32** (or **complex**)
6. **complex64**
7. **bool**

### 4.2 Limited-Range Integer Types
- **`wrap<N>`** or **`clamp<N>`** represent 32-bit integers restricted to `[0 .. N-1]`.
  - `wrap<N>` uses modulo to remain in range.
  - `clamp<N>` saturates at the bounds.
  ```c
  wrap<5> w;
  clamp<5> c;

  loop (7)
  {
      ++w; // cycles: 0, 1, 2, 3, 4, 0, 1
      ++c; // saturates at 4
  }
  ```

### 4.3 Complex Numbers
- **`complex32`** and **`complex64`**, plus **`complex`** as shorthand for `complex32`.
- Imag part with an `i` suffix:
  ```c
  complex32 c1 = 2.0fi + 3.0f;
  let c2 = 3.0f + 2.0fi;
  let c3 = complex (3.0f, 2.0f);

  let ci = c1.imag;   // imaginary part
  let cr = c1.real;   // real part
  ```

### 4.4 Arrays
- Declared with `[size]` after the type:
  ```c
  int[3] x;
  float64[6] y;
  MyStruct[4] z;
  ```
- Must have a compile-time size. Can be **copied** by value.
- If indexing is not proven safe, a **wrap** is inserted at runtime (warning).  
- **Subarray** syntax: `x[start:end]`

#### 4.4.1 Multi-Dimensional Arrays
- Declare with multiple comma-separated sizes or nested brackets:
  ```c
  int[3,4,5] x;
  int[5][4][3] y; // same shape
  ```

### 4.5 Arrays vs Slices
- **Slices** (`int[]`) are references (similar to “fat pointers”).
- Copying a slice does **not** copy elements—just references the same memory.
- The compiler enforces lifetime rules to prevent dangling references.

### 4.6 Vectors
- **Small** fixed-size arrays that only hold primitive numeric types.  
- Parallel arithmetic is supported (possible SIMD).
  ```c
  int<4> x = (1,2,3,4);
  let y = sum(x); // sum = 10
  ```
- Maximum number of elements is typically platform-limited (e.g. ≤128).

### 4.7 Structures
- Declared similarly to C/C++ but without a trailing semicolon.
  ```c
  struct ExampleStruct
  {
      int<5> member1, member2;
      float[] sliceMember;
      float64[4] fixedArrayMember;
      int64 bigIntMember;
  }
  ```
- All members zero-initialized if not explicitly set.

### 4.8 String Type
- **`string`** is read-only; no concatenation or dynamic reallocation.

### 4.9 Enums
- Strong, abstract enumeration type. Cannot be cast to/from int:
  ```c
  enum Animal { cat, dog }
  let c = Animal::cat;
  ```

### 4.10 Constant Types
- **`const`** prefix:
  ```c
  const int x = 10;
  ```

### 4.11 Reference Types
- Append `&` to indicate a reference.  
- References currently only allowed in function parameters.

### 4.12 Type Aliases
- **`using`**:
  ```c
  using MyInt = int64;
  using VectorOfInts = int<4>;
  ```

### 4.13 Type Metafunctions
- Operations on types at compile time, e.g. `.size`, `.elementType`, `.isArray`.
  ```c
  using MyArrayType = int64[10];
  MyArrayType.elementType[5] x; // an int64[5]
  static_assert (MyArrayType.isFixedSizeArray, "must be fixed array");
  ```

---

## **5. Functions**

### 5.1 Basics
- Syntax similar to C/C++/Java:
  ```c
  void doSomething (int param)
  {
      // ...
  }
  ```

### 5.2 Member Functions
- Functions that act on structs or appear as free functions with the struct as first param.  
- `this` is used inside struct-based definitions.

### 5.3 Special Functions

#### 5.3.1 `advance()`
- Moves time forward by one frame; only callable in `main()` or functions exclusively called by `main()`.

#### 5.3.2 `void main()`
- The main real-time loop of a **processor**. Typically an infinite loop with calls to `advance()`.

#### 5.3.3 `void init()`
- Runs once before the real-time loop, for initialization. No streaming or event I/O allowed here.

### 5.4 Recursion
- Not supported (ensures predictable, compile-time stack sizing).

### 5.5 Generic Functions
- Use angle brackets after function name for template parameters:
  ```c
  Type add<Type> (Type a, Type b) { return a + b; }
  ```
- Metafunctions + `static_assert` can check constraints.

---

## **6. Variables and Constants**

### 6.1 Local Variables and Parameters
- **`let`** = compile-time constant or single-assignment.  
- **`var`** = mutable.  
- Same type-inference rules as many modern languages.

### 6.2 Processor State Variables
- Declared inside a processor to persist across frames:
  ```c
  processor NumberGenerator
  {
      output stream float out;
      float value;
      int counter = 10;

      void main()
      {
          ...
      }
  }
  ```

### 6.3 Global Constants
- Allowed in namespaces only:
  ```c
  namespace N
  {
      let x = 1234;
  }
  ```

### 6.4 External Constants
- **`external`** means the value is supplied by the environment:
  ```c
  namespace N
  {
      external int[] someData;
  }
  ```
- Externals are implicitly constant.

---

## **7. Control-Flow**

### 7.1 While, For, Loop
- **`while`** and C-style **`for`** supported:
  ```c
  while (i < 5) { ... }
  for (int i = 0; i < 5; ++i) { ... }
  ```
- **`loop`** statement can be infinite or fixed iteration:
  ```c
  loop (5)
      console <- "Hello"; // repeats 5 times

  loop
  {
      advance(); // infinite loop
  }
  ```

### 7.2 break / continue
- Escape loops. Labeled breaks:
  ```c
  my_outer_loop: loop (100)
  {
      loop (200)
      {
          break;               // leaves inner loop
          break my_outer_loop; // leaves outer loop
          continue my_outer_loop; // restarts outer loop
      }
  }
  ```

### 7.3 if/else
- Classic usage:
  ```c
  if (x == 1) ...
  else if (x == 2) ...
  ```
- **Ternary** operator: `condition ? expr1 : expr2`

### 7.4 if const
- Compile-time branching:
  ```c
  if const (Type.isArray)
      ...
  else if const (Type.isFloat)
      ...
  ```

---

## **8. Arithmetic and Operators**

- **Binary**: `+ - * / % ** & | ^ << >> >>> && || < <= > >= == !=`
- **Unary**: `- ! ~`
- **Increment/Decrement**: `++ --`
- C-style **casts**: `int(x)`, `float<4>(someVar)`

---

## **9. Namespaces, Processors, and Graphs**

A **Cmaj program** typically contains:

1. **Namespace** declarations  
2. **Processor** declarations  
3. **Graph** declarations

### 9.1 Namespaces
- Support nesting. Use `::` to reference.

```c
namespace animals
{
    namespace dogs
    {
        processor Woof { ... }
        string getName() { return "dog"; }
    }

    namespace cats
    {
        processor Miaow { ... }
        string getName() { return "cat"; }
    }
}
```

### 9.2 Processors
- An execution unit with **inputs** and **outputs** (streams, events, or values).
- Contains:
  - Endpoint declarations
  - Functions (including `main()`)
  - State variables
  - Types (structs, using, etc.)

```c
processor MyProcessor
{
    input stream float in;
    output stream float out;

    float stateVar;
    void main()
    {
        loop
        {
            out <- in + stateVar;
            advance();
        }
    }
}
```

#### 9.2.1 Processor Aliases
```c
processor MyAlias = some_namespace::MyProcessor(int<2>, 1234);
```

### 9.3 Graphs
- **Collection** of nodes (processors) + connections between them.
- Graph endpoints declared just like processor endpoints:
  ```c
  graph Gain
  {
      input stream float in;
      output stream float out;
      node gainNode = MyGain (float, 0.5f);

      connection
      {
          in -> gainNode -> out;
      }
  }
  ```
- Graph can also have **event handlers** for **input** events if not routing them directly.

#### 9.3.1 Graph Nodes
```c
node myNode = MyProcessor;
node myArrayNode = MyProcessor[4]; // array of 4 instances
node customNode = SomeProc(float32, 100); // parameterized
```

#### 9.3.2 Connections
Use `->` (right arrow):
```c
connection node1.out -> node2.in;
connection node2.out -> [100] -> node1.in; // add a 100-frame delay
```
- Multiple sources to one stream = **summation** if scalar.

#### 9.3.3 Hoisting Endpoints
Expose child endpoints at the parent graph level:
```c
graph Parent
{
    output child.out1;
    node child = MyProcessor;
}
```
Or use wildcards:
```c
graph Parent
{
    output child.out*;  // hoist all outputs starting with "out"
    node child = MyProcessor;
}
```
Renaming:
```c
graph Parent
{
    output child.outL childOutL, child.outR childOutR;
    node child = MyProcessor;
}
```

#### 9.3.4 Over/Under-sampling
```c
node myOversampledNode = SomeProcessor * 4;  // 4x
node myUndersampledNode = SomeProcessor / 2; // 2x

connection [sinc] myOversampledNode -> out; // sinc interpolation
```
- Allowed only for scalar **stream** types.

### 9.4 Delay and Feedback Loops
- Insert `[N]` to create an N-frame delay.  
- At least 1 sample of delay is required for valid feedback.

### 9.5 Declaring Processor Latency
- `processor.latency = N;` can specify a known processing delay.

```c
processor P
{
    processor.latency = 128;
    // ...
}
```
- The host can automatically adjust graph connections for latency compensation.

### 9.6 Console Output
- Special built-in event stream `console`:
  ```c
  processor P
  {
      void main()
      {
          console <- "Hello World!";
          advance();
      }
  }
  ```

---

## **10. Processor and Namespace Specialization Parameters**

- Processors, graphs, and namespaces can take **template-style** parameters:
  ```c
  processor MyProcessor (using MySampleType, int myConstant)
  {
      output stream MySampleType out;
      ...
  }

  graph G
  {
      node p1 = MyProcessor (float, 100);
  }
  ```
- Parameter can be: **`using`** (a type), **`processor`**, **`namespace`**, or a **constant**.

```c
processor Example (using Type, processor MyProc, int value = 42)
{
    ...
}
```

---

## **11. Annotations**

- Use `[[ ... ]]` with comma-separated key-value pairs.  
- For example:
  ```c
  processor P [[ name: "Hello", animal: "Cat", size: 123 ]]
  {
      input stream float x [[ min: 10.0, max: 100.0 ]];
      ...
  }
  ```
- `[[ main ]]` is often used to mark the top-level processor/graph to run.

---

## **12. Built-in Constants**

Inside a processor:

- `processor.frequency`: **float64** frames/second  
- `processor.period`: **float64** seconds/frame  
- `processor.id`: **int32** unique ID per instance  
- `processor.session`: **int32** unique per run

Globally:

- `nan`: float32 Not A Number  
- `inf`: float32 Infinity  
- `pi`: float64 (π)  
- `twoPi`: float64 (2π)

---

## **13. Intrinsic Functions**

### 13.1 Arithmetic
- `abs(x)`, `sqrt(x)`, `pow(x,y)`, `fmod(x,y)`, `remainder(x,y)`, `floor(x)`, `ceil(x)`, `rint(x)`, `log10(x)`, `log(x)`, `exp(x)`, `roundToInt(x)`

### 13.2 Trigonometric
- `sin(x)`, `cos(x)`, `tan(x)`, `asin(x)`, `acos(x)`, `atan(x)`, `atan2(y,x)`, `sinh(x)`, `cosh(x)`, `tanh(x)`, `asinh(x)`, `acosh(x)`, `atanh(x)`

### 13.3 Comparison & Misc
- `max(a,b)`, `min(a,b)`, `select(condition, x, y)`, `lerp(a,b,t)`

---

## **14. Calling Native Functions from Cmaj**

- Possible via `external` function declarations:
  ```c
  external int32 myNativeFunction (float x, double y, float[] z);
  ```
- Must provide a function pointer in the host environment. Parameter/return types must match.

---

# **Example Code Snippets**

Below are **complete** example snippets demonstrating common usage in Cmaj:

---

### **Example 1: A Simple Filter Graph**
```c
graph Filter [[ main ]]
{
    input filter.frequency [[ mid: 1000 ]];
    input filter.q;

    input stream float<2> in;
    output stream float<2> out;

    node filter = std::filters (float<2>)::tpt::svf::Processor;

    connection
    {
        in -> filter.in;
        filter.out -> out;
    }
}
```
This graph:
- Declares stereo audio input/output.
- Declares two control parameters (`frequency` and `q`) that can be fed into the internal filter processor (not shown in detail here).
- Routes `in` → `filter.in` and `filter.out` → `out`.

---

### **Example 2: A Freeverb Implementation**
```c
graph Freeverb [[ main ]]
{
    input stream float audioIn;
    output stream float<2> audioOut;

    input parameterScaler.roomSize  [[ name: "Room Size", min: 0, max: 100, init: 80, text: "Tiny|Small|Medium|Large|Hall" ]];
    input parameterScaler.damping   [[ name: "Damping Factor", min: 0, max: 100, init: 50, unit: "%", step: 1 ]];
    input parameterScaler.width     [[ name: "Width", min: 0, max: 100, init: 100, unit: "%", step: 1 ]];
    input parameterScaler.wetLevel  [[ name: "Wet Level", min: 0, max: 100, init: 33, unit: "%", step: 1 ]];
    input parameterScaler.dryLevel  [[ name: "Dry Level", min: 0, max: 100, init: 40, unit: "%", step: 1 ]];

    node
    {
        parameterScaler   = ParameterScaler;
        wetGainSmoother   = std::smoothing::SmoothedValueStream (0.02f);
        dryGainSmoother   = std::smoothing::SmoothedValueStream (0.02f);
        widthSmoother     = std::smoothing::SmoothedValueStream (0.02f);
        dampingSmoother   = std::smoothing::SmoothedValueStream (0.02f);
        feedbackSmoother  = std::smoothing::SmoothedValueStream (0.02f);

        reverbL = MonoReverb (0);
        reverbR = MonoReverb (23);

        mixer = WetDryMixer;
    }

    connection
    {
        audioIn -> mixer.audioInDry;
        audioIn -> reverbL.audioIn; reverbL -> mixer.audioInWetL;
        audioIn -> reverbR.audioIn; reverbR -> mixer.audioInWetR;

        parameterScaler.wetGainOut  -> wetGainSmoother  -> mixer.wetGain;
        parameterScaler.dryGainOut  -> dryGainSmoother  -> mixer.dryGain;
        parameterScaler.widthOut    -> widthSmoother    -> mixer.width;
        parameterScaler.dampingOut  -> dampingSmoother  -> reverbL.damping, reverbR.damping;
        parameterScaler.feedbackOut -> feedbackSmoother -> reverbL.feedback, reverbR.feedback;

        mixer -> audioOut;
    }
}

// Scaler for parameter events
processor ParameterScaler
{
    input event float roomSize, damping, wetLevel, dryLevel, width;
    output event float wetGainOut, dryGainOut, widthOut, dampingOut, feedbackOut;

    let wetScaleFactor  = 1.5f;
    let dryScaleFactor  = 2.0f;
    let roomScaleFactor = 0.28f;
    let roomOffset      = 0.7f;
    let dampScaleFactor = 0.4f;

    event roomSize (float newValue)
    {
        feedbackOut <- newValue * roomScaleFactor / 100.0f + roomOffset;
    }

    event damping (float newValue)
    {
        dampingOut <- newValue * dampScaleFactor / 100.0f;
    }

    event dryLevel (float newValue)
    {
        dryGainOut <- newValue * dryScaleFactor / 100.0f;
    }

    event wetLevel (float newValue)
    {
        wetGainOut <- newValue * wetScaleFactor / 100.0f;
    }

    event width (float newValue)
    {
        widthOut <- newValue / 100.0f;
    }
}

// Mixer for wet/dry signals
processor WetDryMixer
{
    output stream float<2> out;
    input stream float audioInDry, audioInWetL, audioInWetR;
    input stream float width, wetGain, dryGain;

    void main()
    {
        loop
        {
            let wetGain1 = wetGain * (1.0f + width);
            let wetGain2 = wetGain * (1.0f - width);

            let wet = float<2> (audioInWetL * wetGain1 + audioInWetR * wetGain2,
                                audioInWetR * wetGain1 + audioInWetL * wetGain2);

            out <- wet + dryGain * audioInDry;
            advance();
        }
    }
}

// Single-channel reverb chain
graph MonoReverb (int offset)
{
    input stream float audioIn, damping, feedback;
    output stream float audioOut;

    node
    {
        comb1 = CombFilter (float, offset + 1116);
        comb2 = CombFilter (float, offset + 1188);
        comb3 = CombFilter (float, offset + 1277);
        comb4 = CombFilter (float, offset + 1356);
        comb5 = CombFilter (float, offset + 1422);
        comb6 = CombFilter (float, offset + 1491);
        comb7 = CombFilter (float, offset + 1557);
        comb8 = CombFilter (float, offset + 1617);

        allpass1 = AllpassFilter (float, offset + 225);
        allpass2 = AllpassFilter (float, offset + 341);
        allpass3 = AllpassFilter (float, offset + 441);
        allpass4 = AllpassFilter (float, offset + 556);
    }

    connection
    {
        audioIn -> comb1.in,
                   comb2.in,
                   comb3.in,
                   comb4.in,
                   comb5.in,
                   comb6.in,
                   comb7.in,
                   comb8.in;

        damping -> comb1.damping,
                   comb2.damping,
                   comb3.damping,
                   comb4.damping,
                   comb5.damping,
                   comb6.damping,
                   comb7.damping,
                   comb8.damping;

        feedback -> comb1.feedback,
                    comb2.feedback,
                    comb3.feedback,
                    comb4.feedback,
                    comb5.feedback,
                    comb6.feedback,
                    comb7.feedback,
                    comb8.feedback;

        comb1,
        comb2,
        comb3,
        comb4,
        comb5,
        comb6,
        comb7,
        comb8  -> allpass1 -> allpass2 -> allpass3 -> allpass4 -> audioOut;
    }
}

// Simple Allpass
processor AllpassFilter (using FrameType, int delayLength)
{
    input  stream FrameType in;
    output stream FrameType out;

    FrameType[delayLength] buffer;
    wrap<delayLength> index;

    void main()
    {
        loop
        {
            let newValue = in;
            let delayedValue = buffer[index];
            buffer[index] = newValue + (delayedValue * FrameType (0.5));
            out <- delayedValue - newValue;
            ++index;
            advance();
        }
    }
}

// Simple Comb Filter
processor CombFilter (using FrameType, int delayLength)
{
    input  stream FrameType in;
    output stream FrameType out;
    input  stream float damping, feedback;

    FrameType[delayLength] buffer;
    wrap<delayLength> index;
    FrameType last;
    let gain = 0.015f;

    void main()
    {
        loop
        {
            let delayedValue = buffer[index];
            out <- delayedValue;
            last = last * damping + delayedValue * (1.0f - damping);
            buffer[index] = last * feedback + gain * in;
            ++index;
            advance();
        }
    }
}
```

This **Freeverb** style reverb graph:
- Showcases multiple comb/allpass filters, scaling of parameter events, and smoothing.

---

### **Example 3: Polyphonic Sine Synth**

```c
/**
  A simple polyphonic synth with sine oscillators,
  using the built-in voice allocator and note converter.
*/

graph SineSynth [[ main ]]
{
    input event std::midi::Message midiIn;
    output stream float out;

    let voiceCount = 8;

    node
    {
        voices = Voice[voiceCount];
        voiceAllocator = std::voices::VoiceAllocator (voiceCount);
    }

    connection
    {
        // Convert raw MIDI to note events and route to the voice allocator
        midiIn -> std::midi::MPEConverter -> voiceAllocator;

        // Distribute note events to each voice
        voiceAllocator.voiceEventOut -> voices.eventIn;

        // Sum the audio from all voices
        voices -> out;
    }
}

graph Voice
{
    input event (std::notes::NoteOn, std::notes::NoteOff) eventIn;
    output stream float out;

    node
    {
        noteToFrequency = NoteToFrequency;
        envelope = std::envelopes::FixedASR (0.01f, 0.1f);
        oscillator = std::oscillators::Sine (float32);
    }

    connection
    {
        eventIn -> noteToFrequency -> oscillator.frequencyIn;
        eventIn -> envelope.eventIn;
        (envelope.gainOut * oscillator.out) -> out;
    }
}

processor NoteToFrequency
{
    input event std::notes::NoteOn eventIn;
    output event float32 frequencyOut;

    event eventIn (std::notes::NoteOn e)
    {
        frequencyOut <- std::notes::noteToFrequency(e.pitch);
    }
}
```

This example:
- Shows how to handle MIDI note events, allocate voices, and generate output with a simple sine oscillator.

---

## **Conclusion**

This condensed **Cmajor Language Reference** should provide an LLM (or any developer) with **all necessary constructs** for writing valid Cmaj code:

1. **Lexical and syntax rules**  
2. **Data types** (primitive, complex, arrays, slices, vectors, structs, enums)  
3. **Functions** (including `main()`, `init()`, and specialized/generic)  
4. **Control flow** (loops, conditionals)  
5. **Processors** (with input/output/event/value endpoints)  
6. **Graphs** (with nodes and connections, including over-sampling, feedback, latency)  
7. **Annotations** for metadata  
8. **Examples** demonstrating real-world usage

With this information, an LLM can generate safe, idiomatic Cmaj code for DSP tasks ranging from simple filters to more advanced polyphonic synthesis.