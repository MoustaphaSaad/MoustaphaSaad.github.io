# C Design Pattern: Polymorphism

07, Dec, 2018

Today we'll show a pattern I've found to be useful in the "pure coding style" i've developed over the last eight months.

## Polymorphism

If you come from an OOP world you'll be conditioned to think that polymorphism and inheritance are intertwined. You use inheritance to get polymorphic types, but in reality you couldn't be more wrong.

> I thought of objects being like biological cells and/or individual computers on a network, only able to communicate with messages (so messaging came at the very beginning -- it took a while to see how to do messaging in a programming language efficiently enough to be useful). - Alan Kay

As you can see that the corner stone of OOP is really the messaging system and that objects could in respond to the same messages with a somewhat similar behavior so these types of objects make a type group of some sort.

> The big idea is “messaging” [..]. The Japanese have a small word — ma — for “that which is in between” — perhaps the nearest English equivalent is “interstitial”. The key in making great and growable systems is much more to design how its modules communicate rather than what their internal properties and behaviors should be. - Alan Kay

## C++

C++ interpreted the messages as "member functions" so you now define the object and the set of messages/functions it can receive and these messages tell the object to transform from a valid state to another valid state and you can only interact with objects through messages only to guarantee that objects will be in a valid state at any point in time, which is the whole premise of encapsulation really.

```C++
template<typename T>
class Buf
{
	//encapsulated data which no-one can touch
	Allocator allocator;
	T* ptr;
	size_t cap;
	size_t count;

public:
	//public messages for the outside world
	void
	push(const T& value)
	{
		...
	}
};
```

## Classic OOP Example

Do you remember the classic shape oop example you would find in any OOP book? Let's examine it.

```C++
class Rectangle
{
	float mWidth, mHeight;

public:
	Rectangle()
		:mWidth(0.0f), mHeight(0.0f)
	{}

	Rectangle(float width, float height)
		:mWidth(width), mHeight(height)
	{}

	//Note that Rectangle object responds to the area message
	float
	area() const
	{
		return mWidth * mHeight;
	}
};

class Circle
{
	float mRadius;

public:
	Circle()
		:mRadius(0.0f)
	{}

	Circle(float radius)
		:mRadius(radius)
	{}

	//Note that Circle object responds to the area message also
	float
	area() const
	{
		return PI * mRadius * mRadius;
	}
};
```

These two objects responds to the same "area" message so it's only logical to group them in some type

```C++
class Shape
{
public:
	vritual ~Shape() = 0;
	virtual float area() const = 0;
};
```
Then you'd be able to use the types in this generic way
```C++
void print_area(const Shape* shape)
{
	std::cout << "Area: " << shape->area() << std::endl;
}
```

I have two problems with this style of code

### Focus
It puts the types at the focus of the code, which is evident in the way the code is written, If you want to change/add/remove a message to/from an object you'll need to edit the types themselves. Which is quite not aligned with the spirit of OOP.

In this regard i like the golang approach better.
```go
type Rectangle struct {
	Width, Height float32
}

type Circle struct {
	Radius float32
}

type Shape interface {
	Area() float32
}

func (self Rectangle) area() float32 {
	return self.Width * self.Height
}

func (self Circle) area() float32 {
	return PI * self.Radius * self.Radius
}
```

Golang style completely separates the messages from the objects/data and gives you the freedom to add/remove any message to any type without changing the type itself, which is much closer to the OOP spirit.

### Secerts
The C++ style of the code like to keep secrets. Given a `Shape*` you wouldn't be able to get the type of the object without trial and error through `dynamic_cast`, also this style of code in a non garbage collected language like C++ you would need to allocate the objects on the heap and since there's a difference in the size of each object you won't be able to allocate instances from a pool.

## C Unions
Let's do the same shape example using what's called tagged union pattern

```C++
struct Shape
{
	//Shape kind to indicate the kind/type of this shape
	enum KIND
	{
		//I always start the kind with a KIND_NONE so that when i zero initialize the shape it defaults to an invalid shape
		KIND_NONE,
		KIND_RECTANGLE,
		KIND_CIRCLE
	};

	//kind tag to identify which shape it is
	KIND kind;
	//anonymous union which will hold all the variants of the shapes
	union
	{
		//anonymous struct to hold the data of the rectangle object
		struct {
			float width, height;
		} rectangle;

		//anonymous struct to hold the data of the circle object
		struct {
			float radius;
		} circle;
	};
};

//function constructors
Shape
shape_rectangle(float width, float height)
{
	//i always start my values with {} to zero initialize them
	Shape self{};
	//init tag data
	self.kind = Shape::KIND_RECTANGLE;
	//init the specific object
	self.rectangle.width = width;
	self.rectangle.height = height;
	return self;
}

Shape
shape_circle(float radius)
{
	Shape self{};
	self.kind = Shape::KIND_CIRCLE;
	self.circle.radius = radius;
	return self;
}

float
shape_area(const Shape& self)
{
	//now any function can switch on the kind of the shape
	switch(self.kind)
	{
		case Shape::KIND_RECTANGLE:
			return self.rectangle.width * self.rectangle.height;

		case Shape::KIND_CIRCLE:
			return PI * self.circle.radius * self.circle.radius;

		//i always assert on the default case because theoretically this case should be unreachable
		//but it can be reached if i added another type later to the shape struct so the assert will
		//act as a runtime error to tell me to not forget to handle the new type in this function also
		default:
			assert(false && "UNREACHABLE");
			return 0.0f;
	}
}
```
I came to prefer this style because it doesn't contain secrets at all you can inspect the shape type, create the shape on the stack, and make a shapes object pool since all the shapes has the same memory size.

## Function Overloading

Another simple technique to add to your toolbox regarding polymorphism is the function overloading polymorphism style which the `std::cout` uses to extend it's printing capability

```C++
struct Rectangle
{
	float width, height;
};

float
area(const Rectangle& self)
{
	return self.width * self.height;
}

struct Circle
{
	float radius;
}

float
area(const Circle& self)
{
	return PI * self.radius * self.radius;
}

template<typename T>
void
print_area(const T& shape)
{
	std::cout << area(shape) << std::endl;
}
```
I've had some troubles with this kind of polymorphism regarding type checking and bad error messages since it uses templates. Another issue is that templates adds a lot of overhead in compile times.