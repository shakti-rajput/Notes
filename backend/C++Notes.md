# C++ Interview Notes

## 1. Dangling Pointer

- Pointer to invalid/dead object (object that is not in the access limit of the program)
- Memory freed, pointer still exists

## 2. Memory Leak

- Pointer lost, memory still allocated to the program

## 3. Stack

- Automatic lifetime
- Local variables commonly live here

## 4. Heap

- Dynamically allocated

```cpp
int* p = new int(10);
```

- Explicit/managed lifetime
- `new`/allocators dynamically allocate
- Must be managed, preferably via RAII

**RAII** stands for **Resource Acquisition Is Initialization**.

RAII = ownership tied to object lifetime → destructor automatically cleans up the resource.

## 5. Smart Pointers

- They reduce memory leaks and dangling-pointer risks and make ownership semantics explicit.
- `unique_ptr`, `shared_ptr`, `weak_ptr`
- Smart pointers are primarily about ownership.
- Raw pointers can be used when you don't own the object.

## 6. `unique_ptr`

```cpp
auto p = std::make_unique<int>(10);
```

You cannot copy it:

```cpp
auto p2 = p;  // ❌
```

But you can move ownership:

```cpp
auto p2 = std::move(p);  // ✅
```

## 7. `shared_ptr`

```cpp
auto p1 = std::make_shared<int>(10);
```

- The object is destroyed when the last `shared_ptr` owning it is destroyed.
- You can copy it:

```cpp
auto p2 = p1;
```

## 8. `weak_ptr`

```cpp
auto shared = make_shared<int>(10);
weak_ptr<int> weak = shared;
```

- Used with `shared_ptr` when you want to observe an object without owning it.
- It doesn't increase the reference count.
- It's especially useful for avoiding reference cycles.

## 9. Race Condition

## 10. Atomic

```cpp
atomic<int> counter;
```

## 11. Mutex and Thread Synchronization

### `std::mutex`

``` cpp
#include <mutex>

std::mutex m;
int counter = 0;

void increment()
{
    m.lock();

    counter++;

    m.unlock();
}
```

### Using `std::lock_guard`

``` cpp
void increment()
{
    std::lock_guard<std::mutex> lock(m);

    counter++;
}
```

### Using `std::unique_lock`

``` cpp
void increment()
{
    std::unique_lock<std::mutex> lock(m);

    counter++;
}
```

------------------------------------------------------------------------

### `std::recursive_mutex`

A `recursive_mutex` allows the same thread to lock the same mutex
multiple times.

#### Manual `lock()` / `unlock()`

``` cpp
std::recursive_mutex m;

void foo()
{
    m.lock();

    bar();

    m.unlock();
}

void bar()
{
    m.lock();

    // same thread locks m again

    m.unlock();
}
```

#### Using `std::lock_guard`

``` cpp
std::recursive_mutex m;

void foo()
{
    std::lock_guard<std::recursive_mutex> lock(m);

    bar();
}

void bar()
{
    std::lock_guard<std::recursive_mutex> lock(m);

    // same thread can lock it again
}
```

The first `lock_guard` locks it once.

The second `lock_guard` locks it a second time.

The recursive mutex keeps track of this.

#### Using `std::unique_lock`

``` cpp
void foo()
{
    std::unique_lock<std::recursive_mutex> lock(m);

    bar();
}

void bar()
{
    std::unique_lock<std::recursive_mutex> lock(m);

    // allowed
}
```

------------------------------------------------------------------------

### `std::timed_mutex`

A `timed_mutex` allows you to try to acquire the mutex for a limited
amount of time.

#### Manual-style locking

``` cpp
std::timed_mutex m;

if (m.try_lock_for(std::chrono::seconds(1))) {

    // got the lock

    m.unlock();

} else {

    // couldn't get lock within 1 second
}
```

#### Using `std::unique_lock`

