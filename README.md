#  Demystifying STL std::map Associative Container

`std::map` is an associative STL(Standard Template Library) container of __Key-Value__ pairs, where the keys are stored in a sorted order. `std::map` doesn't support multiple __Keys__ or same value. The underlying implementation of `std::map` consists of [Red-Black Tree](https://en.wikipedia.org/wiki/Red%E2%80%93black_tree).

`std::map` is one of the widely used STL container and even though the general usage of might be trivial, but the choice of functions and way of usage greatly impacts the performance of the containers, especially with the availability of new member functions as well as with the `move semantics` of `C++11` onwards.

In this tutorial, we'll see the various ways we can use the `std::map` as well as performance aspects of the code

## A generic usage of std::map

Since `std::map` is an associative container with __Key-Value__ pairs, we'll use `int` as __Key__ and a class called `class TestMap` as value for all the examples. The `class TestMap` shall implement `C++11` Rule of 5 with `destructor`, `copy constructor`, `assignment operator`, `move constructor`, and `move assignment operator`. For simplicity purpose, we'll just do a normal `cout` in respective functions. Here is the `TestMap` class

```
class TestMap {
public:
	TestMap() {
		cout << "Constructor... => "  << endl;
	}
	~TestMap() {
		cout << "Destructor... => "<< endl;
	}
	TestMap(const TestMap &) {
		cout<< "Copy Constructor... => " << endl;
	}
	TestMap & operator=(const TestMap & a1) {
		cout << "Assignment Operator... => "<< endl;
		return *this;
	}
	TestMap(const TestMap &&) {
		cout << "Move Constructor => "<< endl;
	}
	TestMap & operator=(const TestMap &&) {
		cout << "Move Assignment Operator => "<< endl;
		return *this;
	}
};

```
Now let's create a `std::map` using `TestMap` as a Value in Map

```
map<int, TestMap> mymap;
mymap[1] = TestMap();

for (auto & mPair : mymap) {
	cout << "Key => " << mPair.first << "  Value => " << (int*)(&mPair.second) << endl;
}

```

The output of the code will be

```

Constructor... =>
Constructor... =>
Move Assignment Operator =>
Destructor... =>
Key => 1  Value => 0x416aa94 // current address of TestMap instance
Destructor... =>

```

Let's decode the output

```
Constructor... =>             // for TestMap() temporary
Constructor... =>             // For map instance
Move Assignment Operator =>   // Move from temporary to map instance
Destructor... =>              // Temporary TestMap() deleted
Key => 1  Value => 0x416aa94  // current address of TestMap instance
Destructor... =>              // Map instance Deleted
```

As we can see that a simple creation of `std::map` calls constructor twice and move assignment operator once. You may now imagine the performance impacts with hundreds / thousands of entries into the map. Let's see what insertion mechanisms are available in `std::map` and what we should use and when

## Inserting Data into std::map
In this section, we shall see usage of various insertion methods of `std::map`, their usage and impacts.

### using subscript []

The very first example we used in this tutorial was using subscript `[]` method. We can create a the __Key-Value__ pair by using providing __Key__ in the subscript `[]` and associating a value with it using equal to `=`. However, it's not necessary to provide the value immediately as `std::map` will anyway create an instance of `TestMap()` value. So the code below is a valid

```
mymap[2];             // Key => 2 and Value => instance of TestMap()
mymap[2] = TestMap(); // Replace the Value with Value => TestMap()

```
This provides an important insight to what happens when we use `[]` to insert the value into the `std::map`. It's instantiated with both __Key__ and __Value__. Any further assignment just changes the value. In the case above where we're using temporary `TestMap()`, the `move assignment operator` gets called. However, if we use an instance like

```
map<int, TestMap> mymap;
TestMap tMap;
mymap[1] = tMap;
```
The normal `assignment operator` shall be called. Even with instance we have to call two constructors and an assignment operator to generate one map entry.

### using insert(...) function

`std::map` provides a built-in function called `insert(...)` which inserts the elements in the container, if it's not already present. `insert(...)` takes a `std::pair` and can be called in two way, either by creating a temporary `std::pair` or by passing an already existing `std::pair`.

```
map<int, TestMap> mymap;
TestMap tMap;
mymap.insert({ 1, tMap });  // Using Temporary pair
pair<int, TestMap> pr = make_pair(2, tMap);
mymap.insert(pr);  // Using Already existing pair

```
The only difference between these two is that `move constructor` should get called with temporary (compiler dependent), otherwise `copy constructor` will get called, as well as in the case of pre-existing pair. The output of constructors shall look like

```
Constructor... =>                       // For TestMap tMap;
Copy Constructor... =>                  // For temporary pair { 1, tMap}
Copy Constructor... => OR Move Constructor... => // For copying to the std::map
Copy Constructor... =>                // For make_pair(2, tMap)
Copy Constructor... =>                // For copying to the std::map

```
Based on the results, we see that both subscript `[]` and `insert(...)` are similar in nature as far as number of calls are concerned barring the difference of `[]` using the assignment operator and `insert(...)` using copy constructor.

### Handling Duplicate values int [] and insert(...) function

As we know that `std::map` doesn't support multiple identical __keys__, however, if we reuse the key with new value, the subscript `[]` will override the existing value with the new one while `insert(...)` will not do the same. There is a special function called `insert_or_assign(...)` which will either insert the __Key__ (if not available already) or update the existing __Value__ with a new one.

