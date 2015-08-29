- Feature Name: try_catch_blocks_in_rust
- Start Date: 2015-08-29
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Most modern languages implement an exception system, which is a practical mechanism to the developper, but at the same time very likely to compromise safety, and is not acceptable in Rust's standards.

However reproducing try/catch blocks constructs could be feasible, based on the current `std::Result<T, E>` paradigm. It would provide lots of benefits in practice, and could be done with some compiler magic.


# Motivation

While Rust's was designed with a non-negociable safety requirements,  the absence of an exception system has resulted with a massive use of `std::Result<T, E>`, returned by any function introducing a possibility to fail.

The introduction of the try! macro did provide in some case improvements, but its requirements don't make it usable in most contexts. (Error of different types)

This leads in practice to an massive amount of code, requiring to implement a behavior for each possible for each function call returning a Result, and greatly increases verbosity of the code produced.


# Detailed design

A try/catch block in Rust would basically adopt the traditionnal syntax, with a try, catch and throw keywords.

Here is a simple use case:

```

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
       throw ErrorA::new("something went wrong")
   }
}

```
In this example, we consider `some_operation` to have the signature `some_operation() -> Result<R, ErrorB>`.

Hence, the try blocks acquire a special caracteristic, by eliding any `Result<R, E>` and returning the `R` instance directly if the call succeeded, it's responsible to check weither the operation failed or not via the standard `Result<T,E>` construct.

In case the operation did failled, the try block would drop the current `try` scope safely (same behavior as `return` statement) and initiates a call to `catch(err : &ErrorB)`.

The `catch` block would be internally a closure with a `&ErrorB` as input that returns a `Result<R, ErrorA>` just like `someFunction`.
Inside the catch block, the implementor can either resolve the error by returning a `R` instance, or return a `ErrorA` using the `throw` keyword.

Hence the compiler should translate the code above to something like:

```

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

The example above shows the compiler has to be able to split the try block into a chain of statements, where each "split" occurs when a call to a function returning a Result<,> is made.
If the result is Ok for the current node, the next "statement node"  is executed.
If an error occured the catch block handling the corresponding error Type is called.

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

-With such a feature, Rust code would greatly gain in readability and conciseness.
-In the try block, Eliding any Result<T,E> into T would allow to chain calls recursively, without dealing with potential error.  

# Drawbacks

Such feature wouldn't have any impact on existing code in my knowledge.

# Alternatives

I've considered a single catch block, that would require a new trait, to be implemented by all errors, however this would involve a breaking change.

# Unresolved questions

Implementing a stack trace would be useful. 
