# CSharp

## Types

```txt
Signed integral types:
sbyte  : -128 to 127
short  : -32768 to 32767
int    : -2147483648 to 2147483647
long   : -9223372036854775808 to 9223372036854775807

Unsigned integral types:
byte   : 0 to 255
ushort : 0 to 65535
uint   : 0 to 4294967295
ulong  : 0 to 18446744073709551615
```

## Methods

### Aggregate (Reduce)

#### Import Aggregate

```c#
using System.Linq;
```

#### Example 1. Summing numbers

```c#
var nums = new[]{1,2,3,4};
var sum = nums.Aggregate( (a,b) => a + b);
Console.WriteLine(sum); // output: 10 (1+2+3+4)
```

#### Example 2. create a csv from an array of strings

```c#
var chars = new []{"a","b","c", "d"};
var csv = chars.Aggregate( (a,b) => a + ',' + b);
Console.WriteLine(csv); // Output a,b,c,d
```

#### Example 3. Multiplying numbers using a seed

```c#
var multipliers = new []{10,20,30,40};
var multiplied = multipliers.Aggregate(5, (a,b) => a * b);
Console.WriteLine(multiplied); //Output 1200000 ((((5*10)*20)*30)*40)
```

#### References

- [linq-aggregate-algorithm-explained](https://stackoverflow.com/questions/7105505/linq-aggregate-algorithm-explained)
