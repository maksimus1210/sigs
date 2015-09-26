# sigs
Simple thread-safe signal/slot C++11 library, which is templated and include-only. No linking required. Just include the header file "*sigs.h*".

In all its simplicity, the class `sigs::Signal` implements a signal that can be triggered when some event occurs. To receive the signal slots can be connected to it. A slot can be any callable type: lambda, functor, function, or member function. Slots can be disconnected when not needed anymore.

A signal is triggered by invoking its `operator()()` with an optional amount of arguments to be forwarded to each of the connected slots' invocations. But they must conform with the parameter types of `sigs::Signal::SlotType`, which reflects the first template argument given when instantiating a `sigs::Signal`.

# Examples
The most simple use case is having a `void()` invoked:

```c++
sigs::Signal<> s;
s.connect([]{ std::cout << "Hello, signals. I'm an invoked slot.\n"; });
s(); // Trigger it, which will call the function.
```

As mentioned above you can pass arbitrary arguments to the slots but the types will be enforced at compile-time.

```c++
sigs::Signal<std::function<void(int, std::string)>> s;
s.connect([](int n, const std::string &str) {
  std::cout << "I received " << n << " and " << str << std::endl;
});

// Prints "I received 42 and I like lambdas!".
s(42, "I like lambdas!");

// Error! "no known conversion from 'char const[5]' to 'int' for 1st argument".
s("hmm?", "I like lambdas!");
```

Slots can be disconnected by the use of tags. The default tag type is `std::string` and can be retrieved by `sigs::Signal::TagType`.

```c++
sigs::Signal<> s;
s.connect([]{ std::cout << "Hi"; });
s.connect([]{ std::cout << " there!\n"; }, "the tag");

// Prints "Hi there!".
s();

// Disconnect second slot by tag name.
s.disconnect("the tag");

// Prints "Hi".
s();
```

Note that all slots can be disconnected by giving no tag. And since `sigs::Signal::disconnect` takes an `std::initializer_list` you can disconnect several at the same time, e.g. `s.disconnect({"tag1", "tag2"})`.

Slots can be any callable type: lambda, functor, or function. Even member functions.

```c++
void func() {
  std::cout << "Called function\n";
}

class Functor {
public:
  void operator()() {
    std::cout << "Called functor\n";
  }
};

class Foo {
public:
  void test() {
    std::cout << "Called member fuction\n";
  }
};

sigs::Signal<> s;
s.connect(func);
s.connect([]{ std::cout << "Called lambda\n"; });
s.connect(Functor());

Foo foo;
s.connect(&foo, &Foo::test);

s();

/* Prints:
Called function
Called lambda
Called functor
Called member funtion
*/
```

However, if connecting a member function taking one or more arguments then you have to hint at the number (not the type or value).

```c++
class Foo {
public:
  void test(int i, int j) {
    std::cout << "Called member function: i=" << i << ", j=" << j << std::endl;
  }
};

sigs::Signal<std::function<void(int,int)>> s;

Foo foo;
s.connect(&foo, &Foo::test, "tag", std::placeholders::_1, std::placeholders::_2);

// Prints "Called member function: i=42, j=1105".
s(42, 1105);
```

Note the use of `std::placeholders::_1` and `std::placeholders::_2` because `Foo::test` takes two arguments, and that the tag becomes required.

Another useful feature is the ability to connect signals to signals. If a first signal is connected to a second signal, and the second signal is triggered, then all of the slots of the first signal are triggered as well - and with the same arguments.

```c++
sigs::Signal<> s1;
s1.connect([]{ std::cout << "Hello 1 from s1\n"; });
s1.connect([]{ std::cout << "Hello 2 from s1\n"; });

sigs::Signal<> s2;
s2.connect(s1);

s2();

/* Prints:
Hello 1 from s1
Hello 2 from s1
*/
```