``` cpp
std::timed_mutex m;

void foo()
{
    std::unique_lock<std::timed_mutex> lock(
        m,
        std::defer_lock
    );

    if (lock.try_lock_for(std::chrono::seconds(1))) {

        // got the lock

    } else {

        // timeout
    }
}
```

------------------------------------------------------------------------

### `std::recursive_timed_mutex`

This combines:

-   Recursive locking
-   Timed locking

#### Manual locking

``` cpp
std::recursive_timed_mutex m;

if (m.try_lock_for(std::chrono::seconds(1))) {

    // got lock

    m.unlock();
}
```

#### Because it is recursive

``` cpp
void foo()
{
    m.lock();

    bar();

    m.unlock();
}

void bar()
{
    m.lock();

    // same thread can lock again

    m.unlock();
}
```

#### Using `std::unique_lock`

``` cpp
std::recursive_timed_mutex m;

void foo()
{
    std::unique_lock<std::recursive_timed_mutex> lock(
        m,
        std::defer_lock
    );

    if (lock.try_lock_for(std::chrono::seconds(1))) {

        bar();

    } else {

        // couldn't get lock
    }
}
```

------------------------------------------------------------------------

### `std::shared_mutex`

A `shared_mutex` supports multiple readers but only one writer at a
time.

``` text
             shared_mutex
                 │
        ┌────────┴────────┐
        │                 │
     READERS            WRITER
        │                 │
   multiple allowed    one allowed
```

#### Manual exclusive lock

For writing:

``` cpp
std::shared_mutex m;
int value = 0;

void write()
{
    m.lock();

    value++;

    m.unlock();
}
```

Only one writer can enter.

#### Using `std::unique_lock`

``` cpp
void write()
{
    std::unique_lock<std::shared_mutex> lock(m);

    value++;
}
```

#### Manual shared locking

For reading, we don't want to block other readers.

``` cpp
std::shared_mutex m;

void read()
{
    m.lock_shared();

    std::cout << value;

    m.unlock_shared();
}
```

Multiple threads can do this simultaneously.

``` text
Reader 1 ──┐
Reader 2 ──┼── READ ──> allowed
Reader 3 ──┘
```

#### Using `std::shared_lock`

Much cleaner:

``` cpp
void read()
{
    std::shared_lock<std::shared_mutex> lock(m);

    std::cout << value;
}
```


## Lock Management

### `std::lock_guard`

``` cpp
std::lock_guard<std::mutex> lock(m);
```
It automatically unlocks the mutex when it goes out of scope.

------------------------------------------------------------------------

### `std::unique_lock`

``` cpp
std::unique_lock<std::mutex> lock(m);
```

``` cpp
std::unique_lock<std::mutex> lock(m);

lock.unlock();

// do something without the mutex

lock.lock();

// protected again
```

------------------------------------------------------------------------

### `std::shared_lock`

Used for read locking with `std::shared_mutex`.

``` cpp
std::shared_lock<std::shared_mutex> lock(m);
```

Multiple threads can hold shared locks simultaneously.

------------------------------------------------------------------------

## Why Use Lock Management?

A mutex provides the actual synchronization. Lock-management classes
make locking safer and easier.

The mutex is automatically unlocked when `lock` goes out of scope.

This is RAII:

``` text
constructor → lock mutex
destructor  → unlock mutex
```

This also protects against early returns and exceptions.

### Example of the problem with manual locking

``` cpp
void increment()
{
    m.lock();

    counter++;

    if (counter == 1)
        return;       // forgot m.unlock()

    m.unlock();
}
```

The mutex can remain locked.

With `lock_guard`:

``` cpp
void increment()
{
    std::lock_guard<std::mutex> lock(m);

    counter++;

    if (counter == 1)
        return;
}
```

The mutex is automatically unlocked when `lock` is destroyed.

------------------------------------------------------------------------

## `std::condition_variable`

A `condition_variable` allows a thread to sleep until a condition may
have become true.

Example:

``` cpp
std::mutex m;
std::condition_variable cv;
std::queue<int> queue;
```

Consumer:

