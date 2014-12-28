- title : Java 8 Streams through the lens of OCaml/F#
- description : Java 8 streams via the lens of OCaml/F#
- author : Nick Palladinos
- theme : night
- transition : default

***


### Java 8 Streams

through the lens of OCaml/F#

***

### Clash of the Lambdas


![Clash of the Lambdas](images/cotl.jpg)
ICOOOLPS'14
[arxiv](http://arxiv.org/abs/1406.6631)


---

### Performance Benchmarks

#### Sum (windows)

![Sum](images/sum_win.jpg)

---

#### Sum (linux)

![Sum](images/sum_lin.jpg)

---

#### Sum of squares (windows)

![Sum Squares](images/sumOfSquares_win.jpg)

---

#### Sum of squares (linux)

![Sum Squares](images/sumOfSquares_lin.jpg)

---

#### Sum of even squares (windows)

![Sum Even Squares](images/sumOfSquaresEven_win.jpg)

---

#### Sum of even squares (linux)

![Sum Even Squares](images/sumOfSquaresEven_lin.jpg)

---

#### Cartesian product (windows)

![Cartesian product](images/cartesian_win.jpg)

---

#### Cartesian product (linux)

![Cartesian product](images/cartesian_lin.jpg)

***

#### Java 8 very fast

---


### What makes Java 8 faster?

***

### Streams!

***

### Typical Pipeline Pattern

    source |> inter |> inter |> inter |> terminal

- inter : intermediate operations, e.g. map, filter
- terminal : produces result or side-effects, e.g. reduce, iter

***

### OCaml Streams example
    // let (|>) a f = f a
    let data = [| 1..10000000 |] 
    data
    |> Stream.ofArray
    |> Stream.filter (fun i -> i % 2 = 0) 
    |> Stream.map (fun i -> i * i) 
    |> Stream.sum 
    
[OCaml Streams](https://ocaml.org/learn/tutorials/streams.html)

***

### Stream is pulling

     type Stream<'T> = unit -> (unit -> 'T)

     val ofArray : 'T[] -> Stream<'T>
     let ofArray values =  
		fun () -> 
			let index = ref -1
			(fun () ->
				incr index
				if !index < Array.length values then 
					f values.[!index])
				else
					raise StreamEnd)


---

### Simple functions

     val map : ('T -> 'R) -> Stream<'T> -> Stream<'R> 
	 let map f stream = 
		fun () ->
			let next = stream ()
			fun () -> f (next ())

---			
			
