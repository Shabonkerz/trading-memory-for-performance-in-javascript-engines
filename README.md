# Trading memory for performance in JavaScript engines


In the early days of the web, when MySpace was the unrelenting eyesore that it was, and Internet Explorer had a 96% market share, JavaScript performance was a concern, but not nearly a priority. Most JavaScript interpreters at the time had a simple compilation scheme that translated JavaScript code into bytecode, and, since then, numerous JIT(Just-In-Time) tiers have been added to milk every last drop of performance. To assist, and have said tiers achieve optimal performance, certain techniques must be employed, namely inline caching, as well as inline expansion, and hidden classes. We'll examine those techniques in more detail.

## Hidden Classes (or 'Shapes')

Almost all browsers make use of Self's technique of converting dynamic structures to static ones for performance gains. Google calls them 'hidden classes', or 'maps' in V8, while Mozilla uses the term 'shapes' within SpiderMonkey, but they're the same concept, in that they substitute an instance of a static structure(ie. a class instance) for a dynamic one(ie. a dictionary/hashtable). Dynamic structures require lookups for accessing properties, usually by means of a hash function that produces the index of where to find the relevant data. With class instances, however, no lookup is required, as indexes, or offsets, for properties are readily available. The nominal case provides significant a significant performance boost, but it's not without pitfalls.

Generally, JavaScript objects are comprised of a limited number of properties when the developer adheres to best practices and limits the responsibilities of the objects in question. V8's hidden classes approach is excellent for such cases, but if an object has an absurd number of properties, there is a considerable amount of memory bloat, albeit non-fatal.

To illustrate, let's examine the nominal case first. Let's use an application that has the need for encapsulating `Person` into a object, and each `Person` has the properties `name`, `age`, `gender`, and `height`, and said properties are unconditionally initialized within the constructor in a consistent order.

```
function Person(name, age, gender, height) {

	// Transitions from C0 to C1.
	this.name = name;

	// Transitions from C1 to C2.
	this.age = age;

	// Transitions from C2 to C3.
	this.gender = gender;

	// Transitions from C3 to C4.
	this.height = height;
}
```

Upon the constructor being called, V8 creates a hidden class `C0` with no properties, and then sees a `name` property, after which it creates a `C1` class with a name property and records the transition from `C0` to `C1` and associates it with the empty set of properties and the set containing one property, `name`. Similarly, when V8 sees the `age` property, it'll create a `C2` class that has a name and age property, and records the transition from `C1` to `C2`, and the same process continues as properties are added to `Person`. In the end, we are left with `C0`, `C1`, `C2`, `C3`, and `C4`, but every instance of `Person` will correspond to an instance of `C4` unless transitioning to `C4`(ie. the constructor hasn't yet completed). We end up with 4 types(`C0`, `C1`, `C2`, `C3`) that are only *transitory*. If, however, an object contained N properties, there would become N transitory hidden classes, all of which are stored in memory. Furthermore, if the ordering of the initialization of the properties were to deviate, **additional** hidden classes would be created and memory would further bloat.