```
map<int, int> mx;
mx[1] = 1;
mx[1] = 100;                // Update Value => 100
mx.insert({ 1,1 });
mx.insert({ 1,100 });       // Value will remain 1
mx.insert_or_assign(1,100); // Update Value => 100

```
### using emplace(...) function : Available C++11 onwards

`std::map` also provides a built-in function called `emplace(...)` from `C++11` onwards. It inserts the new elements into the `std::map` by constructing the map in-place with the given arguments.

Let's execute the code below

```
map<int, TestMap> mymap;
mymap.emplace(1, TestMap());
// and
TestMap tMap;
mymap.emplace(2, tMap);

```
The output will be

```
Constructor... =>              // For Temporary TestMap()
Move Constructor =>            // For Creating the Map
// and
Constructor... =>             // For TestMap tMap
Copy Constructor... =>        // For creating the Map

```
As we can see from the output that `emplace(...)` prevents one step by avoiding the creation of `std::map` instance before assigning the values into it. The in place construction may save valuable CPU cycles and should be used wherever possible, especially when we're looking into the scenarios of hundreds and thousands of elements inside the map.

## Removing the elements from std::map

There are three ways to erase elements from the `std::map` using the built-in `erase(....)` function. We'll consider the following code for our erase example

```

map<int, TestMap> mymap;
TestMap tMap;
mymap.emplace(1, tMap);
mymap.emplace(2, tMap);
mymap.emplace(3, tMap);
mymap.emplace(4, tMap);
mymap.emplace(5, tMap);

```

#### erase(....) using iterator

We can call `erase(...)` by passing an iterator of the __Key__ that needs deletion. We can pass the iterator manually or we can use the built-in `find(...)` function to get the iterators for us. For example, if we want to delete the element with __Key__ `3`, we can write the code as follows

```
mymap.erase(mymap.find(3));   // Find returns the Iterator

```

#### erase(....) using the key

We can also call `erase(...)` by directly passing the __Key__ into the function. The function then can be written as

```
mymap.erase(3);  // Delete the map entry with Key => 3

```

#### erase(....) the range of values

In both the cases above, `erase(...)` can erase only one element from the `std::map`, however, it's possible to provide a range in the function which should be deleted. The range variables are inclusive exclusive which means it includes the first, but excludes the second parameter of the range.

```
mymap.erase(mymap.find(2), mymap.find(4)); // Deletes 2, 3
mymap.erase(mymap.find(2), mymap.end()); // Delete from 2 -> end

```

#### Which one to use ?

For deleting the range of values, we have no choice, but `erase(...)` with iterator has amortized constant complexity while `erase(...)` with __Key__ has logarithmic complexity.  However, before deciding to use `erase(...)` with iterator, just remember that `find(...)` which returns the iterator to remove also has logarithmic complexity.

So in general, a deletion will be a logarithmic complexity exercise unless we have an iterator available for deletion.

## std::map Lookup

Faster lookup is one of the primary reasons of using the associative sorted containers like `std::map`. It provides some interesting functionalities for lookup, let's have a look at them

### find(...) and count(...)

We've just saw the usage of `find(...)` while using `erase(...)`, only thing we need to be aware of is that `erase(...)` return iterator to the end of `std::map` if it doesn't find the desired __Key__

```
if (mymap.find(1) == mymap.end())
	cout << "Not Found...." << endl;
```

Similarly `count(...)` return number of __Keys__, if present. Since `std::map` allows only unique __Keys__ to be present in the `std::map`, it will always return `1`, if the __Key__ is found

```
if (mymap.count(100) == 0)
	cout << "Not Found...." << endl;

```
### equal_range(...)

The `std::map` member function `equal_range(...)` returns a `std::pair` which contains the iterator of '2' elements in the map.

- 1. First iterator returns the element which is >= __Input Key__
- 2. Second iterator return the first element which is > __Input Key__

To properly understand this, let's change our `mymap` with values like

```

mymap.emplace(1, tMap);
mymap.emplace(2, tMap);
mymap.emplace(5, tMap);
mymap.emplace(8, tMap);
mymap.emplace(10, tMap);
mymap.emplace(15, tMap);

```
so if we write the code with `equal_range(...)` as

```
pair<map<int, TestMap>::iterator, map<int, TestMap>::iterator> eqRange;
eqRange = mymap.equal_range(8);

```

The output shall contains the following

- 1st Iterator (>= Input Key) will point to `8`
- 2nd Iterator (> Input Key) will point to `10`

However, if we try to find a key which is non-existing in the map, the return value of both iterator will be same

```
eqRange = mymap.equal_range(7);

```

- 1st Iterator (>= Input Key) will point to `8`
- 2nd Iterator (> Input Key) will point to `8`

### lower_bound(...)

The `std::map` built-in function `lower_bound(...)` returns the same value as is returned by the first iterator of `equal_range(...)` function i.e Iterator >= Input Key

```
map<int, TestMap>::iterator it = mymap.lower_bound(7);
// or
map<int, TestMap>::iterator it = mymap.lower_bound(8);
```
The Iterator will return `8`

### upper_bound(...)

The `std::map` built-in function `upper_bound(...)` returns the same value as is returned by the second iterator of `equal_range(...)` function i.e Iterator > Input Key
```
map<int, TestMap>::iterator it = mymap.upper_bound(7);  // return '8'
map<int, TestMap>::iterator it = mymap.upper_bound(8); // return '10'
```

These are some of the major functions one will use while using `std::map` . Hopefully, it will help people in using the `std::map` efficiently.

