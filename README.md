<!-- markdownlint-disable line_length MD024 -->

# Operator Overloading Proposal

All examples will use or extend the following class:

```js
class Vec {
  constructor(x, y, z) {
    this.x = x;
    this.y = y;
    this.z = z;
  }

  toString() {
    return `Vector { ${this.x}, ${this.y}, ${this.z} }`;
  }
}
```

A vector is a good object to use for this proposal as it is relatively simple but allows for full demonstrations of all proposed operator overloads (though they may not all make sense).

## Overview

This is an attempt to create a clean standard on which to base operator overloading in order to write cleaner code. Many other languages have operator overloading. This proposal is based primarily on C++ operator overloading, but written with current JS paradigms in mind for maximum compatibility.

The old ways of doing most of these operator overloads result in more verbose code, which, past the function/method declarations, is less readable than using simple operators would be.

```js
class ComparableVec extends Vec {
  equals(vec) {
    return this.x === vec.x && this.y === vec.y && this.z === vec.z;
  }

  strictEquals(vec) {
    return vec instanceof Vec && this.equals(vec);
  }
}

const v1 = new ComparableVec(0, 0, 0);
const v2 = new ComparableVec(0, 0, 0);
const v3 = { x: 0, y: 0, z: 0 };

v1 == v2; // false
v1 == v3; // false

v1 === v2; // false
v1 === v3; // false

v1.equals(v2); // true
v1.equals(v3); // true

v1.strictEquals(v2); // true
v1.strictEquals(v3); // false
```

A simple example, using only `.equals` and `.strictEquals`, is not bad, but if you want to do any sort of complex math on an object, it quickly becomes less readable. Thanks to default operator behavior, the resulting code can also be more bug-ridden or less predictable in its results.

```js
class ArithmeticVec extends Vec {
  add(vec) {
    return new ArithmeticVec(this.x + vec.x, this.y + vec.y, this.z + vec.z);
  }

  subtract(vec) {
    return new ArithmeticVec(this.x - vec.x, this.y - vec.y, this.z - vec.z);
  }

  multiply(vec) {
    return new ArithmeticVec(this.x * vec.x, this.y * vec.y, this.z * vec.z);
  }

  divide(vec) {
    return new ArithmeticVec(this.x / vec.x, this.y / vec.y, this.z / vec.z);
  }

  // etc.
}

const v1 = new ArithmeticVec(1, 4, 7);
const v2 = new ArithmeticVec(5, 3, 4);
const v3 = new ArithmeticVec(2, 5, 9);
const v4 = new ArithmeticVec(8, 4, 0);

v4 + v2;             // "[object Object][object Object]"
v4 * (v1 + v3) / v2; // NaN

v4.add(v2);                         // Vector { 13, 7, 4 }
v4.multiply(v1.add(v3)).divide(v2); // Vector { 6.4, 12, 0 }
```

## Comparison Operators

### == and ===

Probably the two operators that everybody would most like to be able to overload, current behavior acts as follows:

```js
new Vec(0, 0, 0) == new Vec(0, 0, 0);     // false
new Vec(0, 0, 0) == { x: 0, y: 0, z: 0 }; // false

new Vec(0, 0, 0) === new Vec(0, 0, 0);     // false
new Vec(0, 0, 0) === { x: 0, y: 0, z: 0 }; // false
```

#### Proposed Symbol Mapping and behavior

| Symbol                | Operator |
| :-------------------- | :------- |
| `Symbol.equals`       | `==`     |
| `Symbol.strictEquals` | `===`    |

If these are not overridden on a class prototype or object instance, they would default to their current behavior. Even if overridden, `===` should always check to see if both arguments are the same reference before continuing on to the given `[Symbol.strictEquals]` behavior, with `==` doing the same unless being called from an overridden `[Symbol.strictEquals]`.

If only `Symbol.equals` is overridden, then the assumed behavior for `===` would be as follows:

1. If this is a class prototype definition, check the prototype of the RHS is the same as the LHS using `instanceof`. Otherwise, assert the argument being compared to has `typeof obj === 'object'`.
2. Return the result of the `Symbol.equals` check

`Symbol.strictEquals` should not be overridden unless `Symbol.equals` is overridden in the class prototype or object that is attempting to override `Symbol.strictEquals`.

`!=` and `!==` would operate as expected and simply return the negated result of calling, and would not be overloadable.