The naive alternative is for each object to have a hashtable containing a key-value pair for each property, and such requires an underlying array that is equal or greater(in most cases) in size than the total number of keys in said hashtable. This compared to hidden classes approach has a worse space complexity in the average case, but not the worst case. In the worst case<sup>[1](#worstcase)</sup>, where properties are randomized in order and conditionally set, property hashtables see no difference from the average case, but the hidden classes approach see much more memory allocation. For example, if we were to conditionally call a different private initialization method within the `Person` constructor, where the `name` property is omitted in the `Person` object, V8 would would then create 3 additional hidden classes. Using the same example, property hashtables would store one less key.

```
Person.prototype.conditionalInitialize = function (age, gender, height) {
	// Transitions from C0 to C5.
	this.age = age;

	// Transitions from C5 to C6.
	this.gender = gender;

	// Transitions from C6 to C7.
	this.height = height;
};

function Person(name, age, gender, height) {

	if (!name) {
		this.conditionalInitialize(age, gender, height);
	}
	else {
		// Transitions from C0 to C1.
		this.name = name;

		// Transitions from C1 to C2.
		this.age = age;

		// Transitions from C2 to C3.
		this.gender = gender;

		// Transitions from C3 to C4.
		this.height = height;
	}
}

```

It's a no-brainer why modern JavaScript engines have hidden classes in place. The time complexity for a property access or write ends up being constant, and a better constant than a hashtable lookup since the index is already available. The space complexity is reasonable except in cases where the language usage is clearly not.




<a name="worstcase">1</a>: Actually, the worst case would also include *randomized* property names, but then it is an entirely different object. If such a constructor existed, however, the created hidden classes would not be reused.

## Inline caching

In addition to quick property accesses, most browsers have a need for swift method calls, regardless of the type upon which the method is being called. With the example code below, you can see that the interpreter must locate a method entitled `eat` that belongs to `dave`.

```
var hamburger = new Hamburger();
var dave = new Person('Dave');
var didEat = dave.eat(hamburger);
```
First, it must look at the object's own properties, and if no hit, it examines the `Person` prototype. This ends up being rather expensive as it involves  lookups in potentially multiple hashtables on each call. To counteract this expense, V8 and SpiderMonkey both make use of inline caching, where the result of the method lookup is cached within the call site for instant access.

Inline caching isn't without pitfalls itself. Inline caching must be place guard conditions to check the type of the object prior to using the cached function call. If the guard conditions fail, it experiences a cache miss, and has to resort to using lookups. In the case of polymorphic objects, where the type can vary upon each call, inline caching ends up with a significant number of cache misses, which lead to lookups and cache overwrites. As a worst case example, suppose we had 2 types of objects, `Cat` and `Dog`, and both had a method named `eat` and the code below existed:

```
var food = new Bird();
_.forEach([new Dog(), new Cat(), new Dog(), new Cat()], (animal) => {
	animal.eat(food);
});
```

On the first iteration, the `eat` method for `Dog` is retrieved and cached, but on the next iteration it is a cache miss. Same process occurs for the `eat` method for `Cat`, but again the next iteration is a cache miss. None of the iterations end up using the cached method, but the type checks still execute, and thus the naive inline caching does more harm than good.

The potentially obvious answer to this problem is to combine the type checks and the caches for `Dog` and `Cat`, and extend the cache as additional types are encountered. This is considered polymorphic inline caching, or PIC, and most JavaScript engines utilize this approach.


## Inline expansion

In addition to inline caching, modern JavaScript engines will utilize inline expansion, where short methods are inlined at the calling site. For example:

```
function add(x, y) {
	return x + y;
}

function add10(x) {
	return add(x, 10);
}
```

would become:

```
function add10(x) {
	return x + 10;
}
```

As a result, one less stack frame is needed, and the additional overhead for calling `add(x,y)` at the target site is removed. The branch or jump instruction is also removed as a result, and thusly CPU pipelining can be more fully utilized. Inline expansion does end up duplicating the body of the function being inlined, meaning additional memory requirements, but it is well worth the performance boost.

## Additional points

### Process-per-tab

Most browsers(Chrome and Firefox, at least) that use these engines will launch a process for each tab created, further enhancing reliability by preventing the errors caused in one to affect another tab. This also means that each process has its own JavaScript interpreter assigned to a single thread, and can make use of concurrency and parallelism with support and management from the OS. In cases where synchronous AJAX requests(generally a horrible idea) were made in one tab, this means it won't block the program execution in other tabs.

### Garbage Collection

Garbage collection is actually trading performance for memory, since its objective is to reclaim memory on a periodic basis, but for the period in which it is inactive, the reverse can be argued to be true. In refusal to be too philosophical, I'll consider it a draw and regard GC as managing memory and performance.
