---
title: "Using python to configure rust Part 2"
date: 2025-11-26T18:02:00+00:00
author: "Emily Selwood"
draft: false
slug: "pyo3_config_part_2"
tags: ["rust", "python", "pyo3"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---

Continuing on from [Part 1](/post/pyo3_config_part_1) If you've not read that you probably should before continuing here. Previously we setup a pyo3 project and ran some python code from rust which was able to create a rust class and return it to rust. (I think I can say rust a few more times in this paragraph. Rust rust rust)

This is the kind of python config we are aiming to be able to use inside our rust program:

```python
@configure
def configA():
  return Config(
    name= "blah"
  )

@configure
def configB(configA):
  return Config(
    name= configA.name + "_with_something_extra"
  )

@configure
def configC(configA, configB):
  return Config(
    name= configB.name + "_and_even_more_" + configA.name,
  )
```

Note the use of the `@configure` decorator to mark functions that do configuration stuff. That is the first thing we are going to do today.

## Step 3: Find all the functions with the decorator

Slight side bar: what the heck is a decorator?

In python a decorator is a bit of syntactic sugar for wrapping a function in another one. Defined by a function that takes the function its "attached" to as a parameter and returns a function. We could implement this `configure` decorator in python like so:

```python
LIST_OF_FUNCTIONS=[]

def configure(inner_func):
    LIST_OF_FUNCTIONS.append(inner_func)
    return inner_func
```

We could then loop over `LIST_OF_FUNCTIONS` and we'd have all our functions. 

Of course we need that list in rust. Which isn't the easiest thing to create, what we just made in python is a global mutable variable. Which is a terrible idea as soon as more than one thread is involved. So rust doesn't let you do that. 

Let me introduce you to the [append only vec](https://crates.io/crates/append-only-vec)

Once that's added to our `Cargo.toml` we can add this to our rust python module. 

```rust
use append_only_vec::AppendOnlyVec;
pub static CONFIGURE_FUNCTIONS: AppendOnlyVec<Py<PyAny>> = AppendOnlyVec::new();

#[pyfunction]
pub fn configure(py: Python, inner: Py<PyAny>) -> PyResult<Py<PyAny>> {
    CONFIGURE_FUNCTIONS.push(inner.clone_ref(py));
    Ok(inner)
}
```

Please don't ask me how that works but the `AppendOnlyVec` is doing some magic that lets it add to its self with out being a modifiable version of its self. In the python code we can import the configure function along with the Config object and we can wrap our configure functions in the decorator as in the first code block.

```python

from pyconfigmod import Config, configure

@configure
def configA():
  return Config(
    name= "blah"
  )

@configure
def configB(configA):
  return Config(
    name= configA.name + "_with_something_extra"
  )

@configure
def configC(configA, configB):
  return Config(
    name= configB.name + "_and_even_more_" + configA.name,
  )

```

Now we have a list of functions we can loop over them. However we don't know what their parameters are or what order we need to execute them in. So ...


## Step 4: Finding the arguments for the functions.

At the moment we have a list of function references. We don't know what they are called and we don't know what arguments they take.

Python objects have a bunch of built in attributes commonly called magic methods, special methods, or dunder methods. They are the ones that have two underscores around their names. There is a helpful list of them in the [python documentation](https://docs.python.org/3/reference/datamodel.html#special-method-names).

The one we need here is one I've not used in python before. `__code__` This contains information about the code that created the object. Details can be found, as usual, in the [python docs](https://docs.python.org/3/reference/datamodel.html#code-objects). For our needs we can use the `co_name` attribute to get the function name. I'm not using the `co_qualname` because I've not decided on a good mapping for characters that aren't allowed in parameter names. This does mean we need all `@configure` functions to have unique names, which kind of sucks currently. This can be a project for later.

The arguments to the function are a little more complex. Rather than separating out the arguments, they are part of the list of local variables. There is `co_argcount` which will give us how many there are, so we can pull them from the front of the list of local variables.

First lets get the function name

```rust
fn get_func_name<'l>(_py: Python<'l>, func: &Bound<'l, PyAny>) -> PyResult<String> {
    if !func.is_callable() {
        // TODO: figure out error handling
        panic!("Object not callable.")
    }

    let code = func.getattr("__code__")?;
    let name: String = code.getattr("co_name")?.extract()?;
    Ok(name)
}
```

I really need to sort out error handling at some point, but I'm still in prototype mode so I'm just going to panic if we get given something that isn't a function. 

In the function we get the `__code__` attribute and then the `co_name` attribute from that, then we are done.

Now we can move on to the arguments.

```rust
fn get_arg_names<'l>(_py: Python<'l>, func: &Bound<'l, PyAny>) -> PyResult<Vec<String>> {
    if !func.is_callable() {
        // TODO: figure out error handling
        panic!("Object not callable.")
    }

    let code = func.getattr("__code__")?;

    let arg_count: i32 = code.getattr("co_argcount")?.extract()?;
    let bound_var_names: Bound<'l, PyAny> = code.getattr("co_varnames")?;

    if bound_var_names.is_instance_of::<PyTuple>() {
        let tuple = bound_var_names.cast::<PyTuple>()?;
        let mut result = Vec::new();
        for i in 0..arg_count as usize {
            let arg_name: String = tuple.get_item(i)?.extract()?;

            result.push(arg_name);
        }

        Ok(result)
    } else {
        panic!("co_varnames wasn't a tuple")
    }
}
```

This is a bit more complex ... We grab the `__code__` attribute. We grab the `co_argcount` from it so we know how many arguments will be on the front of the `co_varnames` tuple. We then have to do some slightly nasty casting to get our `PyAny` into a `PyTuple` which we can then loop over the `arg_count` entries and pull out the argument names.

I feel ok panicking if the `co_varnames` attribute isn't a tuple given that its a built in python type and if that goes wrong the entire python environment is likely in flames so we won't be able to do much any way. Should probably check that the tuple actually has `arg_count` entries too but again, if the python environment we are using has gone that far wrong we have bigger issues.

Now that we have a way to get the name and arguments of a python function in rust land, we can build a `HashMap` of function name to arguments. In our main, after we have attached all the modules, we can do something like the following.

```rust

        let mut dependencies: HashMap<String, Vec<String>> = HashMap::new();
        let mut functions: HashMap<String, Bound<PyAny>> = HashMap::new();
        for func in pyconfigmod::CONFIGURE_FUNCTIONS.iter() {
            let bind = func.bind(py);
            if bind.is_callable() {
                let func_name = get_func_name(py, bind)?;
                let mut arg_names = get_arg_names(py, bind)?;

                println!("func {} args: {:?}", func_name, arg_names);
                functions.insert(func_name.clone(), bind.clone());
                dependencies
                    .entry(func_name)
                    .or_insert_with(Vec::new)
                    .append(&mut arg_names);
            }
        }
```

Here we are going through the `CONFIGURE_FUNCTIONS` list we created with the decorator, which got run when we loaded the module. Binding the functions to our python instance, so we can pass around references nicely. Then if its callable grabbing its name and arguments and adding them to the dependencies map. I'm also creating a name to function look up at the same time, we will need that later.

# Part 5: Sorting our functions

Now that we have our functions and know the names of their arguments we can sort them to work out the order that they need to run so we have the arguments required for each function before we call it. This set of functions is called a Directed Acyclic Graph (or DAG for short) This is a bunch of fancy words to mean they form a tree structure, with out any loops (or cycles).

The really important thing for us is the lack of cycles. If we have functions that depend on each other then we will not be able to run them because there is no way for us to start. E.G

```python
@configure
def configA(configB):
    return Config(name="Config A" + configA.name)
    
@configure
def configB(configA):
    return Config(name="Config B" + configA.name)
```

To work out `configB` we need `configA` and to work out `configA` we need `configB` which is impossible for us to do. So we will be raising an error if this happens. 

There are quite a few guides on doing a DAG sort online. I'm going to include the code I wrote here and explain it, as its not too long.

```rust

fn remove_args(funcs: &mut HashMap<String, Vec<String>>, name: &str) {
    for (_k, v) in funcs.iter_mut() {
        if let Some(pos) = v.iter().position(|e| e == name) {
            v.remove(pos);
        }
    }
}

fn sort_functions(funcs: &HashMap<String, Vec<String>>) -> Vec<String> {
    let mut working = funcs.clone();

    let mut result = Vec::new();

    while !working.is_empty() {
        let keys: Vec<String> = working.keys().cloned().collect();
        let mut found_something = false;
        for k in keys {
            let args = working.get(&k).unwrap();
            if args.is_empty() {
                result.push(k.clone());

                remove_args(&mut working, &k);

                working.remove(&k);
                found_something = true;
                break;
            }
        }
        // if we get here without breaking then we must have a loop or arguments that don't match functions.
        if !found_something {
            // TODO: real errors here please
            println!("working left with: {:?}", working);
            panic!(
                "Could not find any functions that have no parameters left. There must be a loop or parameters that don't match any functions."
            )
        }
    }

    return result;
}
```

First we need a modifiable copy of our hashmap of functions and their arguments, we need to remove things from this as we work so its nicer to take a clone of it here.

Then we get on to the actual algorithm its self. This looks a bit more complex due to rust and the borrow checker, but the principle remains the same. Look through our working map of functions, find one that doesn't have any arguments. This means it has no dependencies, so add it to the result list. Then go through the working map and remove any arguments with the name of the function we have. We also need to remove the function from the working map or we will end up with an infinite loop.

Then we start again, and we keep going until the working map is empty. If we ever manage to loop over all the entries in the working map and don't find an entry that has no arguments, it means we have a loop somewhere, or we have argument names that don't match a function we know of. Either way that's an error and we need to tell the user about it, or in our case panic and stop the program.

One thing to note about this implementation is it is not entirely stable, by design. If there are two functions that take the same arguments, we don't care what order they run in. `working.keys()` returns the keys in a random order. If two configs don't have a defined order it should not matter which order they run in. If there is accidentally a dependency then that is a bug and they should have an extra parameter to make the ordering defined.

# Part 6: Running the configure functions

Now that we have the information about all the configure functions and know the order they need to be executed in, we can start running them.

```rust
        let sorted_functions = sort_functions(&dependencies);
        println!("{:?}", sorted_functions);

        // execute the functions, keeping track of their results, looking up the arguments as needed.
        let mut results_map: HashMap<String, Config> = HashMap::new();
        for func_name in sorted_functions {
            println!("Executing {}", &func_name);
            let args = dependencies.get(&func_name).unwrap();
            let kwargs = create_kwargs(py, &results_map, args)?;
            let result: Config = functions
                .get(&func_name)
                .unwrap()
                .call((), Some(&kwargs))?
                .extract()?;

            results_map.insert(func_name.clone(), result);
        }

        println!("results: {:?}", results_map);

```

The code here loops over the list of function names that comes out of the sort function. Builds a set of `kwargs` (Key word args in python land) which is a `PyDict` that gets expanded into named arguments. Calls the function with those kwargs and finally stores the result for later.

When we create the kwargs we have to build a `PyDict`, I also convert our `Config` objects into `ReadOnlyConfig` objects so that later configure functions can not mess with the results of earlier functions. This is a near clone of the original `Config` object but it doesn't have the constructor function or the `set_all` argument on the `pyclass` macro. There is also a function to create one from a `Config` object.

The `create_kwargs` is as follows:

```rust
fn create_kwargs<'l>(
    py: Python<'l>,
    existing_results: &HashMap<String, Config>,
    args: &Vec<String>,
) -> PyResult<Bound<'l, PyDict>> {
    let result = PyDict::new(py);

    for arg in args {
        let cr = existing_results.get(&arg.to_string());
        if cr.is_none() {
            panic!(
                "Couldn't find existing result for {}. This shouldn't be possible if the sort worked right",
                arg
            );
        }

        result.set_item(arg, ReadOnlyConfig::from_config(cr.unwrap()))?;
    }

    Ok(result)
}

```

As usual the error handling can be massively improved here, but something has gone horribly wrong if we don't have a result for a function we need. The sort would have had to have broken somehow. 

Now we have a fully working configuration system, if MVP level of implementation, rather than production code. Lets run it on our config and see what happens.

```bash
emily@diamondslab:~/Projects/breezeblock$ cargo run
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/breezeblock`
func configA args: []
func configB args: ["configA"]
func configC args: ["configA", "configB"]
dependencies: 
{"configC": ["configA", "configB"], "configA": [], "configB": ["configA"]}
["configA", "configB", "configC"]
Executing configA
Executing configB
Executing configC
results: {"configB": Config { config_class: "", repository: "", name: "blah_with_something_extra", features: [], parameters: {} }, "configC": Config { config_class: "", repository: "", name: "blah_with_something_extra_and_even_more_blah", features: [], parameters: {} }, "configA": Config { config_class: "", repository: "", name: "blah", features: [], parameters: {} }}
```

In the results section at the end we can see it has done what we wanted. 

There are many improvements that could be made here, but the basic principle works just fine. Now we can go and do something with our config objects.

I've pushed the full code up to [codeberg](https://codeberg.org/emily_s/breezeblock) so if you want to play around with it feel free. If you find this interesting or want to use it in something I'd love to hear from you, drop me a message on mastodon @emily_s@mastodon.me.uk
