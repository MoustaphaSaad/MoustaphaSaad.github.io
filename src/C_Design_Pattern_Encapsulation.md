# C Design Pattern: Encapsulation

28, Dec, 2018

Today we'll show a pattern I've found to be useful in the "pure coding style" I've developed over the last 9 months.

## Encapsulation

Encapsulation is the corner stone of OOP and yet I've yet to find and OOP advocate who understands it, as we've established earlier encapsulation means that an object will encapsulate its state so that no one else can access this state and all you can access from outside the object is an interface of messages you can send to the object which in response it will move from a **valid** state to another **valid** state thus making an API that no one can misuse, and it's easy to replace since all you have to do is substitute the old object with a newer one that can respond to the same messages and you're good to go.

> I thought of objects being like biological cells and/or individual computers on a network, only able to communicate with messages (so messaging came at the very beginning -- it took a while to see how to do messaging in a programming language efficiently enough to be useful). - Alan Kay

## Fake Encapsulation
```C++
class Rectangle
{
	float width, height;

public:
	float getWidth() const { return width; }
	void setWidth(float w) { width = w; }

	float getHeight() const { return height; }
	void setHeight(float h) { height = h; }
};
```
As we you can see the trivial thinking that the above code is "Encapsulated" is not true, you might as well make the width and height members public, all you've done here is expose them with different syntax.

حطيت طبقة عازلة من الفانكشنز
شكرا يا عم الكهربائي

**CAN WE PLEASE STOP TEACHING THAT!!**

## Real Encapsulation: C++

Encapsulation in C++ can be achieved in a nice way i guess but i don't use this because of the overhead it produces for every virtual function call. But i don't mind this style at all since it's really captures the real spirit of the OOP, I only prefer the C version better since it's more boiled down and at the end of the day "to each his own".

**Window.h**

```C++
//you only declare an interface in the header with only public pure virtual functions
class IWindow
{
public:
	virtual Event pollEvent() = 0;
	virtual void minimize() = 0;
	virtual void maximize() = 0;
	virtual void close() = 0;
};

//you can create a window using this function and you don't care what is the
//underlying type of the interface all you care about is that it provides an implementation
//to the above 4 functions
std::unique_ptr<IWindow> make_window(uint32_t width, uint32_t height, const std::string& title);
```

**Window.cpp**
```C++
//we put the implementation of the interface in the cpp since no one really is interested in
//the actual implementation of the interface
class CWindow final: public IWindow
{
	Event pollEvent() override
	{
		...
	}
};

std::unique_ptr<IWindow> make_window(uint32_t w, uint32_t h, const std::string& t)
{
	return std::make_unique<CWindow>(w, h, t);
}
```

## Real Encapsulation: C

The C version of the above code is quite boiled down to the basics and really quite versatile in comparison.

### The Handle

In C++ we defined an interface and handed the user a unique_ptr to this interface.
In C we'll construct what's called a handle which can be a number representing an index in array or a pointer into an object pool or a pointer which is normally allocated using malloc/free

**Window.h**

```C++
//personally i like the handle to be strongly typed so i create a dummy struct
//and typedef a pointer to it to benefit from the strongly typing of C++
//of course you can typedef it to be void* but then you can pass any pointer
//to the window functions
typedef struct Window__{ int dummy; } *Window;

Window window_new(uint32_t width, uint32_t height, const char* title);
void   window_free(Window window);
Event  window_poll(Window window);
void   window_minimize(Window window);
void   window_maximize(Window window);
void   window_close(Window window);
```

**Window.cpp**

```C++
struct Internal_Window
{
	//... your data goes here
};

Window
window_new(uint32_t width, uint32_t height, const char* title)
{
	Internal_Window* self = (Internal_Window*)malloc(sizeof(Internal_Window));
	//init you window here

	//you can cast a pointer to another pointer since they are all of the same type
	//and we merely defined the Window__ type in header for strongly typing purposes
	return (Window)self;
}

void
window_free(Window window)
{
	Internal_Window* self = (Internal_Window*)window;
	//free the window here
	free(self);
}
```

In the above code i choose to allocate the window using the malloc/free but i could also easily created a static window array in the cpp and returned an index to one of it's elements.

## Conclusion

Encapsulation is all about making the API represent the user intent and hiding the implementation into a cpp file where no one else can even reference it.

In practice this type of encapsulation allows you to change the implementation without even recompiling the user code all you have to do is provide the newer dll. Also it results in the most user friendly APIs i've seen.