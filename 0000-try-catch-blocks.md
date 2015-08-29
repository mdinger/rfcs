- Feature Name: try_catch_blocks
- Start Date: 2015-08-29
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Most modern languages implement an exception system, which is a practical mechanism to the developer, but at the same time very likely to compromise safety, and is not acceptable in Rust's standards.

The main concept here would be to reproduce try/catch blocks constructs without any exception approach, based on the current `std::Result<T, E>` paradigm. It would provide lots of benefits in practice, and could be done with some compiler magic.


# Motivation

While Rust's was designed with a non-negotiable safety requirements,  the absence of an exception system has resulted with a massive use of `std::Result<T, E>`, returned by any function introducing a possibility to fail.

The introduction of the try! macro did provide in some case improvements, but its requirements don't make it usable in most contexts. (in the context of different Error types)

This leads in practice to a massive amount of code, requires a behavior be implemented for each possible function call returning a Result, and greatly increases verbosity of the code produced.


# Detailed design

A try/catch block in Rust would basically adopt the traditional syntax, with a try, catch and throw keywords.

Here is a simple use case:

```rust

fn someFunction() -> Result<R, ErrorA>
{
   try
   {  //try scope
      
      let some_result = some_operation();
      
      //return the result
      some_result 
   }
   catch(err : &ErrorB)
   {
       //catch scope
       throw ErrorA::new("something went wrong")
   }
}

```
In this example, we consider `some_operation` to have the signature `some_operation() -> Result<R, ErrorB>`.

Hence, the try block acquires a special characteristic; by eliding any `Result<R, E>` and returning the `R` instance directly, it's responsible for determining if the operation failed or not via the standard `Result<T,E>` construct.

Upon operation failure, the try block will drop the current `try` scope safely (same behavior as `return` statement) and initiate a call to `catch(err : &ErrorB)`.

The `catch` block would internally be a closure with an `&ErrorB` as input, returning a `Result<R, ErrorA>` just like `someFunction`.
Inside the catch block, the implementor can either resolve the error by returning a `R` instance, or return a `ErrorA` using the `throw` keyword.

Hence the compiler should translate the code above to something like:

```rust

fn someFunction() -> Result<R, ErrorA>
{
   let catchErrorB = |err : &ErrorB| -> Result<R, ErrorA> {
   {
      //throw is translated to Err()
      Err(ErrorA::new("something went wrong")) 
   };
   
   let tryBlock = |_| -> Result<R, ErrorA>
   {
      //try scope
      match some_operation() {
           
            Ok(some_result) => 
            {
               //call suceeded!
               //continue to the next statements...
               
                  //return the result
                  some_result
               
            },
            Err(errorB) => catchErrorB(errorB) //this returns error A
        }
      
    
   };

   tryBlock(0)

}

```

The example above shows the compiler has to be able **to split the try block into a chain of statements, where each "split" occurs when a call to a function returning a Result<,> is made.**

- If the result is Ok for the current node, the next "statement node"  is executed.

- If an error occurred the catch block handling the corresponding error type is called.

The compiler will refuse to compile if a required Catch block implementing a specific error type is missing. 
Hence a try block with multiple calls that involve ErrorB, ErrorC, and ErrorD would require:
```
try
{
//...
}catch(err : &ErrorB)
{

}catch(err : &ErrorC)
{

}catch(err : &ErrorD)
{

}
```

#Benefits

- With such a feature, Rust code would greatly gain in readability and conciseness.
- In the try block, Eliding any Result<T,E> into T would allow calls to be chained recursively, without dealing with potential error in the try body.
- Most of the time the same behavior is used multiple times for multiple potential errors.

# Drawbacks

Such feature wouldn't have any impact on existing code in my knowledge.

# Alternatives

I've considered a single catch block, that would require a new trait, to be implemented by all errors, however this would involve a major breaking change.

# Unresolved questions

Implementing a stacktrace would be useful. (if there isn't any yet)