``` cpp
void consumer()
{
    while (true)
    {
        std::unique_lock<std::mutex> lock(m);

        cv.wait(lock, [] {
            return !queue.empty();
        });

        int item = queue.front();
        queue.pop();

        lock.unlock();

        consumeItem(item);
    }
}
```

Producer:

``` cpp
void producer()
{
    while (true)
    {
        int item = produceItem();

        {
            std::lock_guard<std::mutex> lock(m);
            queue.push(item);
        }

        cv.notify_one();
    }
}
```

The wait condition here is:

``` cpp
!queue.empty()
```

Meaning:

> "Continue only when the queue is not empty."

### Multiple wait conditions

You can have multiple conditions:

``` cpp
cv.wait(lock, [] {
    return !queue.empty() || shutdown;
});
```
------------------------------------------------------------------------

## `std::atomic`

`std::atomic` is not a mutex.

It is used when simple shared state can be manipulated atomically.

``` cpp
#include <atomic>

std::atomic<int> counter{0};

void increment()
{
    counter++;
}
```

For a simple counter, a mutex may not be necessary.

------------------------------------------------------------------------

# Complete Thread Synchronization Picture

``` text
THREAD SYNCHRONIZATION
│
├── MUTEX
│   │
│   ├── std::mutex
│   │      │
│   │      ├── m.lock()
│   │      │   ...
│   │      │   m.unlock()
│   │      │
│   │      ├── lock_guard
│   │      └── unique_lock
│   │
│   ├── recursive_mutex
│   │      │
│   │      ├── m.lock()
│   │      ├── lock_guard
│   │      └── unique_lock
│   │
│   ├── timed_mutex
│   │      │
│   │      ├── m.try_lock_for()
│   │      └── unique_lock
│   │
│   ├── recursive_timed_mutex
│   │      │
│   │      └── unique_lock
│   │
│   └── shared_mutex
│          │
│          ├── WRITE
│          │    └── unique_lock
│          │
│          └── READ
│               └── shared_lock
│
├── ATOMIC
│
└── CONDITION_VARIABLE
```


## 12. Semaphore
```cpp
#include <iostream>
#include <semaphore>
#include <thread>
#include <vector>
#include <chrono>

std::counting_semaphore<3> dbSemaphore(3);

void useDatabase(int id)
{
    dbSemaphore.acquire();

    std::cout << "Thread " << id << " using database\n";

    std::this_thread::sleep_for(std::chrono::seconds(2));

    std::cout << "Thread " << id << " finished\n";

    dbSemaphore.release();
}

int main()
{
    std::vector<std::thread> threads;

    for (int i = 1; i <= 6; i++)
    {
        threads.emplace_back(useDatabase, i);
    }

    for (auto& t : threads)
    {
        t.join();
    }
}
```

## 13. `transform`

```cpp
transform(v.begin(), v.end(), out.begin(), square)
```

- `square` is a function which accepts one `int` value.
- This works in parallel.

## 14. `vector<bool>`

- `vector<bool>` is different.
- It uses a proxy.
- So you cannot:

```cpp
bool& x = v[0];  // ❌
```

- But:

```cpp
cout << v[0];  // ✅
```

## 15. Thread Pool

- Worker threads
- Task queue
- Mutex
- Condition variable
- Shutdown mechanism
// C++
boost::asio::thread_pool pool(3);

boost::asio::post(pool, [] { doWork(1); });
boost::asio::post(pool, [] { doWork(2); });
boost::asio::post(pool, [] { doWork(3); });

pool.join();

## 16. Lvalues / Rvalues

- Lvalues can take only lvalues unless there is `const`.
- `const` can take rvalues.
- Rvalues only take temporary rvalues and can steal the resources of temporary rvalues.

## 17. Double-Checked Locking

- Naive double-checked locking is unsafe → proper memory ordering/publication isn't guaranteed.

Use `call_once` or:

```cpp
static Singleton& getInstance() {
    static Singleton instance;
    return instance;
}
```

## 18. Name Mangling