### Filter is recursive!
			
			
	 val filter : ('T -> bool) -> Stream<'T> -> Stream<'T> 
	 let filter p stream = 
		let rec loop v next =
			if p v then v else loop (next ()) next
		fun () -> 
			let next = stream ()
			fun () -> loop (next ()) next

---

### Simple functions

	 val iter : ('T -> unit) -> Stream<'T> -> unit
	 let iter f stream
		let rec loop v next =
			f v; loop (next ()) next
		let next = stream ()
		try
			loop (next ()) next
		with StreamEnd -> ()


	val sum : Stream<'T> -> int
	let sum stream = 
		let sum = ref 0
		iter (fun v -> sum := !sum + v) stream
		!sum
		

---

### Simple functions
			
	 val zip : Stream<'T> -> Stream<'R> -> Stream<'T * 'R> 
	 let zip stream stream' = 
		fun () ->
			let next, next' = stream (), stream' ()
			fun () -> (next (), next' ())
---

### flatMap is tricky!

     val flatMap : ('T -> Stream<'R>) -> Stream<'T> -> Stream<'R> 
	 let flatMap f stream = 
		let current = ref None
		fun () ->
			let next = stream ()
			fun () -> 
				let rec loop () =
					match !current with
					| None -> 
						current := Some (f (next ()))
						loop ()
					| Some next' -> 
						try 
							next' () 
						with StreamEnd -> 
							current := f (next ())
							loop ()
				loop ()

***

### Java 8 Streams are pushing

    Stream.OfArray(data)
		  .filter(i -> i % 2 == 0) 
    	  .map(i -> i * i) 
    	  .sum();	

The source is pushing data down the pipeline.

***

### How does it work?

---

### Starting from foreach

    val iter : ('T -> unit) -> 'T[] -> unit
	let iter f values = 
		for value in values do
			f value

---

### Flip the args

    'T[] -> ('T -> unit) -> unit

---

### Stream!

    type Stream<'T> = ('T -> unit) -> unit
	
    val ofArray : 'T[] -> Stream<'T>
	let ofArray values k = 
		for value in values do
			k value
---

### Continuation passing style!

***


### Let's make us some (simple) Streams!

***

### Simple functions

    type Stream = ('T -> unit) -> unit
	
	val map : ('T -> 'R) -> Stream<'T> -> Stream<'R> 
	let map f stream = 
		fun k -> stream (fun v -> k (f v))

---

### Simple functions

		
	val filter : ('T -> bool) -> Stream<'T> -> Stream<'T> 
	let filter f stream = 
		fun k -> stream (fun v -> if f v then k v else ()) 

---

### Simple functions
		
	val sum : Stream<'T> -> int
	let sum stream = 
		let sum = ref 0
		stream (fun v -> sum := !sum + v)
		!sum
		
---

### flatMap is easy

	val flatMap : ('T -> Stream<'R>) -> Stream<'T> -> Stream<'R> 
	let flatMap f stream = 
		fun k -> stream (fun v -> let stream' = f v in stream' k)

***


### What about zip?

	Stream.zip : Stream<'T> -> Stream<'S> -> Stream<'T * 'S>

Zip needs to synchronise the flow of values.

Zip needs to pull!

---
	type Stream<'T>     = ('T -> unit) -> unit
	type StreamPull<'T> = unit -> (unit -> 'T)
	
    val toPull : Stream<'T> -> StreamPull<'T>
	let toPull stream = ???
	
	val zip : Stream<'T> -> Stream<'R> -> Stream<'T * 'R>
	let zip stream stream' = 
		let pullStream, pullStream' = toPull stream, toPull stream'
		let next, next' = pullStream (), pullStream' () 
		fun k -> k (next (), next' ())

***

### Streams can push and pull

	/// Provides functions for iteration
	type Iterable = {
		Bulk : unit -> unit 
		TryAdvance : unit -> bool 
	}

	/// Represents a Stream of values.
	type Stream<'T> = Stream of ('T -> unit) -> Iterable

---

### ofArray

    val ofArray : 'T[] -> Stream<'T>
	let ofArray values = 
		fun k ->
			let bulk () =
				for value in values do
					k value
					
			let index = ref -1
			let tryAdvance () =
				incr index;
				if !index < Array.length values then 
					(k values.[!index])
					true
				else
					false
			{ Builk = bulk; TryAdvance = tryAdvance  }
---

### toPull

    val toPull : Stream<'T> -> StreamPull<'T>
	let toPull stream = 
		fun () ->
			let current = ref None
			let { Bulk = _; TryAdvance = next } = stream (fun v -> current := v)
			fun () ->
				let rec loop () =
					if next () then
						match !current with
						| Some v ->
							current := None
							v
						| None -> loop ()
					else raise StreamEnd
				loop ()
			

---			

### toPull - Revised

    val toPull : Stream<'T> -> StreamPull<'T>
	let toPull stream = 
		fun () ->
			let buffer = new ResizeArray<'T>()
			let { Bulk = _; TryAdvance = next } = stream (fun v -> buffer.Add(v))
			let index = ref -1
			fun () ->
				let rec loop () =
					incr index
					if !index < buffer.Count then
						buffer.[!index]
					else
						buffer.Clear()
						index := -1
						if next () then
							loop ()
						else raise StreamEnd
				loop ()
	
			
---

### Gotcha!

     let pull = 
		[|1..10|]
		|> Stream.ofArray
		|> Stream.flatMap (fun _ -> Stream.infinite)
		|> Stream.toPull
		
	let next = pull () 
	next () // OutOfMemory Exception

***

### The Streams library

Implements a rich set of operations

***

### More examples

***

### Parallel Streams

	let data = [| 1..10000000 |]
	data
	|> ParStream.ofArray
	|> ParStream.filter (fun x -> x % 2 = 0)
	|> ParStream.map (fun x -> x * x)
	|> ParStream.sum

---

### ParStream 
	type ParStream<'T> = (unit -> ('T -> unit)) -> unit
	
	val ofArray : 'T[] -> ParStream<'T>
	let ofArray values = 
		fun thunk -> 
			let forks = 
				values 
				|> partitions 
				|> Array.map (fun p -> (p, thunk ())) 
				|> Array.map (fun (p, k) -> fork p k)
			join forks	
