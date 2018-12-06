# Pure Coding Style

06, Dec, 2018

Eight months ago i decided that i've had it with C++ complexity and "magic" so i went on a journey towards bringing back the joy of programming into my life. The plan is simple start from c and add complexity as needed.

This series of documents tells the result that i've ended up with.

## Pure World

If you were paying attention in your theory of computation course you would know that a computer is a machine which has memory and able to inspect/change memory based on the values found inside the memory cells. This influenced my style a lot, the style should be as pure as the "theoretical" computer machine.

### Program

The program consists of two main things:

- Data in the form of `struct`, `union`, `enum`, and other various types
- Code in the form of functions that operate on the data

```C++
//data
struct Point
{
	float x, y;
};

//code
Point
point_sub(const Point& a, const Point& b)
{
	return Point {
		a.x - b.x,
		a.y - b.y
	};
}
```

Data is just some bytes in the memory nothing less, nothing more. You can memcpy them around, write them to disk, send them over the network, etc...

Code transforms the data from one form to another and it only operates on the given parameters no global state should be attached.

## Real-life benchmark
I wanted to compare the performance of the pure and simple coding style to the modern C++ coding style and it was shocked at how the performance and my enjoyment of programming could be
This is the numbers from my pet compiler which does lexing, parsing, semantic analysis on 6500LoC file of go-like language

- Modern C++: This is just modern C++ containers, uniqure_ptr, and simple 1-level inheritance nothing crazy like shared_ptr, etc...

	```
	λ .\bin\x64\debug\scratch.exe
	count: 768 decls, time: 487.105569 ms

	λ .\bin\x64\release\scratch.exe
	count: 768 decls, time: 61.23689 ms
	```

- Pure Style

	```
	.\bin\x64\debug\scratch.exe
	count: 768 decls, time: 41.405712 ms

	.\bin\x64\release\scratch.exe
	count: 768 decls, time: 4.644 ms
	```

## Style
What i settled on was C with the addition of four features from C++

- Namespaces mainly for organizing the code
- Function overloading for the rare cases of defining the same function/behavior for different types, but i tend to minimize the usage of overloading because they are confusing overall.
- Simple templates for things that's as simple as the buf example above not meta-programming nonsense
- operator overloading where it's convenient like adding `operator[]` for the buf above

1. Namespaces
	1. Code should always be placed in a namespace
	2. Namespace name should be as short as possible and all lowercase
	3. Nested namespaces should be written

		```C++
		namespace ms::math
		{
			//code
		}
		```
	4. Bring only what you need inside a namespace

		```C++
		namespace ms::math
		{
			using std::cout; //to print maybe
		}
		```

2. Types
	1. Never use class, always use struct
	2. All types should be POD
	3. Add _ at the start of a variable to signal that this specific member is private i.e. `size_t _cap`
	4. All types should follow this style `New_Point`,
		- start with upper case letter `Point` if a type consists of more than one word then the first letter of each word should be upper case and words should be separated with an `_`
	5. Member fields should be snake case `member_field`
	6. Never use member functions

3. Functions
	1. Function name should be snake case `point_sub`
	2. Functions that operates on a type should start with the type name `subject_sub-subject_verb`, this will allow you to write `subject_` and auto-complete your way around the API
		- `file_open`: opens a file.
		- `file_close`: closes a file.
		- `file_cursor_move`: this function operates on a file, specifically the file cursor and it moves it.
	3. Functions that operates on a type should take an instance of that type be reference/value as the first parameter called self
		- `buf_push(Buf<T>& self, const T& value)`: for example operates on a buf so it takes it as the first parameter
	4. Prefer a unique name for each function over function overloading, All of the following functions creates a new string but in a subtle different ways so they have different names
		- `str_new`: creates a new empty string
		- `str_from_c`: creates a new string and copies the c string into it
		- `str_with_allocator`: creates a new empty string with custom allocator

## Example
Let's see a simple type which represents a buffer of data in memory to get the feel for the coding style
```C++
template<typename T>
struct Buf
{
	Allocator allocator;
	T* ptr;
	size_t cap;
	size_t count;
};

template<typename T>
inline static Buf<T>
buf_new()
{
	Buf<T> self{};	//the {} ensures that this variable is zero initialized
	self.allocator = global_allocator;
	return self;
}

template<typename T>
inline static Buf<T>
buf_with_allocator(Allocator allocator)
{
	Buf<T> self{};
	self.allocator = allocator;
	return self;
}

template<typename T>
inline static void
buf_free(Buf<T>& self)
{
	if(self.cap)
		free_from(self.allocator, Block{ self.ptr, self.cap * sizeof(T) });
	self.cap = 0;
	self.count = 0;
	self.ptr = nullptr;
}
```

So the usage would be as simple as this
```C++
void
foo()
{
	auto numbers = buf_new<int>();
	for(int i = 0; i < 100; ++i)
		buf_push(numbers, i);
	buf_free(numbers);
}
```