#### Examples

Only `Symbol.equals` overridden:

```js
class ComparableVec extends Vec {
  [Symbol.equals](vec) {
    return this.x === vec.x && this.y === vec.y && this.z === vec.z;
  }

  // Assumed behavior for [Symbol.strictEquals] since [Symbol.equals] has been overridden:
  // [Symbol.strictEquals](vec) {
  //   return vec instanceof ComparableVec && this == vec;
  // }
}

const comparableVecObj = {
  x: 0,
  y: 0,
  z: 0,

  [Symbol.equals](obj) {
    return this.x === obj.x && this.y === obj.y && this.z === obj.z;
  }

  // Assumed behavior for [Symbol.strictEquals] since [Symbol.equals] has been overridden:
  // [Symbol.strictEquals](obj) {
  //   return typeof obj === 'object' && this == obj;
  // }
}

new ComparableVec(0, 0, 0) == new Vec(0, 0, 0);           // true
new ComparableVec(0, 0, 0) == new ComparableVec(0, 0, 0); // true
new ComparableVec(0, 0, 0) == { x: 0, y: 0, z: 0 };       // true
new ComparableVec(0, 0, 0) == comparableVecObj;           // true

new ComparableVec(0, 0, 0) === new Vec(0, 0, 0);           // false
new ComparableVec(0, 0, 0) === new ComparableVec(0, 0, 0); // true
new ComparableVec(0, 0, 0) === { x: 0, y: 0, z: 0 };       // false
new ComparableVec(0, 0, 0) === comparableVecObj;           // false

comparableVecObj == new Vec(0, 0, 0);           // true
comparableVecObj == new ComparableVec(0, 0, 0); // true
comparableVecObj == { x: 0, y: 0, z: 0 };       // true

comparableVecObj === new Vec(0, 0, 0);           // true
comparableVecObj === new ComparableVec(0, 0, 0); // true
comparableVecObj === { x: 0, y: 0, z: 0 };       // true
```

Both `Symbol.equals` and `Symbol.strictEquals` overridden:

```js
class ComparableVec extends Vec {
  [Symbol.equals](vec) {
    return this.x === vec.x && this.y === vec.y && this.z === vec.z;
  }

  [Symbol.strictEquals](vec) {
    return (vec instanceof Vec || ('x' in vec && 'y' in vec && 'z' in vec)) && this == vec;
  }
}

const comparableVecObj = {
  x: 0,
  y: 0,
  z: 0,

  [Symbol.equals](obj) {
    return this.x === obj.x && this.y === obj.y && this.z === obj.z;
  }

  [Symbol.strictEquals](obj) {
    return typeof obj === 'object' && 'x' in obj && 'y' in obj && 'z' in obj && this == obj;
  }
}

new ComparableVec(0, 0, 0) == new Vec(0, 0, 0);           // true
new ComparableVec(0, 0, 0) == new ComparableVec(0, 0, 0); // true
new ComparableVec(0, 0, 0) == { x: 0, y: 0, z: 0 };       // true
new ComparableVec(0, 0, 0) == comparableVecObj;           // true

new ComparableVec(0, 0, 0) === new Vec(0, 0, 0);           // true
new ComparableVec(0, 0, 0) === new ComparableVec(0, 0, 0); // true
new ComparableVec(0, 0, 0) === { x: 0, y: 0, z: 0 };       // false
new ComparableVec(0, 0, 0) === comparableVecObj;           // false

comparableVecObj == new Vec(0, 0, 0);           // true
comparableVecObj == new ComparableVec(0, 0, 0); // true
comparableVecObj == { x: 0, y: 0, z: 0 };       // true

comparableVecObj === new Vec(0, 0, 0);           // true
comparableVecObj === new ComparableVec(0, 0, 0); // true
comparableVecObj === { x: 0, y: 0, z: 0 };       // true
```

### <, <=, >, and >=

#### Current behavior

Current behavior makes no sense, as it does not throw an error, but instead always returns `false` for `<` and `>` and always returns `true` for `<=` and `>=`.

#### Proposed Symbol Mapping and behavior

| Symbol                    | Operator |
| :------------------------ | :------- |
| `Symbol.greaterThan`      | `>`      |
| `Symbol.greaterThanEqual` | `>=`     |
| `Symbol.lessThan`         | `<`      |
| `Symbol.lessThanEqual`    | `<=`     |

