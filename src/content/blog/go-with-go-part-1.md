---
author: Saurabh Dalakoti
pubDatetime: 2024-06-23T12:39:57Z
modDatetime: 2024-04-21T19:32:03Z
title: Go With Go Part 1
featured: false
draft: false
tags:
  - Backend
  - Go
description: Diving into Go Language Part 1
---

# TLDR

These are the book notes taken from [Learning-GO](https://www.oreilly.com/library/view/learning-go/9781492077206/) book.

What this article is about

- Outlining some of the difference from other languages
- Outlining go approach of doing things apart from other languages like Python, c++, JS
- Its not a complete guide or reference on how you should write code in Go.
- Go's approach on
  - 3rd party tooling
  - formatting
  - builds and automation
- types
- strings, maps, structs
- flow control box shadowing, iteration
- functions, closures, pointers
- Embedding, Interfaces, Runtime

# 3rd Party Go Tools

Go’s method for publishing code is a bit different than most other languages. Go
developers don’t rely on a centrally hosted service, like Maven Central for Java or the
NPM registry for JavaScript. Instead, they share projects via their source code reposi‐
tories. The go install command takes an argument, which is the location of the
source code repository of the project you want to install, followed by an @ and the
version of the tool you want (if you just want to get the latest version, use @latest). It
then downloads, compiles, and installs the tool into your $GOPATH/bin directory.

# Formatting your Code

One of the chief design goals for Go was to create a language that allowed you to
write code efficiently. This meant having simple syntax and a fast compiler. It also led
Go’s authors to reconsider code formatting. Most languages allow a great deal of flexi‐
bility on how code is laid out. Go does not. Enforcing a standard format makes it a
great deal easier to write tools that manipulate source code. This simplifies the com‐
piler and allows the creation of some clever tools for generating code.
There is a secondary benefit as well. Developers have historically wasted extraordi‐
nary amounts of time on format wars. Since Go defines a standard way of formatting
code, Go developers avoid arguments over One True Brace Style and Tabs vs. Spaces,
For example, Go programs use tabs to indent, and it is a syntax error if the opening
brace is not on the same line as the declaration or command that begins the block.

> The Go development tools include a command, go fmt, which automatically reformats your code to match the standard format. It does things like fixing up the white‐
> space for indentation, lining up the fields in a struct, and making sure there is proper spacing around operators.

# Builds with Makefiles

Modern software development relies on repeatable, automatable builds that can be run by anyone, anywhere, at any time. This avoids the age-old developer excuse of “It works on my machine!” The way to do this is to use some kind of script to specify your build steps. Go developers have adopted make as their solution.

```shell

.DEFAULT_GOAL := build

fmt:
	go fmt ./...
.PHONY:fmt

lint: fmt
	golint ./...
.PHONY:lint

vet: fmt
	go vet ./...
.PHONY:vet

build: vet
	go build hello.go
.PHONY:build

```

Even if you haven’t seen a Makefile before, it’s not too difficult to figure out what is
going on. Each possible operation is called a target. The .DEFAULT_GOAL defines which target is run when no target is specified. In our case, we are going to run the build target. Next we have the target definitions. The word before the colon (:) is the name of the target. Any words after the target (like vet in the line build: vet) are the
other targets that must be run before the specified target runs. The tasks that are per‐
formed by the target are on the indented lines after the target. (The .PHONY line keeps
make from getting confused if you ever create a directory in your project with the
same name as a target.)

Once the Makefile is in the ch1 directory, type: `make`

You should see the following output:

```txt
go fmt ./...
go vet ./...
go build hello.go
```

By entering a single command, we make sure the code was formatted correctly, vet
the code for nonobvious errors, and compile. We can also run the linter with make
lint, vet the code with make vet, or just run the formatter with make fmt. This
might not seem like a big improvement, but ensuring that formatting and vetting
always happen before a developer (or a script running on a continuous integration
build server) triggers a build means you won’t miss any steps.

# Types

## Zero Value

Go, like most modern languages, assigns a default zero value to any variable that is
declared but not assigned a value. Having an explicit zero value makes code clearer
and removes a source of bugs found in C and C++ programs. As we talk about each
type, we will also cover the zero value for the type.

## Special Integer type

- A byte is an alias for uint8; it is legal to assign, compare, or perform mathematical operations between a byte and a uint8. However, you rarely see uint8 used in Go code; just call it a byte.

## Arrays

- declaring arrays
- `var x = [3]int{10, 20, 30}`
- `var x = [...]int{10, 20, 30}`

## Slice

Most of the time, when you want a data structure that holds a sequence of values, a
slice is what you should use. What makes slices so useful is that the length is not part
of the type for a slice. This removes the limitations of arrays.

> Go is a call by value language.

The rules as of Go 1.14 are to double the size of the slice when the capacity is less than 1,024 and then grow by at least 25% afterward.

```go
var x []int
fmt.Println(x, len(x), cap(x))
x = append(x, 10)
fmt.Println(x, len(x), cap(x))
x = append(x, 20)
fmt.Println(x, len(x), cap(x))
x = append(x, 30)
fmt.Println(x, len(x), cap(x))
x = append(x, 40)
fmt.Println(x, len(x), cap(x))
x = append(x, 50)
fmt.Println(x, len(x), cap(x))
```

When you build and run the code, you’ll see the following output. Notice how and
when the capacity increases:

```shell
[] 0 0
[10] 1 1
[10 20] 2 2
[10 20 30] 3 4
[10 20 30 40] 4 4
[10 20 30 40 50] 5 8
```

While it’s nice that slices grow automatically, it’s far more efficient to size them once.
If you know how many things you plan to put into a slice, create the slice with the
correct initial capacity. We do that with the make function.

### Slicing Slices

A slice expression creates a slice from a slice. It’s written inside brackets and consists of a starting offset and an ending offset, separated by a colon (:). If you leave off the
starting offset, 0 is assumed.

```go
x := []int{1, 2, 3, 4}
y := x[:2]
z := x[1:]
d := x[1:3]
e := x[:]
fmt.Println("x:", x)
fmt.Println("y:", y)
fmt.Println("z:", z)
fmt.Println("d:", d)
fmt.Println("e:", e)
```

### Slices share storage sometimes

When you take a slice from a slice, you are not making a copy of the data. Instead,
you now have two variables that are sharing memory. This means that changes to an
element in a slice affect all slices that share that element. Let’s see what happens when
we change values.

To avoid complicated slice situations, you should either never use append with a sub‐
slice or make sure that append doesn’t cause an overwrite by using a full slice expres‐
sion. This is a little weird, but it makes clear how much memory is shared between
the parent slice and the subslice.

### using copy

If you need to create a slice that’s independent of the original, use the built-in
copy function.
The copy function takes two parameters. The first is the destination slice and the sec‐
ond is the source slice. It copies as many values as it can from source to destination,
limited by whichever slice is smaller, and returns the number of elements copied. The
capacity of x and y doesn’t matter; it’s the length that’s important.

```go
x := []int{1, 2, 3, 4}
y := make([]int, 2)
num = copy(y, x)
```

The variable y is set to [1 2] and num is set to 2.

## Strings and runes and bytes

You might think that a string in Go is made out of runes, but that’s not the case. Under the covers, Go uses a sequence of bytes to represent a string.

### Immutability

Since strings are immutable, they don’t have the modification problems that slices of slices do. There is a different problem, though. A string is composed of a sequence of bytes, while a code point in UTF-8 can be anywhere from one to four bytes long. Our previous example was entirely composed of code points that are one byte long in UTF-8, so everything worked out as expected. But when dealing with languages other than English or with emojis, you run into code points that are multiple bytes long in UTF-8:

```go
var s string = "Hello ☀️"
var s2 string = s[4:7]
var s3 string = s[:5]
var s4 string = s[6:]
```

In this example, s3 will still be equal to “Hello.” The variable s4 is set to the sun emoji.
But s2 is not set to “o ☀️ ” Instead, you get “o .” That’s because we only copied the
first byte of the sun emoji’s code point, which is invalid.

Go allows you to pass a string to the built-in len function to find the length of the
string. Given that string index and slice expressions count positions in bytes, it’s not
surprising that the length returned is the length in bytes, not in code points:

```go
var s string = "Hello ☀️"
fmt.Println(len(s))
```

This code prints out 10, not 7, because it takes four bytes to represent the sun with
smiling face emoji in UTF-8.

> Even though Go allows you to use slicing and indexing syntax with strings, you should only use it when you know that your string only contains characters that take up one byte. WHICH IS NOT TRUE IN CASE Of NONENGLISH & EMOJIS.

## Maps

`totalWins := map[string]int{}`

```go
teams := map[string][]string {
"Orcas": []string{"Fred", "Ralph", "Bijou"},
"Lions": []string{"Sarah", "Peter", "Billie"},
"Kittens": []string{"Waldo", "Raul", "Ze"},
}
```

`ages := make(map[int][]string, 10)`

Maps are like slices in several ways:

- Maps automatically grow as you add key-value pairs to them.
- If you know how many key-value pairs you plan to insert into a map, you can use
  make to create a map with a specific initial size.
- Passing a map to the len function tells you the number of key-value pairs in a
  map.
- The zero value for a map is nil.
- Maps are not comparable. You can check if they are equal to nil, but you cannot
  check if two maps have identical keys and values using == or differ using !=.

### Comma ok Idiom

A map returns the zero value if you ask for the value associated with a key that’s not in the map. This is handy when implementing things like the counter we saw earlier. However, you sometimes do need to find out if a key is in a map. Go provides the comma ok idiom to tell the difference between a key that’s associated with a zero value and a key that’s not in the map.

Rather than assign the result of a map read to a single variable, with the comma ok
idiom you assign the results of a map read to two variables. The first gets the value
associated with the key. The second value returned is a bool. It is usually named ok. If
ok is true, the key is present in the map.

### Using Maps as set

Many languages include a set in their standard library. A set is a data type that ensures
there is at most one of a value, but doesn’t guarantee that the values are in any partic‐
ular order. Checking to see if an element is in a set is fast, no matter how many ele‐
ments are in the set. (Checking to see if an element is in a slice takes longer as you
add more elements to the slice.)
Go doesn’t include a set, but you can use a map to simulate some of its features.

```go

intSet := map[int]bool{}
vals := []int{5, 10, 2, 5, 8, 7, 3, 9, 1, 2, 10}
for _, v := range vals {
	intSet[v] = true
}
fmt.Println(len(vals), len(intSet))
fmt.Println(intSet[5])
fmt.Println(intSet[500])
if intSet[100] {
	fmt.Println("100 is in the set")
}

```

> If you need sets that provide operations like union, intersection, and subtraction, you
> can either write one yourself or use one of the many third-party libraries that provide
> the functionality.

Some people prefer to use struct{} for the value when a map is being used to implement a set. (We’ll discuss structs in the next section.) The advantage is that an empty struct uses zero bytes, while a boolean uses one byte.

The disadvantage is that using a struct{} makes your code more clumsy. You have a less obvious assignment, and you need to use the comma ok idiom to check if a value is in the set:

```go
intSet := map[int]struct{}{}
vals := []int{5, 10, 2, 5, 8, 7, 3, 9, 1, 2, 10}
for _, v := range vals {
	intSet[v] = struct{}{}
}
if _, ok := intSet[5]; ok {
	fmt.Println("5 is in the set")
}
```

Unless you have very large sets, it is unlikely that the difference in memory usage is significant enough to outweigh the disadvantages.

### Iterate over map

```go
m := map[string]int{
	"a": 1,
	"c": 3,
	"b": 2,
}
for i := 0; i < 3; i++ {
	fmt.Println("Loop", i)
	for k, v := range m {
		fmt.Println(k, v)
	}
}

```

The order of the keys and values varies; some runs may be identical. This is actually a
security feature. In earlier Go versions, the iteration order for keys in a map was usu‐
ally (but not always) the same if you inserted the same items into a map. This caused
two problems:

- People would write code that assumed that the order was fixed, and this would
  break at weird times.
- If maps always hash items to the exact same values, and you know that a server is storing some user data in a map, you can actually slow down a server with an attack called Hash DoS by sending it specially crafted data where all of the keys hash to the same bucket.

## Structs

Maps are a convenient way to store some kinds of data, but they have limitations.
They don’t define an API since there’s no way to constrain a map to only allow certain keys. Also, all of the values in a map must be of the same type. For these reasons,
maps are not an ideal way to pass data from function to function. When you have
related data that you want to group together, you should define a struct.

Most languages have a concept that’s similar to a struct, and the syntax that Go uses
to read and write structs should look familiar:

```go
type person struct {
	name string
	age int
	pet string
}
```

Once a struct type is declared, we can define variables of that type:
`var fred person`
Here we are using a var declaration. Since no value is assigned to fred, it gets the
zero value for the person struct type. A zero value struct has every field set to the
field’s zero value.

```go
beth := person{
age: 30,
name: "Beth",
}
```

### Anonymous struct

```go
var person struct {
	name string
	age int
	pet string
}

person.name = "bob"
person.age = 50
person.pet = "dog"

pet := struct {
	name string
	kind string
}{
	name: "Fido",
	kind: "dog",
}
```

### Comparing and Converting Structs

> Unlike in Python or Ruby, in Go there’s no magic method that can be overridden to
> redefine equality and make == and != work for incomparable structs. You can, of
> course, write your own function that you use to compare structs.

> Anonymous structs add a small twist to this: if two struct variables are being compared and at least one of them has a type that’s an anonymous struct, you can compare them without a type conversion if the fields of both structs have the same names, order, and types. You can also assign between named and anonymous struct types if the fields of both structs have the same names, order, and types:

# Control Flow, Blocks Shadowing, iteration

## Universal Block

There’s actually one more block that is a little weird: the universe block. Remember, Go is a small language with only 25 keywords. What’s interesting is that the built-in types (like int and string), constants (like true and false), and functions (like make or close) aren’t included in that list. Neither is nil. So, where are they?

Rather than make them keywords, Go considers these predeclared identifiers and defines them in the universe block, which is the block that contains all other blocks.

Because these names are declared in the universe block, it means that they can be shadowed in other scopes. You can see this happen by running the code

```go
fmt.Println(true)
true := 10
fmt.Println(true)
When you run it, you’ll see:
true
10
```

When you run it, you’ll see:

```txt
true
10
```

You must be very careful to never rede"ne any of the identi"ers in the universe block. If
you accidentally do so, you will get some very strange behavior. If you are lucky, you’ll
get compilation failures. If you are not, you’ll have a harder time tracking down the
source of your problems.

You might think that something this potentially destructive would be caught by lint‐
ing tools. Amazingly, it isn’t. Not even shadow detects shadowing of universe block
identifiers.

## Special Scoping inside if block

```go
if n := rand.Intn(10); n == 0 {
	fmt.Println("That's too low")
} else if n > 5 {
	fmt.Println("That's too big:", n)
} else {
	fmt.Println("That's a good number:", n)
}
```

Having this special scope is very handy. It lets you create variables that are available
only where they are needed. Once the series of if/else statements ends, n is unde‐
fined.

## Iteration

- The for-range value is a copy

## Switch

```go
words := []string{"a", "cow", "smile", "gopher", "octopus", "anthropologist"}
for _, word := range words {
	switch size := len(word); size {
		case 1, 2, 3, 4:
			fmt.Println(word, "is a short word!")
		case 5:
			wordLen := len(word)
			fmt.Println(word, "is exactly the right length:", wordLen)
		case 6, 7, 8, 9:
		default:
			fmt.Println(word, "is a long word!")
	}
}
```

When we run this code, we get the following output:

```txt
a is a short word!
cow is a short word!
smile is exactly the right length: 5
anthropologist is a long word!
```

## Go to

There is a fourth control statement in Go, but chances are, you will never use it. Ever
since Edgar Dijkstra wrote “Go To Statement Considered Harmful” in 1968, the goto
statement has been the black sheep of the coding family. There are good reasons for
this. Traditionally, goto was dangerous because it could jump to nearly anywhere in a
program; you could jump into or out of a loop, skip over variable definitions, or into
the middle of a set of statements in an if statement. This made it difficult to under‐
stand what a goto-using program did.

# Functions

## Simulating Named and Optional Parameters

Before we get to the unique function features that Go has, let’s mention two that Go
doesn’t have: named and optional input parameters. With one exception that we will
cover in the next section, you must supply all of the parameters for a function. If you
want to emulate named and optional parameters, define a struct that has fields that
match the desired parameters, and pass the struct to your function. Example 5-1
shows a snippet of code that demonstrates this pattern.

```go
type MyFuncOpts struct {
	FirstName string
	LastName string
	Age int
}

func MyFunc(opts MyFuncOpts) error {
	// do something here
}

func main() {
	MyFunc(MyFuncOpts {
		LastName: "Patel",
		Age: 50,
	})
	My Func(MyFuncOpts {
		FirstName: "Joe",
		LastName: "Smith",
	})
}
```

> In practice, not having named and optional parameters isn’t a limitation. A function
> shouldn’t have more than a few parameters, and named and optional parameters are
> mostly useful when a function has many inputs. If you find yourself in that situation,
> your function is quite possibly too complicated.

## Variadic Input Parameters and Slices

```go
func addTo(base int, vals ...int) []int {
	out := make([]int, 0, len(vals))
	for _, v := range vals {
		out = append(out, base+v)
	}
	return out
}

func main() {
	fmt.Println(addTo(3))
	fmt.Println(addTo(3, 2))
	fmt.Println(addTo(3, 2, 4, 6, 8))
	a := []int{4, 3}
	fmt.Println(addTo(3, a...))
	fmt.Println(addTo(3, []int{1, 2, 3, 4, 5}...))
}

```

## Multiple Return Values

```go
func divAndRemainder(numerator int, denominator int) (int, int, error) {
	if denominator == 0 {
		return 0, 0, errors.New("cannot divide by zero")
	}
	return numerator / denominator, numerator % denominator, nil
}
```

## Functions Are Values

Just like in many other languages, functions in Go are values. The type of a function
is built out of the keyword func and the types of the parameters and return values.
This combination is called the signature of the function. Any function that has the
exact same number and types of parameters and return values meets the type
signature.
Having functions as values allows us to do some clever things, such as build a primi‐
tive calculator using functions as values in a map. Let’s see how this works.

```go
func add(i int, j int) int { return i + j }
func sub(i int, j int) int { return i - j }
func mul(i int, j int) int { return i * j }
func div(i int, j int) int { return i / j }
```

Next, we create a map to associate a math operator with each function:

```go
var opMap = map[string]func(int, int) int{
"+": add,
"-": sub,
"*": mul,
"/": div,
}

func main() {
	expressions := [][]string{
		[]string{"2", "+", "3"},
		[]string{"2", "-", "3"},
		[]string{"2", "*", "3"},
		[]string{"2", "/", "3"},
		[]string{"2", "%", "3"},
		[]string{"two", "+", "three"},
		[]string{"5"},
	}
	for _, expression := range expressions {
		if len(expression) != 3 {
			fmt.Println("invalid expression:", expression)
			continue
		}
		p1, err := strconv.Atoi(expression[0])
		if err != nil {
			fmt.Println(err)
			continue
		}
		op := expression[1]
		opFunc, ok := opMap[op]
		if !ok {
			fmt.Println("unsupported operator:", op)
			continue
		}
		p2, err := strconv.Atoi(expression[2])
		if err != nil {
			fmt.Println(err)
			continue
		}
		result := opFunc(p1, p2)
		fmt.Println(result)
	}
}

```

## Anonymous Functions

```go
func main() {
	for i := 0; i < 5; i++ {
		func(j int) {
			fmt.Println("printing", j, "from inside of an anonymous function")
		}(i)
	}
}
```

Running the program gives the following output:

```txt
printing 0 from inside of an anonymous function
printing 1 from inside of an anonymous function
printing 2 from inside of an anonymous function
printing 3 from inside of an anonymous function
printing 4 from inside of an anonymous function
```

## Closures

Functions declared inside of functions are special; they are closures. This is a com‐
puter science word that means that functions declared inside of functions are able to
access and modify variables declared in the outer function.

All of this inner function and closure stuff might not seem all that interesting at first.
What benefit do you get from making mini-functions within a larger function? Why
does Go have this feature?

One thing that closures allow you to do is limit a function’s scope. If a function is only
going to be called from one other function, but it’s called multiple times, you can use
an inner function to “hide” the called function. This reduces the number of declara‐
tions at the package level, which can make it easier to find an unused name.

Closures really become interesting when they are passed to other functions or
returned from a function. They allow you to take the variables within your function
and use those values outside of your function.

### Passing Functions as Parameters

```go
type Person struct {
	FirstName string
	LastName string
	Age int
}
people := []Person{
	{"Pat", "Patterson", 37},
	{"Tracy", "Bobbert", 23},
	{"Fred", "Fredson", 18},
}
fmt.Println(people)
```

Next, we’ll sort our slice by last name and print out the results:

```go
// sort by last name
sort.Slice(people, func(i int, j int) bool {
	return people[i].LastName < people[j].LastName
})
fmt.Println(people)
```

The closure that’s passed to sort.Slice has two parameters, i and j, but within the
closure, we can refer to people so we can sort it by the LastName field. In computer
science terms, people is captured by the closure. Next we do the same, sorting by the
Age field:

```go
// sort by age
sort.Slice(people, func(i int, j int) bool {
	return people[i].Age < people[j].Age
})
fmt.Println(people)
```

Running this code gives the following output:

```txt
[{Pat Patterson 37} {Tracy Bobbert 23} {Fred Fredson 18}]
[{Tracy Bobbert 23} {Fred Fredson 18} {Pat Patterson 37}]
[{Fred Fredson 18} {Tracy Bobbert 23} {Pat Patterson 37}]
```

### Returning Functions from Functions

Not only can you use a closure to pass some function state to another function, you
can also return a closure from a function. Let’s show this off by writing a function
that returns a multiplier function. You can run this program on The Go Playground.
Here is our function that returns a closure:

```go
func makeMult(base int) func(int) int {
	return func(factor int) int {
		return base * factor
	}
}
```

And here is how we use it:

```go
func main() {
	twoBase := makeMult(2)
	threeBase := makeMult(3)
	for i := 0; i < 3; i++ {
		fmt.Println(twoBase(i), threeBase(i))
	}
}
```

Running this program gives the following output:

```txt
0 0
2 3
4 6
```

Now that you’ve seen closures in action, you might wonder how often they are used
by Go developers. It turns out that they are surprisingly useful. We saw how they are
used to sort slices. A closure is also used to efficiently search a sorted slice with
sort.Search. As for returning closures, we will see this pattern used when we build
middleware for a web server in “Middleware”. Go also uses closures to
implement resource cleanup, via the defer keyword.

## defer

> How can we achieve this in kotlin/java ?

The code within defer closures runs a"er the return statement. As I mentioned, you
can supply a function with input parameters to a defer. Just as defer doesn’t run
immediately, any variables passed into a deferred closure aren’t evaluated until the
closure runs.

Let’s look at a way to handle database transaction cleanup using named return values and defer:

```go
func DoSomeInserts(ctx context.Context, db *sql.DB, value1, value2 string)
(err error) {
	tx, err := db.BeginTx(ctx, nil)
	if err != nil {
		return err
	}
	defer func() {
		if err == nil {
			err = tx.Commit()
		}
		if err != nil {
			tx.Rollback()
		}
	}()
	_, err = tx.ExecContext(ctx, "INSERT INTO FOO (val) values $1", value1)
	if err != nil {
		return err
	}
	// use tx to do more database inserts here
	return nil
}
```

# Go is call by value

You might hear people say that Go is a call by value language and wonder what that
means. It means that when you supply a variable for a parameter to a function, Go
always makes a copy of the value of the variable. Let’s take a look.

```go
type person struct {
age int
name string
}
```

Next, we write a function that takes in an int, a string, and a person, and modifies
their values:

```go
func modifyFails(i int, s string, p person) {
	i = i * 2
	s = "Goodbye"
	p.name = "Bob"
}
```

We then call this function from main and see if the modifications stick:

```go
func main() {
	p := person{}
	i := 2
	s := "Hello"
	modifyFails(i, s, p)
	fmt.Println(i, s, p)
}
```

As the name of the function indicates, running this code shows that a function won’t
change the values of the parameters passed into it:
`2 Hello {0 }`

I included the person struct to show that this isn’t just true for primitive types. If you
have programming experience in Java, JavaScript, Python, or Ruby, you might find
the struct behaviour very strange. After all, those languages let you modify the fields
in an object when you pass an object as a parameter to a function. The reason for the
difference is something we will cover when we talk about pointers.

The behavior is a little different for maps and slices. Let’s see what happens when we
try to modify them within a function. You can run this code on The Go Playground.
We’re going to write a function to modify a map parameter and a function to modify
a slice parameter:

```go
func modMap(m map[int]string) {
	m[2] = "hello"
	m[3] = "goodbye"
	delete(m, 1)
}

func modSlice(s []int) {
	for k, v := range s {
		s[k] = v * 2
	}
	s = append(s, 10)
}
```

We then call these functions from main:

```go
func main() {
	m := map[int]string{
		1: "first",
		2: "second",
	}
	modMap(m)
	fmt.Println(m)
	s := []int{1, 2, 3}
	modSlice(s)
	fmt.Println(s)
}
```

When you run this code, you’ll see something interesting:

```txt
map[2:hello 3:goodbye]
[2 4 6]
```

For the map, it’s easy to explain what happens: any changes made to a map parameter
are reflected in the variable passed into the function. For a slice, it’s more compli‐
cated. You can modify any element in the slice, but you can’t lengthen the slice. This is
true for maps and slices that are passed directly into functions as well as map and
slice fields in structs.

This program leads to the question: why do maps and slices behave differently than
other types? It’s because maps and slices are both implemented with pointers. We’ll go
into more detail in the next chapter.

> Call by value is one reason why Go’s limited support for constants is only a minor
> handicap. Since variables are passed by value, you can be sure that calling a function
> doesn’t modify the variable whose value was passed in (unless the variable is a slice or
> map). In general, this is a good thing. It makes it easier to understand the flow of data
> through your program when functions don’t modify their input parameters and
> instead return newly computed values.

# Pointers

Go’s pointer syntax is partially borrowed from C and C++. Since Go has a garbage
collector, most of the pain of memory management is removed. Furthermore, some
of the tricks that you can do with pointers in C and C++, including pointer arithmetic,
are not allowed in Go.

The & is the address operator. It precedes a value type and returns the address of the
memory location where the value is stored:

```go
x := "hello"
pointerToX := &x
```

The \* is the indirection operator. It precedes a variable of pointer type and returns the
pointed-to value. This is called dereferencing:

```go
x := 10
pointerToX := &x
fmt.Println(pointerToX) // prints a memory address
fmt.Println(*pointerToX) // prints 10
z := 5 + *pointerToX
fmt.Println(z) // prints 15
```

Before dereferencing a pointer, you must make sure that the pointer is non-nil. Your
program will panic if you attempt to dereference a nil pointer:

```go
var x *int
fmt.Println(x == nil) // prints true
fmt.Println(*x) // panics
```

A pointer type is a type that represents a pointer. It is written with a \* before a type
name. A pointer type can be based on any type:

```go
x := 10
var pointerToX *int
pointerToX = &x
```

## Pointers Indicate Mutable Parameters

MIT’s course on Software Construction sums up the reasons why: “[I]mmutable types are safer from bugs, easier to understand, and more ready for change.
Mutability makes it harder to understand what your program is doing, and much
harder to enforce contracts.”

The lack of immutable declarations in Go might seem problematic, but the ability to
choose between value and pointer parameter types addresses the issue. As the Soft‐
ware Construction course materials go on to explain: “[U]sing mutable objects is just
fine if you are using them entirely locally within a method, and with only one reference to the object.” Rather than declare that some variables and parameters are
immutable, Go developers use pointers to indicate that a parameter is mutable.

Since Go is a call by value language, the values passed to functions are copies. For
nonpointer types like primitives, structs, and arrays, this means that the called function cannot modify the original. Since the called function has a copy of the original
data, the immutability of the original data is guaranteed.

## Pointer Passing Performance

Passing a value into a function takes longer as the data gets larger. It takes about a millisecond once the value gets to be around 10 megabytes of data.

The behaviour for returning a pointer versus returning a value is more interesting. For
data structures that are smaller than a megabyte, it is actually slower to return a
pointer type than a value type. For example, a 100-byte data structure takes around 10
nanoseconds to be returned, but a pointer to that data structure takes about 30 nano‐
seconds. Once your data structures are larger than a megabyte, the performance
advantage flips. It takes nearly 2 milliseconds to return 10 megabytes of data, but a
little more than half a millisecond to return a pointer to it.

You should be aware that these are very short times. For the vast majority of cases, the
difference between using a pointer and a value won’t affect your program’s performance. But if you are passing megabytes of data between functions, consider using a pointer even if the data is meant to be immutable.

## The Zero Value Versus No Value

The other common usage of pointers in Go is to indicate the difference between a
variable or field that’s been assigned the zero value and a variable or field that hasn’t
been assigned a value at all. If this distinction matters in your program, use a nil
pointer to represent an unassigned variable or struct field.

## The Difference Between Maps and Slices

As we saw in the previous chapter, any modifications made to a map that’s passed to a
function are reflected in the original variable that was passed in. Now that we know
about pointers, we can understand why: within the Go runtime, a map is implemented as a pointer to a struct. Passing a map to a function means that you are copying a pointer.
Because of this, you should avoid using maps for input parameters or return values,
especially on public APIs. On an API-design level, maps are a bad choice because
they say nothing about what values are contained within; there’s nothing that explicitly defines what keys are in the map, so the only way to know what they are is to trace
through the code. From the standpoint of immutability, maps are bad because the
only way to know what ended up in the map is to trace through all of the functions
that interact with it. This prevents your API from being self-documenting. If you are
used to dynamic languages, don’t use a map as a replacement for another language’s
lack of structure. Go is a strongly typed language; rather than passing a map around,
use a struct.

## Slice as Buffer

When reading data from an external resource (like a file or a network connection),
many languages use code like this:

```txt
r = open_resource()
while r.has_data() {
data_chunk = r.next_chunk()
process(data_chunk)
}
close(r)
```

The problem with this pattern is that every time we iterate through that while loop,
we allocate another data_chunk even though each one is only used once. This creates
lots of unnecessary memory allocations. Garbage-collected languages handle those
allocations for you automatically, but the work still needs to be done to clean them up
when you are done processing.

Even though Go is a garbage-collected language, writing idiomatic Go means avoid‐
ing unneeded allocations. Rather than returning a new allocation each time we read
from a data source, we create a slice of bytes once and use it as a buffer to read data
from the data source:

```go
file, err := os.Open(fileName)
if err != nil {
	return err
}
defer file.Close()
data := make([]byte, 100)
for {
	count, err := file.Read(data)
	if err != nil {
		return err
	}
	if count == 0 {
		return nil
	}
	process(data[:count])
}
```

Remember that we can’t change the length or capacity of a slice when we pass it to a
function, but we can change the contents up to the current length. In this code, we
create a buffer of 100 bytes and each time through the loop, we copy the next block of
bytes (up to 100) into the slice. We then pass the populated portion of the buffer to
process.

## Reducing the Garbage Collector’s Workload

Using buffers is just one example of how we reduce the work done by the garbage
collector. When programmers talk about “garbage” what they mean is “data that has
no more pointers pointing to it.” Once there are no more pointers pointing to some
data, the memory that this data takes up can be reused. If the memory isn’t recovered,
the program’s memory usage would continue to grow until the computer ran out of
RAM. The job of a garbage collector is to automatically detect unused memory and recover it so it can be reused. It is fantastic that Go has a garbage collector, because
decades of experience have shown that it is very difficult for people to properly manage memory manually. But just because we have a garbage collector doesn’t mean we should create lots of garbage.

The heap is the memory that’s managed by the garbage collector (or by hand in languages like C and C++). We’re not going to discuss garbage collector algorithm
implementation details, but they are much more complicated than simply moving a
stack pointer. Any data that’s stored on the heap is valid as long as it can be tracked
back to a pointer type variable on a stack. Once there are no more pointers pointing
to that data (or to data that points to that data), the data becomes garbage and it’s the
job of the garbage collector to clear it out.

> A common source of bugs in C programs is returning a pointer to a local variable. In C, this results in a pointer pointing to invalid memory. The Go compiler is smarter. When it sees that a pointer to a local variable is returned, the local variable’s value is stored on the heap.

You might be wondering: what’s so bad about storing things on the heap? There are
two problems related to performance. First is that the garbage collector takes time to
do its work. It isn’t trivial to keep track of all of the available chunks of free memory
on the heap or tracking which used blocks of memory still have valid pointers. This is
time that’s taken away from doing the processing that your program is written to do.
Many garbage collection algorithms have been written, and they can be placed into
two rough categories: those that are designed for higher throughput (find the most
garbage possible in a single scan) or lower latency (finish the garbage scan as quickly as possible). Jeff Dean, the genius behind many of Google’s engineering successes, co-wrote a paper in 2013 called [The Tail at Scale](https://research.google/pubs/the-tail-at-scale/). It argues that systems should be optimized for latency, to keep response times low. The garbage collector used by the Go runtime favors low latency. Each garbage collection cycle is designed to take less than 500 microseconds. However, if your Go program creates lots of garbage, then the
garbage collector won’t be able to find all of the garbage during a cycle, slowing down
the collector and increasing memory usage.

> This approach of writing software that’s aware of the hardware it’s running on is
> called mechanical sympathy. The term comes from the world of car racing, where the
> idea is that a driver who understands what the car is doing can best squeeze the last
> bits of performance out of it. In 2011, Martin Thompson began applying the term to
> software development. Following best practices in Go gives it to you automatically.

Compare Go’s approach to Java’s. In Java, local variables and parameters are stored in
the stack, just like Go. However, as we discussed earlier, objects in Java are implemented as pointers. That means for every object variable instance, only the pointer to it is allocated on the stack; the data within the object is allocated on the heap. Only primitive values (numbers, booleans, and chars) are stored entirely on the stack. This means that the garbage collector in Java has to do a great deal of work. It also means that things like Lists in Java are actually a pointer to an array of pointers. Even though it looks like a linear data structure, reading it actually involves bouncing through memory, which is highly inefficient.

> There are similar behaviours in Python, Ruby, and JavaScript. To work around all of this inefficiency, the Java Virtual Machine includes some very clever garbage collectors that do lots of work, some optimised for throughput, some for latency, and all with configuration settings to tune them for the best performance. The virtual machines for Python, Ruby, and JavaScript are less optimised and their performance suffers accordingly.

Now you can see why Go encourages you to use pointers sparingly. We reduce the
workload of the garbage collector by making sure that as much as possible is stored
on the stack. Slices of structs or primitive types have their data lined up sequentially
in memory for rapid access. And when the garbage collector does do work, it is optimized to return quickly rather than gather the most garbage. The key to making this
approach work is to simply create less garbage in the first place. While focusing on
optimising memory allocations can feel like premature optimisation, the idiomatic
approach in Go is also the most efficient.

If you want to learn more about heap versus stack allocation and escape analysis in Go, there are excellent blog posts that cover the topic, including ones by [Bill Kennedy of Arden Labs](https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-stacks-and-pointers.html) and [Achille Roussel and Rick Branson of Segment](https://segment.com/blog/allocation-efficiency-in-high-performance-go-services/).

# No Enums

Many programming languages have the concept of enumerations, where you can
specify that a type can only have a limited set of values. Go doesn’t have an enumera‐
tion type. Instead, it has iota, which lets you assign an increasing value to a set of
constants.

> The concept of iota comes from the programming language APL (which stood for “A Programming Language”). APL is famous for being so reliant on its own custom notation that it required computers with a special keyboard. For example, (~R∊R∘.×R)/R←1↓ιR is an APL program to find all the prime numbers up to the value of the variable R. It may seem ironic that a language as focused on readability as Go would borrow a concept from a language that is concise to a fault, but this is why you should learn many different programming languages: you can find inspiration everywhere.

When using iota, the best practice is to first define a type based on int that will rep‐
resent all of the valid values:

```go
type MailCategory int
```

Next, use a const block to define a set of values for your type:

```go
const (
	Uncategorized MailCategory = iota
	Personal
	Spam
	Social
	Advertisements
)
```

The first constant in the const block has the type specified and its value is set to iota.
Every subsequent line has neither the type nor a value assigned to it. When the Go
compiler sees this, it repeats the type and the assignment to all of the subsequent con‐
stants in the block, and increments the value of iota on each line. This means that it
assigns 0 to the first constant (Uncategorized), 1 to the second constant (Personal),
and so on. When a new const block is created, iota is set back to 0.

This is the best advice I’ve seen on iota:

> Don’t use iota for defining constants where its values are explicitly defined (elsewhere). For example, when implementing parts of a specification and the specification says which values are assigned to which constants, you should explicitly write the constant values. Use iota for “internal” purposes only. That is, where the constants are referred to by name rather than by value. That way you can optimally enjoy iota by inserting new constants at any moment in time / location in the list without the risk of breaking everything.

             —Danny van Heumen

# Use Embedding for Composition

The software engineering advice “Favor object composition over class inheritance”
dates back to at least the 1994 book Design Patterns by Gamma, Helm, Johnson, and
Vlissides (Addison-Wesley), better known as the Gang of Four book. While Go
doesn’t have inheritance, it encourages code reuse via built-in support for composi‐
tion and promotion:

```go
type Employee struct {
	Name string
	ID string
}

func (e Employee) Description() string {
	return fmt.Sprintf("%s (%s)", e.Name, e.ID)
}

type Manager struct {
	Employee
	Reports []Employee
}

func (m Manager) FindNewEmployees() []Employee {
	// do business logic
}
```

Note that Manager contains a field of type Employee, but no name is assigned to that
field. This makes Employee an embedded feild. Any fields or methods declared on an
embedded field are promoted to the containing struct and can be invoked directly on
it. That makes the following code valid:

```go
m := Manager{
	Employee: Employee{
		Name: "Bob Bobson",
		ID: "12345",
	},
	Reports: []Employee{},
}
fmt.Println(m.ID) // prints 12345
fmt.Println(m.Description()) // prints Bob Bobson (12345)
```

You can embed any type within a struct, not just another struct. This promotes the methods on the embedded type to the containing struct.

## Embedding Is Not Inheritance

Built-in embedding support is rare in programming languages (I’m not aware of
another popular language that supports it). Many developers who are familiar with
inheritance (which is available in many languages) try to understand embedding by
treating it as inheritance. That way lies tears.

# Interfaces

While Go’s concurrency model gets all of the publicity, the real star of Go’s design is its implicit interfaces, the only abstract type in Go.

Let’s see what makes them so great.

Here’s the definition of the Stringer interface in the fmt package:

```go
type Stringer interface {
	String() string
}
```

In an interface declaration, an interface literal appears after the name of the interface
type. It lists the methods that must be implemented by a concrete type to meet the
interface. The methods defined by an interface are called the method set of the
interface.

Like other types, interfaces can be declared in any block.

Interfaces are usually named with “er” endings. We’ve already seen fmt.Stringer, but
there are many more, including io.Reader, io.Closer, io.ReadCloser, json.Mar
shaler, and http.Handler.

## Interfaces Are Type-Safe Duck Typing

This implicit behavior makes interfaces the most interesting thing about types in Go,
because they enable both type-safety and decoupling, bridging the functionality in
both static and dynamic languages.

To understand why, let’s talk about why languages have interfaces. Earlier we mentioned that Design Patterns taught developers to favor composition over inheritance. Another piece of advice from the book is “Program to an interface, not an implementation.” Doing so allows you to depend on behavior, not on implementation, allowing you to swap implementations as needed. This allows your code to evolve over time, as requirements inevitably change.

Dynamically typed languages like Python, Ruby, and JavaScript don’t have interfaces. Instead, those developers use “duck typing,” which is based on the expression `“If it walks like a duck and quacks like a duck, it’s a duck.”` The concept is that you can pass an instance of a type as a parameter to a function as long as the function can find a method to invoke that it expects:

![[Pasted image 20240623110153.png]]

Go’s developers decided that both groups are right. If your application is going to
grow and change over time, you need flexibility to change implementation. However,
in order for people to understand what your code is doing (as new people work on
the same code over time), you also need to specify what the code depends on. That’s
where implicit interfaces come in. Go code is a blend of the previous two styles:

## Accept Interfaces, Return Structs

You’ll often hear experienced Go developers say that your code should “Accept interfaces, return structs.” What this means is that the business logic invoked by your
functions should be invoked via interfaces, but the output of your functions should
be a concrete type. We’ve already covered why functions should accept interfaces:
they make your code more flexible and explicitly declare exactly what functionality is
being used.

For more read Part 2

# Miscellaneous

## Hash map in a gist

In computer science, a map is a data structure that associates (or maps) one value to
another. Maps can be implemented several ways, each with their own trade-offs. The
map that’s built-in to Go is a hash map. In case you aren’t familiar with the concept,
here is a really quick overview.

A hash map does fast lookups of values based on a key. Internally, it’s implemented as
an array. When you insert a key and value, the key is turned into a number using a
hash algorithm. These numbers are not unique for each key. The hash algorithm can
turn different keys into the same number. That number is then used as an index into
the array. Each element in that array is called a bucket. The key-value pair is then
stored in the bucket. If there is already an identical key in the bucket, the previous
value is replaced with the new value.

Each bucket is also an array; it can hold more than one value. When two keys map to
the same bucket, that’s called a collision, and the keys and values for both are stored in
the bucket.

A read from a hash map works in the same way. You take the key, run the hash algo‐
rithm to turn it into a number, find the associated bucket, and then iterate over all the
keys in the bucket to see if one of them is equal to the supplied key. If one is found,
the value is returned.

You don’t want to have too many collisions, because the more collisions, the slower
the hash map gets, as you have to iterate over all the keys that mapped to the same
bucket to find the one that you want. Clever hash algorithms are designed to keep
collisions to a minimum. If enough elements are added, hash maps resize to rebalance
the buckets and allow more entries.

Hash maps are really useful, but building your own is hard to get right. If you’d like to
learn more about how Go does it, watch this talk from GopherCon 2016, [Inside the Map Implementation](https://www.youtube.com/watch?v=Tl7mi9QmLns).

Go doesn’t require (or even allow) you to define your own hash algorithm or equality
definition. Instead, the Go runtime that’s compiled into every Go program has code
that implements hash algorithms for all types that are allowed to be keys.

## UTF-8

UTF-8 is the most commonly used encoding for Unicode. Unicode uses four bytes
(32 bits) to represent each code point, the technical name for each character and
modifier. Given this, the simplest way to represent Unicode code points is to store
four bytes for each code point. This is called UTF-32. It is mostly unused because it
wastes so much space. Due to Unicode implementation details, 11 of the 32 bits are
always zero. Another common encoding is UTF-16, which uses one or two 16-bit (2-
byte) sequences to represent each code point. This is also wasteful; much of the con‐
tent in the world is written using code points that fit into a single byte. And that’s
where UTF-8 comes in.

UTF-8 is very clever. It lets you use a single byte to represent the Unicode characters
whose values are below 128 (which includes all of the letters, numbers, and punctua‐
tion commonly used in English), but expands to a maximum of four bytes to repre‐
sent Unicode code points with larger values. The result is that the worst case for
UTF-8 is the same as using UTF-32. UTF-8 has some other nice properties. Unlike
UTF-32 and UTF-16, you don’t have to worry about little-endian versus big-endian. It
also allows you to look at any byte in a sequence and tell if you are at the start of a
UTF-8 sequence, or somewhere in the middle. That means you can’t accidentally read
a character incorrectly.

The only downside is that you cannot randomly access a string encoded with UTF-8.
While you can detect if you are in the middle of a character, you can’t tell how many
characters in you are. You need to start at the beginning of the string and count. Go
doesn’t require a string to be written in UTF-8, but it strongly encourages it. We’ll see
how to work with UTF-8 strings in upcoming chapters.
Fun fact: UTF-8 was invented in 1992 by Ken Thompson and Rob Pike, two of the
creators of Go.

# Runtime

The Go runtime is compiled into every Go binary. This is different from languages
that use a virtual machine, which must be installed separately to allow programs writ‐
ten in those languages to function. Including the runtime in the binary makes it easier
to distribute Go programs and avoids worries about compatibility issues between the
runtime and the program.

# More

For more read Part 2
