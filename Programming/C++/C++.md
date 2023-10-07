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