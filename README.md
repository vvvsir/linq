# linq
A powerful language integrated query (LINQ) library for Go.

* Written in vanilla Go, no dependencies!
* Complete lazy evaluation with iterator pattern
* Safe for concurrent use
* Supports generic functions to make your code cleaner and free of type assertions
* Supports arrays, slices, maps, strings, channels and custom collections

## Quickstart

Usage is as easy as chaining methods like:

`From(slice)` `.Where(predicate)` `.Select(selector)` `.Union(data)`

**Example 1: Find all owners of cars manufactured after 2015**

```go
type Car struct {
    year int
    owner, model string
}

...


var owners []string

From(cars).Where(func(c interface{}) bool {
	return c.(Car).year >= 2015
}).Select(func(c interface{}) interface{} {
	return c.(Car).owner
}).ToSlice(&owners)
```

Or, you can use generic functions, like `WhereT` and `SelectT` to simplify your code
(at a performance penalty):

```go
var owners []string

From(cars).WhereT(func(c Car) bool {
	return c.year >= 2015
}).SelectT(func(c Car) string {
	return c.owner
}).ToSlice(&owners)
```

**Example 2: Find the author who has written the most books**

```go
type Book struct {
	id      int
	title   string
	authors []string
}

author := From(books).SelectMany( // make a flat array of authors
	func(book interface{}) Query {
		return From(book.(Book).authors)
	}).GroupBy( // group by author
	func(author interface{}) interface{} {
		return author // author as key
	}, func(author interface{}) interface{} {
		return author // author as value
	}).OrderByDescending( // sort groups by its length
	func(group interface{}) interface{} {
		return len(group.(Group).Group)
	}).Select( // get authors out of groups
	func(group interface{}) interface{} {
		return group.(Group).Key
	}).First() // take the first author
```

**Example 3: Implement a custom method that leaves only values greater than the specified threshold**

```go
type MyQuery Query

func (q MyQuery) GreaterThan(threshold int) Query {
	return Query{
		Iterate: func() Iterator {
			next := q.Iterate()

			return func() (item interface{}, ok bool) {
				for item, ok = next(); ok; item, ok = next() {
					if item.(int) > threshold {
						return
					}
				}

				return
			}
		},
	}
}

result := MyQuery(Range(1,10)).GreaterThan(5).Results()
```

## Generic Functions

Although Go doesn't implement generics, with some reflection tricks, you can use go-linq without
typing `interface{}`s and type assertions. This will introduce a performance penalty (5x-10x slower)
but will yield in a cleaner and more readable code.

Methods with `T` suffix (such as `WhereT`) accept functions with generic types. So instead of

    .Select(func(v interface{}) interface{} {...})

you can type:

    .SelectT(func(v YourType) YourOtherType {...})

This will make your code free of `interface{}` and type assertions.

**Example 4: "MapReduce" in a slice of string sentences to list the top 5 most used words using generic functions**

```go
var results []string

From(sentences).
	// split sentences to words
	SelectManyT(func(sentence string) Query {
		return From(strings.Split(sentence, " "))
	}).
	// group the words
	GroupByT(
		func(word string) string { return word },
		func(word string) string { return word },
	).
	// order by count
	OrderByDescendingT(func(wordGroup Group) int {
		return len(wordGroup.Group)
	}).
	// order by the word
	ThenByT(func(wordGroup Group) string {
		return wordGroup.Key.(string)
	}).
	Take(5).  // take the top 5
	// project the words using the index as rank
	SelectIndexedT(func(index int, wordGroup Group) string {
		return fmt.Sprintf("Rank: #%d, Word: %s, Counts: %d", index+1, wordGroup.Key, len(wordGroup.Group))
	}).
	ToSlice(&results)
```