- Name mangling is for acheving overloading in C++. as compiler at compile time may name `fun(int z)` as `funci(int z)` and `fun(double z)` as `fund(double z)`

## 19. `push_back` vs `emplace_back`

```cpp
v.push_back(Person("Rupesh", 35));
```

→ First object is created, then it is moved or copied to the vector.

```cpp
v.emplace_back("Rupesh", 35);
```

→ The object is created only once.

## 20. `std::function`

- Holds the function and it is type saved.

## 21. Static Variable

- Static variable is not thread safe.

## 22. Operator Overloading

- Only when it makes sense.

## 23. Friend

- `friend` class can access all the variables and functions of what the other class can access.
- For testing, you can use the `friend` keyword without messing with the original class.
- If there are, let's say, 20 classes and we want to have a mediator of those 20 classes, we can use the `friend` keyword.

## 24. Reference

```cpp
int &r = i;
```

- `r` is an alias of `i`.
- `&r` represents the address.
- `r++` can be done.
- A reference cannot be reseated. It is final; you cannot change it.
- So initializing as `null` also cannot be done.
- Arithmetic operations cannot be applied to variable `r` but `&r` represents the value `i` so can be done like:

```cpp
(&r)++
```

## 25. Factory Design Pattern

The Factory Design Pattern separates object creation from the main application logic.

Instead of having the Amazon class directly create every type of product, a separate `ProductFactory` handles the creation.

This way, when a new product is added, the object-creation logic is handled in the factory rather than modifying the main Amazon class.

## 26. Virtual Destructor

A base class needs a virtual destructor when you might delete a derived object through a base-class pointer.

If you're not going to delete derived objects through a base pointer, a virtual destructor may not be necessary.

```cpp
Animal* animal = new Dog(); 
delete animal;
```

Here the base destructor should be virtual.

## 27. Using namespace std

Using namespace std is considered bad because if u import some library that has the same function as the same name lets say as `cout` then it will be bad as we will have conflict

## 28. Return Value Of `printf` And `scanf`

Return Value Of `printf` And `scanf` is the number of character `printf` is printing to console or how many variables it is reading from the console.

## 29. Copy Constructor

```cpp
Foo(const Foo &bar)
```

it is const because we dont want to modify the value and it is passed by refrence so that it will not go on recursively calling the copy constructor

## 30. Code Bloating

Do not declare unnecessary variable or inline function if it is used in multiple places.

## 31. Placement New

```cpp
Foo* p = new (memory) Foo();
```

Construct a Foo object at the memory address `memory`. Need manual destructor:

```cpp
p->~Foo();
```

## 32. Override Keyword

Override keyword is used explicitly just to remove the confusion if mistakenly.

## 33. Polymorphism

### 1. Compile-time polymorphism

- Function overloading
- Operator overloading
- Templates

### 2. Runtime polymorphism ⭐

- **Inheritance** - Inheritance creates the relationship,
- **Virtual functions** - virtual enables runtime dispatch
- **Function overriding** - overriding provides the derived behavior
- **Base pointer/reference** - the base pointer/reference gives us the common interface through which runtime polymorphism is demonstrated.

## 34. `final` Keyword

U can use final keyword to stop inheriting your class

## 35. `typeid`

```cpp
typeid(obj)
```

to check if two objects are of same class or not.

## 36. Struct vs Class

When it is about struct either variable, function, or inheritance it is public but class is private.

## 37. Vector

Vector -> It gives advantage of array and linkedlist both

## 38. Function Chaining

To attain function chaining u have to return `*this`;

```cpp
b.seta(5).setb(19).print()
```

when first `seta(5)` was called we didnt return the actual object we return the copy of object

## 39. Enum

Plain enum can be interpreted as index 0 to the first value while class enum u have to explicitly pass the value

## 40. Stop Someone From Copying Your Objects

1. Keep your constructor and assignment operator as private in your class.
2. Inherit dummy class with private copy constructor and assignment operator.
3. `= delete` to your constructor and assignment constructor.