---
			
### ParStream 
	type ParStream<'T> = (unit -> ('T -> unit)) -> unit
	
	val map : ('T -> 'R) -> ParStream<'T> -> ParStream<'R> 
	let map f stream = 
		fun thunk ->
			stream (fun () ->
						let k = thunk ()
						(fun v -> k (f v)))
			
---

### ParStream 
	type ParStream<'T> = (unit -> ('T -> unit)) -> unit
	
	val sum : ParStream<'T> -> int 
	let sum stream = 
		let array = new ResizeArray<int ref>()
		stream (fun () ->
					let sum = ref 0
					array.Add(sum)
					(fun v -> sum := sum + v)
				)
		array |> Array.map (fun sum -> !sum) |> Array.sum
			
***

### Some Benchmarks

##### i7 8 x 3.7 Ghz (4 physical), 6 GB RAM

---

#### Sum of Squares

![Sum of Squares](images/sum_of_square.png)

----

#### Sum of Even Squares

![Sum of Even Squares](images/sum_of_square_even.png)

---

#### Cartesian Product

![Cartesian](images/cartesian_product.png)

---

#### Parallel Sum of Squares

![Parallel Sum](images/parallel_sum_of_square.png)

---

#### Parallel Sum of Squares Even

![Parallel Sum](images/parallel_sum_of_square_even.png)

---

#### Parallel Cartesian

![Parallel Cartesian](images/parallel_cartesian_product.png)

***

### Cloud Streams!

Example: a word count

	cfiles
	|> CloudStream.ofCloudFiles CloudFile.ReadLines
	|> CloudStream.collect Stream.ofSeq 
	|> CloudStream.collect (fun line -> splitWords line |> Stream.ofArray)
	|> CloudStream.filter wordFilter
	|> CloudStream.countBy id
	|> CloudStream.sortBy (fun (_,c) -> -c) count
	|> CloudStream.toCloudArray


https://github.com/mbraceproject/MBrace.Streams

***

### Streams are lightweight and powerful

#### In sequential, parallel and distributed flavors

***

### Beauty and the Beast

Write beautiful functional code with the performance of imperative code.

***

### Stream fusion in Haskell

![Stream fusion](images/Stream_Fusion.jpg)
http://code.haskell.org/~dons/papers/icfp088-coutts.pdf

---


### Stream fusion in Haskell

![Haskell beats C](images/Haskell_beats_C.jpg)
http://research.microsoft.com/en-us/um/people/simonpj/papers/ndp/haskell-beats-C.pdf

---


### Stream fusion in Haskell

![HERMIT](images/HERMIT.jpg)
http://www.ittc.ku.edu/~afarmer/concatmap-pepm14.pdf


***

### Stream operations are non-recursive

In principal, can be always fused (in-lined).

Not always done by F#/OCaml compilers.

***

### Experiments with MLton

by @biboudis

https://github.com/biboudis/sml-streams

MLton appears to always be fusing.

***

#### GitHub 

https://github.com/nessos/Streams

***

#### NuGet

https://www.nuget.org/packages/Streams/

***

#### Slides and samples

https://github.com/palladin/StreamsPLSeminar

***

### Thank you!

***

## Questions?

