# MD5 in F#, Functionally

_Code for this post can be found on [GitHub](https://github.com/zakaluka/fsaoc2016/tree/master/5)._

The Advent of Code Day 5 problem presented a very interesting challenge - it required an MD5 hash function to get the final answer.

As my goal with the AOC challenges is to increase my knowledge of F#, I chose to:

- Implement my own version of the MD5 algorithm
- Implement it functionally (no mutation, if possible)

# Out-of-the-box implementation

.NET has an out-of-the-box implementation of MD5, which can be accessed quite easily from F#.  It resides in the `System.Security.Cryptography` namespace.  Here is a function that takes in a `string` and outputs its hash as a `string`.

```fsharp
let md5ootb (msg: string) =
  use md5 = System.Security.Cryptography.MD5.Create()
  msg
  |> System.Text.Encoding.ASCII.GetBytes
  |> md5.ComputeHash
  |> Seq.map (fun c -> c.ToString("X2"))
  |> Seq.reduce ( + )
```

This helps immensely in two ways:

- It makes it very easy to check whether my hash function is working correctly because its output must match the OOTB MD5 algorithm's output.
- It provides a baseline against which to compare performance.

# Planning the implementation

First, let me just say that I have never implemented a "cryptographic" algorithm, such as a hash function, encryption algorithm, etc.  I knew that the implementation would be more difficult than a normal algorithm and so I spent a considerable amount of time reading the following resources.

1. [RFC 1321: The MD5 Message-Digest Algorithm](https://tools.ietf.org/html/rfc1321)
1. [Wikipedia: MD5](https://en.wikipedia.org/wiki/MD5#Algorithm)
1. [Rosetta Code: MD5/Implementation](https://rosettacode.org/wiki/MD5/Implementation)

The first two resources helped me understand the flow of logic.  The last resource is a compilation of MD5 implementations collected in one spot and, luckily, written in multiple languages.  This gave me different perspectives on how the code can be broken up, alternatives for implementation, etc.

I relied especially heavily on the Java, C#, Python, and Haskell implementations because those are the languages I am most familiar with.

For reference, here is the entire listing from Wikipedia.

```vb
'//Note: All variables are unsigned 32 bit and wrap modulo 2^32 when calculating
var int[64] s, K

'//s specifies the per-round shift amounts
s[ 0..15] := { 7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22 }
s[16..31] := { 5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20 }
s[32..47] := { 4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23 }
s[48..63] := { 6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21 }

'//Use binary integer part of the sines of integers (Radians) as constants:
for i from 0 to 63
    K[i] := floor(2^32 × abs(sin(i + 1)))
end for
'//(Or just use the following precomputed table):
'// ... (I did not use a pre-computed table, so I've removed it for brevity sake)

'//Initialize variables:
var int a0 := 0x67452301   //A
var int b0 := 0xefcdab89   //B
var int c0 := 0x98badcfe   //C
var int d0 := 0x10325476   //D

'//Pre-processing: adding a single 1 bit
append "1" bit to message
'// Notice: the input bytes are considered as bits strings,
'//  where the first bit is the most significant bit of the byte.[48]


'//Pre-processing: padding with zeros
append "0" bit until message length in bits ≡ 448 (mod 512)
append original length in bits mod (2 pow 64) to message


'//Process the message in successive 512-bit chunks:
for each 512-bit chunk of message
    break chunk into sixteen 32-bit words M[j], 0 ≤ j ≤ 15
'//Initialize hash value for this chunk:
    var int A := a0
    var int B := b0
    var int C := c0
    var int D := d0
'//Main loop:
    for i from 0 to 63
        if 0 ≤ i ≤ 15 then
            F := (B and C) or ((not B) and D)
            g := i
        else if 16 ≤ i ≤ 31
            F := (D and B) or ((not D) and C)
            g := (5×i + 1) mod 16
        else if 32 ≤ i ≤ 47
            F := B xor C xor D
            g := (3×i + 5) mod 16
        else if 48 ≤ i ≤ 63
            F := C xor (B or (not D))
            g := (7×i) mod 16
'//Be wary of the below definitions of a,b,c,d
        dTemp := D
        D := C
        C := B
        B := B + leftrotate((A + F + K[i] + M[g]), s[i])
        A := dTemp
    end for
'//Add this chunk's hash to result so far:
    a0 := a0 + A
    b0 := b0 + B
    c0 := c0 + C
    d0 := d0 + D
end for

var char digest[16] := a0 append b0 append c0 append d0 '//(Output is in little-endian)

'//leftrotate function definition
leftrotate (x, c)
    return (x << c) binary or (x >> (32-c));
```

## Considerations

There are a few concerns I had about this implementation.

1. The algorithm assumes that all values are stored in the little-endian format.
1. The algorithm requires a bitwise-left-rotate function, which F# does not provide.
1. How to store a 128-bit hash.

Initially, and I'm not sure why I thought this, I was under the assumption that Intel CPUs use the big-endian format. However, they actually use little-endian, which makes that consideration irrelevant.

It took me an _embarrassing_ amount of time to hunt down a bug where I forgot that the F# left-shift operator `<<<` is NOT a left-rotate operator. Once I narrowed down on the bug, it was easy enough to implement a function to perform the rotations.  However, this was a good lesson in never assuming anything when reading code - my eyes were just glazing over the `<<<` operator, even though the problem was staring me in the face.

Finally, I chose to store the values as 4 `uint32` integers.  This is similar to what the RFC authors, Wikipedia authors, and other implementations did on Rosetta Code. While F# does have the `uint64` type, it would have required translating all the 32-bit-based pseudo-code / code (introducing an element of risk) and I'm not sure what benefits, if any, it would have provided.

## Mapping from Wikipedia to F\#

I started my analysis and design phase by taking the pseudo-code from Wikipedia and mapping it to how I wanted to break up my implementation.  I tried to use the same names for variables and, for consistency's sake, I ended up using the Wikipedia naming scheme.

Oddly enough, Wikipedia uses different variable names compared to the RFC pseudo-code and I did not find a good reason for why the original writers did this.

+--+----------------------------------------+----------------------------------------+
|# |Wikipedia                               |F#                                      |
+==+========================================+========================================+
|1 |```vb                                   |```fsharp                               |
|  |'//s specifies the per-round shift      |val s : int list                        |
|  |'//amounts                              |```                                     |
|  |var int[64] s                           |                                        |
|  |```                                     |                                        |
+--+----------------------------------------+----------------------------------------+
|2 |```vb                                   |```fsharp                               |
|  |'//Use binary integer part of the sines |val k : uint32 list                     |
|  |'//of integers (Radians) as constants:  |```                                     |
|  |var int[64] K                           |                                        |
|  |```                                     |                                        |
+--+----------------------------------------+----------------------------------------+
|3 |```vb                                   |```fsharp                               |
|  |'//Initialize variables:                |type MD5 =                              |
|  |var int a0                              |  {a: uint32;                           |
|  |var int b0                              |   b: uint32;                           |
|  |var int c0                              |   c: uint32;                           |
|  |var int d0                              |   d: uint32;}                          |
|  |```                                     |val initialMD5 : MD5                    |
|  |                                        |```                                     |
+--+----------------------------------------+----------------------------------------+
|4 |```vb                                   |```fsharp                               |
|  |'//Pre-processing: adding a single 1 bit|val padMessage : msg:byte [] -> byte [] |
|  |append "1" bit to message               |```                                     |
|  |                                        |                                        |
|  |'//Pre-processing: padding with zeros   |                                        |
|  |append "0" bit until message length in  |                                        |
|  |bits ≡ 448 (mod 512)                    |                                        |
|  |                                        |                                        |
|  |append original length in bits mod (2   |                                        |
|  |pow 64) to message                      |                                        |
|  |```                                     |                                        |
+--+----------------------------------------+----------------------------------------+
|5 |```vb                                   |```fsharp                               |
|  |'//Process the message in successive    |// Array.chunkBySize and Array.fold     |
|  |'//512-bit chunks:                      |val md5sum : msg:string -> string       |
|  |for each 512-bit chunk of message       |```                                     |
|  |```                                     |                                        |
+--+----------------------------------------+----------------------------------------+
|6 |```vb                                   |```fsharp                               |
|  |'//Initialize hash value for this chunk:|// List.fold                            |
|  |var int A, B, C, D                      |val md5plus : m:MD5 -> bs:byte [] -> MD5|
|  |                                        |```                                     |
|  |'//Main loop:                           |                                        |
|  |for i from 0 to 63                      |                                        |
|  |  '// NOT THE CONTENTS OF THE LOOP      |                                        |
|  |end for                                 |                                        |
|  |                                        |                                        |
|  |'//Add this chunk's hash to result so   |                                        |
|  |'//far:                                 |                                        |
|  |a0, b0, c0, d0                          |                                        |
|  |```                                     |                                        |
+--+----------------------------------------+----------------------------------------+
|7 |```vb                                   |```fsharp                               |
|  |'//Main loop: (contents)                |val md5round : msg:uint32 [] -> MD5 ->  |
|  |if ...                                  |i:int -> MD5                            |
|  |else if ...                             |                                        |
|  |else if ...                             |// Includes:                            |
|  |else if ...                             |val fghi : (uint32 -> uint32 -> uint32  |
|  |                                        |-> uint32) list                         |
|  |dTemp, D, C, B, A                       |                                        |
|  |```                                     |val gIdxs : int list                    |
|  |                                        |```                                     |
+--+----------------------------------------+----------------------------------------+

Table: _Mapping from Wikipedia's pseudo-code to my F# implementation (function and value signatures)._

# Implementation

## Step 1: Define the shift in each of the 64 rounds

On the Wikipedia page, the per-round shift amounts are hard-coded as a 64-value `int` array.

```vb
s[ 0..15] := { 7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22,  7, 12, 17, 22 }
s[16..31] := { 5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20,  5,  9, 14, 20 }
s[32..47] := { 4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23,  4, 11, 16, 23 }
s[48..63] := { 6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21,  6, 10, 15, 21 }
```

In F#, I used some convenience functions in the `List` module to achieve the same result.

```fsharp
let s =
  [[7; 12; 17; 22]; [5; 9; 14; 20]; [4; 11; 16; 23]; [6; 10; 15; 21]]
  |> List.collect (List.replicate 4)
  |> List.concat

val s : int list
```

## Step 2: Calculate the binary integer portion of integer sines

Wikipedia provides two methods of calculating the constants that are based on the `sine` function.  First, here is the description from the RFC.

> ... constructed from the sine function. Let T[i] denote the i-th element of the table, which is equal to the integer part of 4294967296 times abs(sin(i)), where i is in radians.

Wikipedia's two methods of defining `K[i]` (Wikipedia's name for `T[i]`) are as follows:

```vb
for i from 0 to 63
    K[i] := floor(2^32 × abs(sin(i + 1)))
end for
'//(Or just use the following precomputed table):
K[ 0.. 3] := { 0xd76aa478, 0xe8c7b756, 0x242070db, 0xc1bdceee }
K[ 4.. 7] := { 0xf57c0faf, 0x4787c62a, 0xa8304613, 0xfd469501 }
K[ 8..11] := { 0x698098d8, 0x8b44f7af, 0xffff5bb1, 0x895cd7be }
K[12..15] := { 0x6b901122, 0xfd987193, 0xa679438e, 0x49b40821 }
K[16..19] := { 0xf61e2562, 0xc040b340, 0x265e5a51, 0xe9b6c7aa }
K[20..23] := { 0xd62f105d, 0x02441453, 0xd8a1e681, 0xe7d3fbc8 }
K[24..27] := { 0x21e1cde6, 0xc33707d6, 0xf4d50d87, 0x455a14ed }
K[28..31] := { 0xa9e3e905, 0xfcefa3f8, 0x676f02d9, 0x8d2a4c8a }
K[32..35] := { 0xfffa3942, 0x8771f681, 0x6d9d6122, 0xfde5380c }
K[36..39] := { 0xa4beea44, 0x4bdecfa9, 0xf6bb4b60, 0xbebfbc70 }
K[40..43] := { 0x289b7ec6, 0xeaa127fa, 0xd4ef3085, 0x04881d05 }
K[44..47] := { 0xd9d4d039, 0xe6db99e5, 0x1fa27cf8, 0xc4ac5665 }
K[48..51] := { 0xf4292244, 0x432aff97, 0xab9423a7, 0xfc93a039 }
K[52..55] := { 0x655b59c3, 0x8f0ccc92, 0xffeff47d, 0x85845dd1 }
K[56..59] := { 0x6fa87e4f, 0xfe2ce6e0, 0xa3014314, 0x4e0811a1 }
K[60..63] := { 0xf7537e82, 0xbd3af235, 0x2ad7d2bb, 0xeb86d391 }
```

For my solution, I chose to go the function route instead of hard-coding the table into the source code.  This was primarily for brevity's sake.  A side benefit was that it allowed me to put in a long and oddly satisfying function chain.

```fsharp
let k =
  [1. .. 64.] |> List.map (sin >> abs >> (( * ) (2.**32.)) >> floor >> uint32)

val k : uint32 list
```

## Step 3: Define the types

I defined one new type for this problem, `MD5`.  This record holds an MD5 hash as 4 `uint32` values.

```fsharp
type MD5 =
  {
    a : uint32
    b : uint32
    c : uint32
    d : uint32
  }

let initialMD5 =
  {
    a = 0x67452301u
    b = 0xefcdab89u
    c = 0x98badcfeu
    d = 0x10325476u
  }
```

## Step 4: Padding the message

The MD5 algorithm doesn't require a message of a certain length, but instead adds padding so that the final message length is a multiple of 512 bits (or 64 bytes).

There are 3 steps involved in padding:

```vb
'//Pre-processing: adding a single 1 bit
append "1" bit to message

'//Pre-processing: padding with zeros
append "0" bit until message length in bits ≡ 448 (mod 512)
append original length in bits mod (2 pow 64) to message
```

This part actually gave me some trouble because I thought that I had to add one bit (containing `1`) to my message in the MSB (most significant bit) slot, and then start appending `0`s or the message length immediately after that.  That turns out not to be the case, and reading the RFC and other implementations on Rosetta Code provided a good solution to the problem.

Essentially, following the message, we need to add a `0x80` 16-bit word followed by the length (restricted to 8 bytes).  The intervening space, if any, is filled with `0` bytes.

```fsharp
let padMessage (msg : byte []) =
  let msgLen = Array.length msg
  let msgLenInBits = (uint64 msgLen) * 8UL

  let lastSegmentSize =
    let m = msgLen % 64
    if m = 0 then 64
    else m

  let padLen =
    64 - lastSegmentSize + (if lastSegmentSize >= 56 then 64
                            else 0)

  [|
    yield 128uy
    for i in 2..padLen - 8 do
      yield 0uy
    for i in 0..7 do
      yield ((msgLenInBits >>> (8 * i)) |> byte)
  |]
  |> Array.append msg

val padMessage : msg:byte [] -> byte []
```

This is the one part of the implementation that had the highest potential to be "impure".  Specifically, the part that constructs the padding.  However, F# array comprehensions removed the need to mutate an `array` to construct the padding before it is appended to the `msg`.

## Step 7: The core of the main loop (Yes, I know this is out of order)

The Wikipedia pseudo-code is written in an imperative manner where the main loops are encountered before we get to the meat of the hash construction algorithm.  However, in order to explain my construction, I will start by looking at the implementation for a single iteration of the "Main loop", and then build my way out from there.

The Wikipedia pseudo-code for this step is as follows.

```vb
        if 0 ≤ i ≤ 15 then
            F := (B and C) or ((not B) and D)
            g := i
        else if 16 ≤ i ≤ 31
            F := (D and B) or ((not D) and C)
            g := (5×i + 1) mod 16
        else if 32 ≤ i ≤ 47
            F := B xor C xor D
            g := (3×i + 5) mod 16
        else if 48 ≤ i ≤ 63
            F := C xor (B or (not D))
            g := (7×i) mod 16
'//Be wary of the below definitions of a,b,c,d
        dTemp := D
        D := C
        C := B
        B := B + leftrotate((A + F + K[i] + M[g]), s[i])
        A := dTemp
```

In F#, I have three separate constructs that we need to look at.  First, I followed the technique used by a few other implementations and collected the functions represented by `F` in a function `list`.  This allows index-based access to the correct function.

```fsharp
let fxyz x y z : uint32 = (x &&& y) ||| (~~~x &&& z)
let gxyz x y z : uint32 = (z &&& x) ||| (~~~z &&& y)
let hxyz x y z : uint32 = x ^^^ y ^^^ z
let ixyz x y z : uint32 = y ^^^ (x ||| ~~~z)

let fghi =
  [fxyz; gxyz; hxyz; ixyz]
  |> List.collect (List.replicate 16)

val fxyz : x:uint32 -> y:uint32 -> z:uint32 -> uint32
val gxyz : x:uint32 -> y:uint32 -> z:uint32 -> uint32
val hxyz : x:uint32 -> y:uint32 -> z:uint32 -> uint32
val ixyz : x:uint32 -> y:uint32 -> z:uint32 -> uint32
val fghi : (uint32 -> uint32 -> uint32 -> uint32) list
```

Second, I turned `g` into a set of functions and, as above, collected the pre-computed indices in a `list`.

```fsharp
let g1Idx = id
let g2Idx i = (5*i + 1) % 16
let g3Idx i = (3*i + 5) % 16
let g4Idx i = (7*i) % 16

let gIdxs =
  [g1Idx; g2Idx; g3Idx; g4Idx]
  |> List.collect (List.replicate 16)
  |> List.map2 (fun idx func -> func idx) [0..63]

val g1Idx : ('a -> 'a)
val g2Idx : i:int -> int
val g3Idx : i:int -> int
val g4Idx : i:int -> int
val gIdxs : int list
```

Finally, I implemented a single round of the MD5 computation.

```fsharp
let md5round (msg:uint32[]) {MD5.a=a; MD5.b=b; MD5.c=c; MD5.d=d} i =
  let rotateL32 r x = (x<<<r) ||| (x>>>(32-r))
  let f = fghi.[i] b c d
  let a' = b + (a + f + k.[i] + msg.[gIdxs.[i]] |> rotateL32 s.[i])
  {a=d; b=a'; c=b; d=c}

val md5round : msg:uint32 [] -> MD5 -> i:int -> MD5
```

The `md5round` function takes in the computed-so-far MD5 hash and an index, calculates the updated hash, and passes it back to the caller.

## Step 6: Hash a single 512-bit chunk

This step actually implements the "Main loop", using the logic we coded for a single iteration of the loop.

Here is Wikipedia's pseudo-code, although I have removed the text (replaced by a comment) that is part of Step 7.

```vb
for each 512-bit chunk of message
    break chunk into sixteen 32-bit words M[j], 0 ≤ j ≤ 15
//Initialize hash value for this chunk:
    var int A := a0
    var int B := b0
    var int C := c0
    var int D := d0
//Main loop:
    for i from 0 to 63
        '// SEE STEP 7
    end for
'//Add this chunk's hash to result so far:
    a0 := a0 + A
    b0 := b0 + B
    c0 := c0 + C
    d0 := d0 + D
end for
```

Note that although Wikipedia uses the labels `a0`, `b0`, `c0`, and `d0`, these are mutable values that permanently change with each 512-bit chunk that is processed.  Thus, each new 512-bit chunk starts with these 4 values that have been changed by all the previous chunks that have been processed.

The flow in this part of the code, without the details from Step 7, can be broken down into 3 parts:

1. Start with an initial value.
1. Run an algorithm a limited number of times, wherein each execution of the algorithm uses the results from the previous run as the input.
1. Take the final value and continue with the overall flow.

This is a perfect scenario for a `fold` operation.  Here is the F# code from my implementation.

```fsharp
let md5plus m (bs:byte[]) =
  let msg =
    bs
    |> Array.chunkBySize 4
    |> Array.take 16
    |> Array.map (fun elt -> System.BitConverter.ToUInt32(elt, 0))
  let m' = List.fold (md5round msg) m [0..63]
  {a=m.a+m'.a; b=m.b+m'.b; c=m.c+m'.c; d=m.d+m'.d}

val md5plus : m:MD5 -> bs:byte [] -> MD5
```

`md5plus` expects the caller to pass in the initial value for the fold and the message that must be processed.

`md5round` operates on a message (that will not change for a single 512-bit chunk), an MD5 value, and an index.  Those are the values that `md5plus` is using for the `fold`.

At the end, `md5plus` passes back an updated MD5 hash which incorporates the changes derived from processing the given 512-bit chunk.

## Step 5: Process the entire message and derive a single MD5 hash

The last step is to bring together all the individual pieces into a single algorithm that can:

- Take in a message of any length
- Pad the message correctly
- Calculate the MD5 hashes for each 512-bit chunk
- Produce a single MD5 hash.

Wikipedia's pseudo-code for this step is quite simple.

```vb
var char digest[16] := a0 append b0 append c0 append d0
'//(Output is in little-endian)
```

However, since I built the solution as a set of functions, this final function must do a little more work.

```fsharp
let md5sum (msg: string) =
  System.Text.Encoding.ASCII.GetBytes msg
  |> padMessage
  |> Array.chunkBySize 64
  |> Array.fold md5plus initialMD5
  |> (fun {MD5.a=a; MD5.b=b; MD5.c=c; MD5.d=d} ->
    System.BitConverter.GetBytes a
    |> (fun x -> System.BitConverter.GetBytes b |> Array.append x)
    |> (fun x -> System.BitConverter.GetBytes c |> Array.append x)
    |> (fun x -> System.BitConverter.GetBytes d |> Array.append x))
  |> Array.map (sprintf "%02X")
  |> Array.reduce ( + )

val md5sum : msg:string -> string
```

The main part of the algorithm is just the first 4 lines:

```fsharp
let md5sum (msg: string) =
  System.Text.Encoding.ASCII.GetBytes msg
  |> padMessage
  |> Array.chunkBySize 64
  |> Array.fold md5plus initialMD5
```

The rest of the function is devoted to converting the MD5 hash into a user-friendly representation.

# Testing

Based on the testing I've done so far, this algorithm works correctly and produces exactly the same results as the OOTB MD5 algorithm.

I first started by implementing the "standard" tests (originally specified in the RFC) and comparing the results with the OOTB algorithm.

Then, I added an [FsCheck](https://github.com/fscheck/FsCheck) test that compared my algorithm against the OOTB one for 10,000 tests - all passed without any problems.  Now that I've finally learned how to properly use FsCheck at a basic level, I'm starting to find it difficult to completely trust my results unless all the FsCheck tests pass.  I don't think that this is necessarily a bad thing, because it tends to make explicit assumptions that I made in my code (e.g. for my MD5 algorithm, I assume that the original message is an actual string and not just a `NULL`).

# Performance

_Raw data for performance measurements on [GitHub](https://github.com/zakaluka/blog/blob/master/aoc/d5runtimes.xlsx)._

I measured performance by using the AOC Day 5, Part 1 problem, which computes thousands of MD5 hashes.  Here are the results, derived using F# interactive's `#time` directive.  I ran each test 3 times and averaged the results.

| Algorithm  | Real   | CPU    | Gen0     | Gen1  | Gen2 |
|:-----------|-------:|-------:|---------:|------:|-----:|
| OOTB       | 53.37  | 53.37  | 6780.33  | 6.00  | 0.67 |
| Functional | 172.79 | 172.76 | 34754.00 | 17.67 | 1.67 |

Table: _Average run-time and garbage collection performance of the algorithms, in seconds_

Here is the same chart, but with percentages.

| Algorithm  | Real | CPU  | Gen0 | Gen1 | Gen2 |
|:-----------|-----:|-----:|-----:|-----:|-----:|
| OOTB       | 100% | 100% | 100% | 100% | 100% |
| Functional | 324% | 324% | 513% | 294% | 250% |

Table: _Average run-time and garbage collection performance of the algorithms as a percentage, with the OOTB algorithm as the baseline._

Obviously, my functional version of this algorithm is a LOT worse than the OOTB implementation.

# Lessons Learned

1. Be extremely careful of assumptions when reading code.  Each character and symbol should be analyzed to ensure that it is appropriate for the task at hand.
1. Define the `tee` function at the beginning of each project because it WILL be useful at some point.
1. It's been a goal of mine to implement a "cryptographic" algorithm for a few years now, but I never found the time before.  I have to say that it was a lot of fun and I can't wait to do it again!
    - Not strictly a lesson, unless that lesson is "have fun with hobby projects".

See you next time!