If these are not overridden, then the default behavior will be to throw an error, e.g.:

```js
const v = new Vec(0, 0, 0);

v < v; // Error: No behavior defined for operator '<'
v > v; // Error: No behavior defined for operator '>'
v <= v; // Error: No behavior defined for operator '<='
v >= v; // Error: No behavior defined for operator '>='

const t = {};

t < t; // Error: No behavior defined for operator '<'
t > t; // Error: No behavior defined for operator '>'
t <= t; // Error: No behavior defined for operator '<='
t >= t; // Error: No behavior defined for operator '>='
```

#### Example

```js
class ComparableVec extends Vec {
  [Symbol.greaterThan](vec) {
    return this.x > vec.x && this.y > vec.y && this.z > vec.z;
  }

  [Symbol.lessThan](vec) {
    return this.x < vec.x && this.y < vec.y && this.z < vec.z;
  }

  [Symbol.greaterThanEqual](vec) {
    return this.x >= vec.x && this.y >= vec.y && this.z >= vec.z;
  }

  [Symbol.lessThanEqual](vec) {
    return this.x <= vec.x && this.y <= vec.y && this.z <= vec.z;
  }
}

const v = new ComparableVec(0, 0, 0);

v > new Vec(-1, -1, -1); // true
v > new Vec(0, 0, 0);    // false
v > new Vec(1, 1, 1);    // false

v < new Vec(-1, -1, -1); // false
v < new Vec(0, 0, 0);    // false
v < new Vec(1, 1, 1);    // true

v >= new Vec(-1, -1, -1); // true
v >= new Vec(0, 0, 0);    // true
v >= new Vec(1, 1, 1);    // false

v <= new Vec(-1, -1, -1); // false
v <= new Vec(0, 0, 0);    // true
v <= new Vec(1, 1, 1);    // true
```

## Mathematical Operators

### Binary arithmetic operators (+, -, *, /, %, **)

#### Current behavior

```js
const v = new Vec(0, 0, 0);

// Number on LHS
1 + v;  // "1[object Object]"
1 - v;  // NaN
1 * v;  // NaN
1 / v;  // NaN
1 % v;  // NaN
1 ** v; // NaN

// Object on LHS
// With numbers
v + 1;  // "[object Object]1"
v - 1;  // NaN
v * 1;  // NaN
v / 1;  // NaN
v % 1;  // NaN
v ** 1; // NaN

// With other objects
v + v;  // "[object Object][object Object]"
v - v;  // NaN
v * v;  // NaN
v / v;  // NaN
v % v;  // NaN
v ** v; // NaN
```

#### Proposed Symbol Mapping and behavior

| Symbol            | Operator |
| :---------------- | :------- |
| `Symbol.add`      | `+`      |
| `Symbol.subtract` | `-`      |
| `Symbol.multiply` | `*`      |
| `Symbol.divide`   | `/`      |
| `Symbol.modulus`  | `%`      |
| `Symbol.exponent` | `**`     |

When adding objects to numbers (i.e. number on left hand side of operand), then default behavior will be to return `NaN`, e.g.:

```js
const v = new Vec(0, 0, 0);

1 + v;  // NaN
1 - v;  // NaN
1 * v;  // NaN
1 / v;  // NaN
1 % v;  // NaN
1 ** v; // NaN
```

When adding numbers to objects (i.e. object on left hand side of operand), then default behavior will be to throw an error, e.g.:

```js
const v = new Vec(0, 0, 0);

v + v;  // Error: No behavior defined for operator '+'
v - v;  // Error: No behavior defined for operator '-'
v * v;  // Error: No behavior defined for operator '*'
v / v;  // Error: No behavior defined for operator '/'
v % v;  // Error: No behavior defined for operator '%'
v ** v; // Error: No behavior defined for operator '**'

v + 1;  // Error: No behavior defined for operator '+'
v - 1;  // Error: No behavior defined for operator '-'
v * 1;  // Error: No behavior defined for operator '*'
v / 1;  // Error: No behavior defined for operator '/'
v % 1;  // Error: No behavior defined for operator '%'
v ** 1; // Error: No behavior defined for operator '**'
```

The mathematical assignment operators (`+=`, `-=`, `*=`, `/=`, `%=`, `**=`) would behave as a direct expansion of code, and would not be overloadable, i.e.:

```js
let v = new Vec(0, 0, 0);

v += 3;
```

