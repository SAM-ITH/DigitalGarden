Basic concepts of C++ programming 

#### Intro to C++ 

- C++ is a complied language its means it will convert into machine code before its runs 
- memory management should be done manually (dynamic memory must be allocated and destroyed manually)

#### Printing a value in the screen 
- this can be done in two separate ways 
	- using `printf` or `cout`

```cpp

printf("hello samith");

std:: cout << "hello samith" << std::endl;

```

#### Inputs and outputs in C++ 

- inputs can get through via terminal using following command. but you need to import `#include <iostream>` for that 

```cpp
int a; 
std:: cin >> a;
```
- output can get through using follows 

```cpp
std:: cout << a << std::endl
```

#### Basic data types 

`char` - characters 
`long` - used to record 64 bit integer 
`float` - record 32 bit of real value 
`double` - record 64 bit of real value 

- `fixed`  used with  float  to print specify amount of digits that tell by the `setprecision()` this is explained clearly in the below example 
```cpp
double j;
std:: cout << std::fixed << std::setprecision(3) << j << '\n';
```

#### Conditional statements 

##### for loop 

```cpp
for(<expression1> ; <expression2> ; <expression3>) {
<statement>
}
```
- expression 1 - initialising the flag variables that generally used for controlling and terminating the loop 
- expression 2 - used to checked the terminating condition 
- expression 3 - generally used to update the flag variables 

- example : 
```cpp
for(int i = 0; i < 10; i++) {
    ...
}
```

#### Functions 

- syntax of a function 

```cpp
return_type function_name (arg1_type arg_1, arg2_type arg_2){

return value;
}
```

##### Pointers 

