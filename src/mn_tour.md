# MN Tour

01, Jan, 2019

MN is a minimal support library that i use and helps me in my "Pure Coding" style.

I think the way C was intended is to build your own tools and abstractions on top of the language the C standard library provides both an example and a bare minimum library to use in your application, You need to build your own tools and abstractions that suites you and the programs you intend to build, so feel free to take it change it to suite your needs, also if you wanted to add new features to it send them to me and I'll see what i can do.

You can communicate with me through e-mail moustapha.saad.abdelhamed@gmail.com

The library is licensed under BSD-3 and you can find it here [MN](https://gitlab.com/MoustaphaSaad/mn)

Let's get a quick tour of it's content. Remember this is a quick tour browse the header files for full documentation and to view the full interface nearly each function is documented in-line in the code.

## Memory

Memory is the most used resource in any program that's why i pay extra attention to it

### Representation

Memory is represented in its simplest form is just a `void*` to the memory and a `size_t` of that region of memory

```C++
struct Block
{
	void*  ptr;
	size_t size;

	operator bool() const
	{
		return ptr != nullptr && size != 0;
	}
};
```

Allocation and freeing memory in its simplest form is done using

```C++
inline static Block
alloc(size_t size, uint8_t alignment);

inline static void
free(Block block);
```

All interaction in memory is done using `Block`s but what if you need to allocate a single instance of any type do you have to go through the above interface?

```C++
template<typename T>
inline static T*
alloc();

template<typename T>
inline static T*
alloc_zerod();

template<typename T>
inline static void
free(const T* ptr);
```

The above interface is used in allocation and freeing in a single instance cases

### Allocators

All memory traffic goes through an allocator whether it's a custom allocator or a global one

```C++
//for example this is the clib allocator (malloc/free) this is the default allocator
inline Allocator clib_allocator = Allocator(-1);
```

Remember the above `alloc` and `free` functions those are actually based on the functions below

```C++
Block
alloc_from(Allocator allocator, size_t size, uint8_t alignment);

void
free_from(Allocator allocator, Block block);
```

They just use the `clib_allocator`

#### Allocator Types

There are currently three kinds of allocator

- Stack Allocators: a stack allocator that pumps a pointer into an internal stack and returns a nullptr when it runs out of memory (you can only free the last-element/top-of-stack)
- Arena Allocators: an arena is a list of stacks/buckets and it allocates memory in each stack/bucket until it's full then it adds another stack/bucket to the list (doesn't free anything inside on its own, you can only free the entire arena at once)
- Tmp Allocators: sometimes you don't know a clear owner of the memory or a clear owner doesn't exist at all and for this reason you can use the tmp allocator which is just a `thread_local` arena that you can only `allocator_tmp_free` on a regular basis in case you are the application, it's not recommended to call `allocator_tmp_free` if you are implementing a library because the application is the only place where you are sure that when you call `allocator_tmp_free` there will be no dangling reference pointing to the tmp memory.

### Example

```C++
//Allocators work in a stack manner where you can push and pop
//allocators to the stack and everything by default uses the top of the allocator stack
//here we add the leak detector allocator to be used by the code below
allocator_push(leak_detector());

//you can allocate any type using this way but it allocate a single instance
Point* p = alloc<Point>();
p->x = 12;
p->y = 13;

//don't forget to free it
free(p);


//also you can allocate a block of memory
//this allocates a 1 KB of memory
Block m = alloc(1024, alignof(char));

//you can check the block using a simple if statement
if(m)
{
	//block is not empty
}

//don't forget to free it
free(m);

//here we pop the leak detector from the allocator so we switch back to the default
//allocator which is the clib allocator
//go ahead and remove one of the free calls above to see the leak detection report:
// Leak size: 1024, call stack:
// [12]: mn::callstack_dump
// [11]: <lambda_df79aca8c65b7a47c69b16965c6f8a4c>::operator()
// [10]: <lambda_df79aca8c65b7a47c69b16965c6f8a4c>::<lambda_invoker_cdecl>
// [9]: mn::alloc_from
// [8]: mn::alloc
// [7]: mn_tour_stuff
// [6]: main
// [5]: invoke_main
// [4]: __scrt_common_main_seh
// [3]: __scrt_common_main
// [2]: mainCRTStartup
// [1]: BaseThreadInitThunk
// [0]: RtlUserThreadStart
// 
// Leaks count: 1, Leaks size(bytes): 1024
allocator_pop();

//we can create an arena so that we can allocate nodes freely and don't
//worry about managing the lifetime of each node individually
//at the end all we have to do is just call `allocator_free` to free
//the arena itself and all the nodes that's allocated from it
Allocator arena = allocator_arena_new();

Node* node = alloc_from<Node>(arena);
node->data = 234;
node->next = nullptr;

//don't forget to free the arena
allocator_free(arena);
```

## Buf

Buf is a dynamic c array implementation

```C++
Buf<int> numbers = buf_new<int>();

buf_reserve(numbers, 10);

for(int i = 0; i < 10; ++i)
	buf_push(numbers, i);

//you can iterate over it using the simple form
for(int i = 0; i < numbers.count; ++i)
	printfmt("{}\n", numbers[i]);

//or you can use a range for loop
for(int n: numbers)
	printfmt("{}\n", numbers[i]);

//don't forget to free it
buf_free(numbers);

//you can also create a buf with a custom allocator
Buf<int> numbers = buf_with_allocator<int>(custom_allocator);
```