is equivalent to

```js
let v = new Vec(0, 0, 0);

v = v + 3;
```

#### Example

```js
class ArithmeticVec extends Vec {
  [Symbol.add](other) {
    if (typeof other === 'number') {
      return new ArithmeticVec(this.x + other, this.y + other, this.z + other);
    } else if (other instanceof Vec) {
      return new ArithmeticVec(this.x + other.x, this.y + other.y, this.z + other.z);
    }

    throw new TypeError(`Unable to add disparate types "ArithmeticVec" and "${typeof other}"`);
  }

  [Symbol.subtract](other) {
    if (typeof other === 'number') {
      return new ArithmeticVec(this.x - other, this.y - other, this.z - other);
    } else if (other instanceof Vec) {
      return new ArithmeticVec(this.x - other.x, this.y - other.y, this.z - other.z);
    }

    throw new TypeError(`Unable to subtract disparate types "ArithmeticVec" and "${typeof other}"`);
  }

  [Symbol.multiply](other) {
    if (typeof other === 'number') {
      return new ArithmeticVec(this.x * other, this.y * other, this.z * other);
    } else if (other instanceof Vec) {
      return new ArithmeticVec(this.x * other.x, this.y * other.y, this.z * other.z);
    }

    throw new TypeError(`Unable to multiply disparate types "ArithmeticVec" and "${typeof other}"`);
  }

  [Symbol.divide](other) {
    if (typeof other === 'number') {
      return new ArithmeticVec(this.x / other, this.y / other, this.z / other);
    } else if (other instanceof Vec) {
      return new ArithmeticVec(this.x / other.x, this.y / other.y, this.z / other.z);
    }

    throw new TypeError(`Unable to divide disparate types "ArithmeticVec" and "${typeof other}"`);
  }

  [Symbol.modulus](other) {
    if (typeof other === 'number') {
      return new ArithmeticVec(this.x % other, this.y % other, this.z % other);
    } else if (other instanceof Vec) {
      return new ArithmeticVec(this.x % other.x, this.y % other.y, this.z % other.z);
    }

    throw new TypeError(`Unable to take modulus of disparate types "ArithmeticVec" and "${typeof other}"`);
  }

  [Symbol.exponent](other) {
    if (typeof other === 'number') {
      return new ArithmeticVec(this.x ** other, this.y ** other, this.z ** other);
    } else if (other instanceof Vec) {
      return new ArithmeticVec(this.x ** other.x, this.y ** other.y, this.z ** other.z);
    }

    throw new TypeError(`Unable to take exponent of disparate types "ArithmeticVec" and "${typeof other}"`);
  }

  toString() {
    return `ArithmeticVector { ${this.x}, ${this.y}, ${this.z} }`;
  }
}

const v = new ArithmeticVec(1, 1, 1);

// Number on LHS
1 + v;  // NaN
1 - v;  // NaN
1 * v;  // NaN
1 / v;  // NaN
1 % v;  // NaN
1 ** v; // NaN

// Object on LHS
v + 1;                           // ArithmeticVector { 2, 2, 2 }
v - 1;                           // ArithmeticVector { 0, 0, 0 }
v * 3;                           // ArithmeticVector { 3, 3, 3 }
v / 2;                           // ArithmeticVector { 0.5, 0.5, 0.5 }
new ArithmeticVec(3, 4, 5) % 4;  // ArithmeticVector { 3, 0, 1 }
new ArithmeticVec(4, 5, 6) ** 2; // ArithmeticVector { 16, 25, 36 }

v + new Vec(3, 4, 5);                           // ArithmeticVector { 4, 5, 6 }
v - new Vec(3, 4, 5);                           // ArithmeticVector { -2, -3, -4 }
v * new Vec(3, 4, 5);                           // ArithmeticVector { 3, 4, 5 }
v / new Vec(2, 4, 8);                           // ArithmeticVector { 0.5, 0.25, 0.125 }
new ArithmeticVec(2, 2, 2) % new Vec(1, 2, 3);  // ArithmeticVector { 1, 0, 2 }
new ArithmeticVec(4, 5, 6) ** new Vec(2, 3, 4); // ArithmeticVector { 16, 125, 1296 }

v + {};                           // TypeError: Unable to add disparate types "ArithmeticVec" and "object"
v - {};                           // TypeError: Unable to subtract disparate types "ArithmeticVec" and "object"
v * {};                           // TypeError: Unable to multiply disparate types "ArithmeticVec" and "object"
v / {};                           // TypeError: Unable to divide disparate types "ArithmeticVec" and "object"
new ArithmeticVec(2, 2, 2) % {};  // TypeError: Unable to take modulus of disparate types "ArithmeticVec" and "object"
new ArithmeticVec(4, 5, 6) ** {}; // TypeError: Unable to take exponent of disparate types "ArithmeticVec" and "object"
```

