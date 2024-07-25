# Programming in Matl
## Contents
- [Boring Reference](#Boring-Reference)
    - [File types](#File-types)
    - [Data types](#Data-types)
    - [Scopes](#Scopes)
    - [Keywords](#Keywords)
    - [Expressions](#Expressions)
- [Writing Materials](#Writing-Materials)
- [Writing Libraries](#Writing-Libraries)

## Boring Reference
### File types
In matl there are two types of files:  
```yaml
Materials   :  provides values for properties of some domain and is translated into a fully functional shader code
Libraries   :  can only be used to store functions
```

### Data Types
There are 6 data types in matl:
```yaml
bool      :   logical value, true or false
scalar    :   mathematical scalar, eg. 3.14, 2, 1024.64
vector2   :   mathematical vector with 2 dimensions
vector3   :   mathematical vector with 3 dimensions
vector4   :   mathematical vector with 4 dimensions
texture   :   2 dimensional texture
```
Metal is strongly typed and does not provide any implicit or explicit types conversions.

### Scopes
```python
using domain my_domain            # Material scope

func lerp(y0, y1, alpha)          # Material scope
    let delta = (y1 - y0)         # Function scope
    return y0 + delta * alpha     # Function scope

property color = (1, 1, 1, 1)     # Material scope
property vertex_offset = (0, 0)   # Material scope
```
There 3 scopes in matl:  
```yaml
material    :  global in materials (no indentation)
library     :  global in libraries (no indentation)
function    :  between func keyword and return keyword including return (requires consistent indentation)
```  
Depending on where the code is located other keywords are available.

### Keywords
Matl has only 5 keywords
```yaml
let [variable name] = [expression]                        : creates variable                                     (available in material and functions scopes)
property [property name] = [expression]                   : specify math equation for given property             (available in material scope)
func [function name] ( [arg1 name] , [arg2 name] , ... )  : creates function, opens the function scope           (available in material and library scopes)
return [expression]                                       : returns value from function, end the function scope  (available in function scope)
using [case] [arguments]                                  : do case dependant things                             (available in material and library scopes)
```
using keyword have 3 builtin cases:
```yaml
using domain [name]                         : loads the domain - since this line material can use property keyword (available in materials)
using library [name]                        : links library so it can be used in this material/library             (available in materials and libraries)
using parameter [name] = [default value]    : creates parameter                                                    (available in materials)
```
(the game engine/library You are using can provide extra ones - see its docs)

### Expressions
For mathematical expressions matl uses syntax similar to almost every other programming lanugage:
```js
let a = 2 * lerp(1, 2, 0.5)
```

**Arythmetic operators** 
  
In matl there are 5 arythmetic operators: 
```yaml
- : negation        (allowed:    -scalar, -vectorN)
+ : addition        (allowed:    scalar + scalar, vectorN + vectorN, vectorN + scalar, commutative)
- : subtraction     (allowed:    scalar - scalar, vectorN - vectorN, vectorN - scalar)
* : multiplication  (allowed:    scalar * scalar, vectorN * vectorN, vectorN * scalar, commutative)
/ : division        (allowed:    scalar / scalar, vectorN / vectorN, vectorN / scalar)
```
In all cases the result is the "bigger" type eg. scalar * vector3 returns in vector3.
Note you cannot multiply vectors of diffrent sizes.

**Logical operators**  
  
There are also 10 logical operators:
```yaml
not : negation                (allowed: bool)
and : coniunction             (allowed: bool and bool)
or  : alternative             (allowed: bool or bool)
xor : exclusive alternative   (allowed: bool xor bool)
==  : equality                (allowed: bool == bool, scalar == scalar)
!=  : not equaity             (allowed: bool != bool, scalar != scalar)
<   : less                    (allowed: scalar < scalar)
>   : greater                 (allowed: scalar > scalar)
<=  : less or equal           (allowed: scalar > scalar)
>=  : greater or equal        (allowed: scalar > scalar)
```
All of those return bool as result.

**Accessing vectors components**  
  
Programmer can access given vector components using following syntax:
```python
some_vec_4.g     # returns a scalar, the second vector component
some_vec_4.rgb   # return a vector3, the three first vector components
some_vec_4.brag  # returns a vector4 with the components of the oryginal vector in given order
```

Components names:
```yaml
x or r : first component
y or g : second component
z or b : third component
w or a : fourth component
```

**Constructing vectors**  
  
Vectors can be constructed using parentheses and comas:
```js
let v1 = (1, 2, 3, 4)

let a = 3
let v2 = (a, a, a)
```

**Functions**  
  
Functions can be called with following syntax:
```js
lerp(0.4, 1, 0.5)
library.lerp(0.4, 1, 0.5)
```
The return type will be deduced from the function equations.  

**Symbols**  
  
Domain's symbols can be accessed using following:
```js
domain.[symbol name]
```

**Parameters** 
  
Parameters can be accesed in expressions by their names:
```js
[parameter name]
```

**Conditions**  
  
Variables can be assigned diffrent values varying on conditions, using ``if`` and ``else``.
```js
let a = 3.14
let condition = a > 6.28

let b = if condition : a * 3
        if a == 3.14 : 13
        else           12
```
if using condtions, ``if`` must be the very first token after ``=``. Such code is not allowed:
```js
let b = 2 * (if condition : a * 3  # error
             if a == 3.14 : 13
             else           12)
```
You can have as many if's as you want, but note that there must always be an ``else`` case.  
Also note that all the cases must return the same data type. Such code is not allowed:
```js
let b = if condition : 2          # error
        else           (2, 2)     # error
```

## Writing Materials
There is not many to cover in this topic, when the domains are unknown.  
See if your engine does provide an documentation about materials scripting.
Here is a list of requirements that material must met:
```yaml
1 : Domain must be included (using domain [...])
2 : All properties of the given domain must be specified with property keyword
3 : There must be no errors
```
Once the material met all the rules it can be translated into a proper shader and send to the gpu.

## Writing Libraries
Just dump your functions into a file. See if your engine does have any other requirements