## Str

Str is a `Buf<char>` specialization which maintains a null termination byte at the end also it's utf-8 string

```C++
Str name = str_from_c("Mostafa");
printfmt("{}\n", name);

str_push(name, u8" مصطفى");
printfmt("{}\n", name);

//you can iterator over the individual chars like the buf
for(size_t i = 0; i < name.count; ++i)
	printfmt("{}\n", name[i]);

//or you can iterate over the runes
for(const char* it = name.ptr;
	*it;
	it = rune_next(it))
{
	int32_t c = rune_read(it);
	printfmt("0x{:X}\n", c);
}

//dont' forget to free the name
str_free(name);
```

## Map

Map is an open-addressing hash map implementation

```C++
Map<Str, int> map = map_new<Str, int>();

map_reserve(map, 10);

for(int i = 0; i < 10; ++i)
{
	Str name = str_from_c("Mostafa ");
	//here we use the printf style of formatting since str_pushf uses snprintf under the hood
	str_pushf(name, "%d", i);
	map_insert(map, name, i);
}

//no we can iterate over the map
for(auto it = map_begin(map);
	it != map_end(map);
	it = map_next(map, it))
{
	printfmt("{}: {}\n", it->key, it->value);
}

//we can remove elements from the map
map_remove(map, str_lit("Mostafa 0"));

//also we can do lookups
//str_lit is used to wrap the c string only (no allocation happens)
if(auto it = map_lookup(map, str_lit("Mostafa 0")))
	printfmt("{}: {}\n", it->key, it->value);

//no since we have a composite structure Map of Str if we call `map_free` we would be only 
//freeing the map and leaking all the strings so instead we will call destruct which will
//free the strings also
destruct(map);
```

## File

Let's load the binary content of a file

```C++
//here we open the file with read permission and we choose to open only mode
File f = file_open("D:/content.bin", IO_MODE::READ, OPEN_MODE::OPEN_ONLY);

//we check that the file is opened
if(file_valid(f) == false)
	panic("cannot read file");

//we allocate a block of exactly the same size as the file
Block content = alloc(file_size(f), alignof(char));

//then we read the file into the block
size_t read_size = file_read(f, content);
assert(read_size == content.size);

//don't forget to close the file
file_close(f);
```

## File System Manipulation

Let's print all the file names inside a specific folder

```C++
Buf<Path_Entry> content = path_entries("D:/");

for(Path_Entry& entry: content)
	if(entry.kind == Path_Entry::KIND_FILE)
	{
		//DO AT YOUR OWN RISK
		//here we call str_from_c to create a tmp string with the base path
		//then we call path join to join the file name and the base path together
		//then we call the file remove function to delete the file from disk
		file_remove(path_join(str_from_c("D:/", allocator_tmp()), entry.name));
		printfmt("{}\n", entry.name);
	}

//don't forget to destruct the array
destruct(content);
```

## Pool

Let's create a pool of points

```C++
struct Point
{
	float x, y;
};


//we create a pool with the element size of sizeof(Point) and a bucket size of 1024 Points
Pool pool = pool_new(sizeof(Point), 1024);

//we get a point from the pool
Point* p1 = (Point*)pool_get(pool);
p1->x = 123;
p1->y = 34;

//put back the point into the pool
pool_put(pool, p1);

//we get another point from the pool
Point* p2 = (Point*)pool_get(pool);
p2->x = 123;
p2->y = 34;

//don't forget to free the pool
pool_free(pool);
```

## Memory_Stream

Let's create an in memory stream and print into it

```C++
//we create an in memory stream
Stream stream = stream_memory_new();

//print into the stream
vprintf(stream, "My name is {}, and my age is {}\n", "Mostafa", "25");

//we print the string content of the stream
printfmt("stream content: '{}'\n", stream_str(stream));

//don't forget to free the stream
stream_free(stream);
```

## Reader

Let's read numbers from a file

```C++
//here we create a new reader that's based on a file stream we open
Reader reader = reader_new(stream_file_new("D:/numbers.txt", IO_MODE::READ, OPEN_MODE::OPEN_ONLY));

//while we can read numbers we print them
int num;
while(vreads(reader, num))
	printfmt("{}\n", num);

//when you free the reader it frees the internal stream we passed in the `reader_new` function
reader_free(reader);
```

## Str_Intern

A String interning is an operation in which all of the unique strings is stored once and every time a duplicate is encountered it returns a pointer to the same stored string it's used mainly to avoid string compare functions since all you have to do now is compare the string pointers if they are the same then they have the same content

```C++
Str_Intern intern = str_intern_new();

const char* is = str_intern(intern, "Mostafa");
if(is == str_intern(intern, "Mostafa"))
	printfmt("They are the same pointer\n");

const char* big_str = "my name is Mostafa";
const char* begin = big_str + 11;
const char* end = begin + 7;
if(is == str_intern(intern, begin, end));
	printfmt("we can intern sub strings too\n");

str_intern_free(intern);
```

## Mutex

Let's use a mutex to protect some function from multiple thread access

```C++
Mutex mtx = mutex_new();

//lock the mutex
mutex_lock(mtx);
	//call the function
	foo();
//unlock the mutex
mutex_unlock(mtx);

//don't forget to free the mutex
mutex_free(mtx);
```