### Unary arithmetic operators (-, ++, --)

#### Current behavior

```js
const v = new Vec(0, 0, 0);

-v;  // NaN

v++; // NaN
v--; // NaN

++v; // NaN
--v; // NaN
```

#### Proposed Symbol Mapping and behavior

| Symbol                 | Operator |
| :--------------------- | :------- |
| `Symbol.unaryNegate`   | `-`      |
| `Symbol.unaryAdd`      | `++`     |
| `Symbol.unarySubtract` | `--`     |

Default behavior should be to throw an error, e.g.:

```js
const v = new Vec(0, 0, 0);

-v;  // Error: No behavior defined for unary operator '-'

v++; // Error: No behavior defined for unary operator '++'
v--; // Error: No behavior defined for unary operator '--'

++v; // Error: No behavior defined for unary operator '++'
--v; // Error: No behavior defined for unary operator '--'
```

When overridden, the timing of `Symbol.unaryAdd` and `Symbol.unarySubtract` will be defined by the position of the operator as in normal unary addition and subtraction, i.e. `++object` and `--object` calling `Symbol.unaryAdd` and `Symbol.unarySubtract` before evaluating current value, and `object++` and `object--` calling `Symbol.unaryAdd` and `Symbol.unarySubtract` after evaluating current value.

#### Example

```js
class ArithmeticVec extends Vec {
  [Symbol.unaryAdd]() {
    this.x++;
    this.y++;
    this.z++;
  }

  [Symbol.unarySubtract]() {
    this.x--;
    this.y--;
    this.z--;
  }

  toString() {
    return `ArithmeticVector { ${this.x}, ${this.y}, ${this.z} }`;
  }
}

const v = new ArithmeticVec(0, 0, 0);

let result;

result = v++; // ArithmeticVector { 0, 0, 0 }
result = v;   // ArithmeticVector { 1, 1, 1 }

result = v--; // ArithmeticVector { 1, 1, 1 }
result = v;   // ArithmeticVector { 0, 0, 0 }

result = ++v; // ArithmeticVector { 1, 1, 1 }
result = v;   // ArithmeticVector { 1, 1, 1 }

result = --v; // ArithmeticVector { 0, 0, 0 }
result = v;   // ArithmeticVector { 0, 0, 0 }
```

## Bitwise Operators

### Bitwise And, Or, Xor, and Complement (&, |, ^, ~)

#### Current Behavior

```js
const v = new Vec(0, 0, 0);

v & v; // 0
v | v; // 0
v ^ v; // 0
~v;    // -1
```

#### Proposed Symbol Mapping and behavior

| Symbol                     | Operator |
| :------------------------- | :------- |
| `Symbol.bitwiseAnd`        | `&`      |
| `Symbol.bitwiseOr`         | `|`      |
| `Symbol.bitwiseXor`        | `^`      |
| `Symbol.bitwiseComplement` | `~`      |

Default behavior should be to throw an error, e.g.:

```js
const v = new Vec(0, 0, 0);

v & v; // Error: No behavior defined for bitwise operator '&'
v | v; // Error: No behavior defined for bitwise operator '|'
v ^ v; // Error: No behavior defined for bitwise operator '^'
~v;    // Error: No behavior defined for bitwise operator '~'
```

The bitwise assignment operators (`&=`, `|=`, `^=`) would behave in the same manner as mathematical assignment operators.

#### Example

