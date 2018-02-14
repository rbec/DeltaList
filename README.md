# CodecList
A custom data structure in C# implementing `IReadOnlyList<T>` with reduced memory footprint and ***O*** **(log n)** random access.
## Motivation
Sometimes a list of numbers changes only slowly over the indexes of the list, even though the possible range of values remains large. For example a stock price ticker may take any currency value, but two consecutive ticks will typically only differ by a few cents. This presents an opportunity for compression. The aim of this class is to provide a good trade off between space and speed of access for certain use cases.
## Example
The list of numbers below might be stored as a simple array of 4 byte integers per element.

| Index  | Value |
| ------:|------:|
| *0*      | 1000  |
| *1*      | 1003  |
| *2*      | 1005  |
| *3*      | 1002  |
| *4*      | 995   |
| *5*      | 998   |
| *6*      | 1001  |

Since the numbers only change by a small amount each time we could store just the first element (1000) as a 4 byte integer (`int` in C#) and encode each remaining number as an offset from this first element in a single signed byte (`sbyte` in C#).

| Index  | Value | Offset |
| ------:|------:|-------:|
| *0*      | 1000  | 0      |
| *1*      | 1003  | +3      |
| *2*      | 1005  | +5      |
| *3*      | 1002  | +2      |
| *4*      | 995   | -5      |
| *5*      | 998   | -2      |
| *6*      | 1001  | +1      |

A signed byte in [two's complement](https://en.wikipedia.org/wiki/Two%27s_complement) can range from -128 to +127 which means we can only represent numbers that far from the first number. To solve this problem we can introduce the concept of a *key frame* (the name borrowed from the idea of a [key frame](https://en.wikipedia.org/wiki/Key_frame) in video compression) to store a new starting point every time the numbers move outside of the representable range.

| Index  | Value | KeyFrame | Offset |
| ------:|------:|---------:|-------:|
| *0*      | 1000  | 1000     | 0      |
| *1*      | 1003  |          | +3     |
| *2*      | 1005  |          | +5     |
| *3*      | 1002  |          | +2     |
| *4*      | 995   |          | -5     |
| *5*      | 998   |          | -2     |
| *6*      | 1001  |          | +1     |
| *7*      | 1150  | 1150     | 0      |
| *8*      | 1145  |          | -5     |

We can therefore encode this list of numbers with two lists - one for the key frames and one for the offsets. Each key frame consists of a 4 byte integer specifying the index in the original array from which it applies and a 4 byte value that needs to be added to the subsequent offsets.

#### Key Frames
| Index | Key | Value |
| -----:| ---:| -----:|
| *0*     | 0   | 1000  |
| *1*     | 7   | 1150  |

#### Offsets
| Index  | Offset |
| ------:|-------:|
| *0*      | 0      |
| *1*      | +3     |
| *2*      | +5     |
| *3*      | +2     |
| *4*      | -5     |
| *5*      | -2     |
| *6*      | +1     |
| *7*      | 0      |
| *8*      | -5     |

## Implementation
Our implementation need not tie us to specific types of key frame or offsets. We can use the C# type system to generalise the idea to any type `T` in `IReadOnlyList<T>` with any type of offset `TOffset`. Let's define some interfaces to abstract this concept out from the implementation itself.
```C#
public interface IDecoder<T, in TOffset>
{
    T Decode(T keyValue, TOffset offset);
}

public interface ICodec<T, TOffset> : IDecoder<T, TOffset>
{
    TOffset Default { get; }
    bool TryEncode(T value, T keyValue, out TOffset offset);
}
```
The example above using `int` and `sbyte` would be implemented like this:

``` C#
public struct Int32SByteCodec : ICodec<int, sbyte>
{
    public sbyte Default => 0;

    public bool TryEncode(int value, int keyValue, out sbyte offset)
    {
        var difference = value - keyValue;
        offset = (sbyte) difference;
        return difference >= sbyte.MinValue && difference <= sbyte.MaxValue;
    }

    public int Decode(int keyValue, sbyte offset) => keyValue + offset;
}
```

| Index  | Value | KeyFrame | Offset |
| ------:|------:|---------:|-------:|
| *0*      | 1000  | 1000     | 0      |
| *1*      | 1003  |          | +3     |
| *2*      | 1005  |          | +5     |
| *3*      | 1002  |          | +2     |
| *4*      | 995   |          | -5     |
| *5*      | 998   |          | -2     |
| *6*      | 1001  |          | +1     |
| *7*      | 1150  | 1150     | 0      |
| >>>***8***  | **1145**  |          | **-5**     |
| *9*      | 800   | 800      | 0      |
| *10*     | 1000  | 1000     | 0     |