```js
class BitwiseVec extends Vec {
  [Symbol.bitwiseAnd](vec) {
    return new BitwiseVec(this.x & vec.x, this.y & vec.y, this.z & vec.z);
  }

  [Symbol.bitwiseOr](vec) {
    return new BitwiseVec(this.x | vec.x, this.y | vec.y, this.z | vec.z);
  }

  [Symbol.bitwiseXor](vec) {
    return new BitwiseVec(this.x ^ vec.x, this.y ^ vec.y, this.z ^ vec.z);
  }

  [Symbol.bitwiseComplement]() {
    return new BitwiseVec(~this.x, ~this.y, ~this.z);
  }

  toString() {
    return `BitwiseVector { ${this.x}, ${this.y}, ${this.z} }`;
  }
}

new Vec(3, 2, 1) & new Vec(1, 2, 3); // BitwiseVector { 1, 2, 1 }
new Vec(3, 2, 1) | new Vec(1, 2, 3); // BitwiseVector { 3, 2, 3 }
new Vec(3, 2, 1) ^ new Vec(1, 2, 3); // BitwiseVector { 2, 0, 2 }
~new Vec(3, 2, 1);                   // BitwiseVector { -4, -3, -2 }
```

### Bitwise Shifting (<<, >>, >>>)

#### Current Behavior

```js
const v = new Vec(0, 0, 0);

v << v; // 0
v >> v; // 0
v >>> v; // 0
```

#### Proposed Symbol Mapping and behavior

| Symbol                      | Operator |
| :-------------------------- | :------- |
| `Symbol.leftShift`          | `<<`     |
| `Symbol.rightShift`         | `>>`     |
| `Symbol.unsignedRightShift` | `>>>`    |

Default behavior should be to throw an error, e.g.:

```js
const v = new Vec(0, 0, 0);

v << v;  // Error: No behavior defined for shifting operator '<<'
v >> v;  // Error: No behavior defined for shifting operator '>>'
v >>> v; // Error: No behavior defined for shifting operator '>>>'
```

The bitwise shift assignment operators (`<<=`, `>>=`, `>>>=`) would behave in the same manner as mathematical and bitwise assignment operators.

#### Example

```js
class ShiftingVec extends Vec {
  [Symbol.leftShift](other) {
    if (typeof other === 'number') {
      return new ShiftingVec(this.x << other, this.y << other, this.z << other);
    } else if (other instanceof Vec) {
      return new ShiftingVec(this.x << other.x, this.y << other.y, this.z << other.z);
    }

    throw new TypeError(`Unable to take left shift of disparate types "ShiftingVec" and "${typeof other}"`);
  }

  [Symbol.rightShift](other) {
    if (typeof other === 'number') {
      return new ShiftingVec(this.x >> other, this.y >> other, this.z >> other);
    } else if (other instanceof Vec) {
      return new ShiftingVec(this.x >> other.x, this.y >> other.y, this.z >> other.z);
    }

    throw new TypeError(`Unable to take right shift of disparate types "ShiftingVec" and "${typeof other}"`);
  }

  [Symbol.unsignedRightShift](other) {
    if (typeof other === 'number') {
      return new ShiftingVec(this.x >>> other, this.y >>> other, this.z >>> other);
    } else if (other instanceof Vec) {
      return new ShiftingVec(this.x >>> other.x, this.y >>> other.y, this.z >>> other.z);
    }

    throw new TypeError(`Unable to take unsigned right shift of disparate types "ShiftingVec" and "${typeof other}"`);
  }
}

const v = new ShiftingVec(1, 1, 1);

v << 1;                            // ShiftingVector { 2, 2, 2 }
new ShiftingVec(64, 32, 16) >> 1;  // ShiftingVector { 32, 16, 8 }
new ShiftingVec(-1, -1, -1) >> 1;  // ShiftingVector { -1, -1, -1 }
new ShiftingVec(-1, -1, -1) >>> 1; // ShiftingVector { 2147483647, 2147483647, 2147483647 }

v << new Vec(1, 2, 3);                            // ShiftingVector { 2, 4, 8 }
new ShiftingVec(64, 32, 16) >> new Vec(1, 2, 3);  // ShiftingVector { 32, 8, 2 }
new ShiftingVec(-1, -1, -1) >> new Vec(1, 2, 3);  // ShiftingVector { -1, -1, -1 }
new ShiftingVec(-1, -1, -1) >>> new Vec(1, 2, 3); // ShiftingVector { 2147483647, 1073741823, 536870911 }

v << {};  // TypeError: Unable to take left shift of disparate types "ShiftingVec" and "object"
v >> {};  // TypeError: Unable to take right shift of  disparate types "ShiftingVec" and "object"
v >>> {}; // TypeError: Unable to take unsigned right shift of disparate types "ShiftingVec" and "object"
```
