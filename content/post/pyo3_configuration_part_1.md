---
title: "Using python to configure rust Part 1"
date: 2025-11-24T21:02:00+00:00
author: "Emily Selwood"
draft: false
slug: "pyo3_config_part_1"
tags: ["rust", "python", "pyo3"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---
I've recently been playing with running python code from rust to use as a kind of configuration and extension system. The idea is that my program runs a bunch of python code that the user has defined and that gives it the configuration it needs. The user has the *POWER OF THIS FULLY OPERATIONAL PROGRAMMING LANGUAGE* cough cough, sorry voice box got stuck on imperial functionary there.

I'm doing this as a personal project and proof of concept so don't expect bullet proof code here. If you want to do something similar then you will need to put some effort into hardening this. I'll also be pretty vague about what is being configured here as I don't think it matters right now. Think of something like helm charts or terraform config. Something that would usually be configured by a bunch of yaml files and maybe some templates. I want to know why they aren't configured with code, so I'm experimenting to find the reasons.

This is going to be a series of posts and to avoid it being a book, I'm going to assume you are familiar with rust and python. You know how to set up a project in both languages and are comfortable with their memory models, module systems, and standard libraries. 

We can run python code in rust using the library [pyo3](https://pyo3.rs). To do this we needed to load the users code, find the right functions to call in the users code, and then run them. I could make them name their function something particular, but that doesn't give much power. It would be way nicer if they could add a decorator to their config functions and that allowed me to find them. What would also be cool is if we could chain them, so if the args to one function meant that it would call the function with that name first. E.g

```
@configure
def configA():
  return Config(
    name: "blah"
  )

@configure
def configB(configA):
  return Config(
    name: configA.name + "_with_something_extra"
  )

@configure
def configC(configA, configB):
  return Config(
    name: configB.name + "_and_even_more_" + configA.name,
  )
```

Where it is able to work out that it needs the results of `configA` and `configB` to be able to call `configC` 

There are a few steps here: 
1) Load all the python files in a folder into the python environment
1) Create a rust struct that python code can create for the config objects
1) Find all the functions with the `@configure` decorator on them
2) Work out what the arguments are for each function
3) Work out the order they should be executed in so we have the results needed as inputs to other functions
4) Execute the functions and save their results so we can pass them as parameters later.

This is going to be a lot  so I'll split this into several posts.

## Step 1: Load all the python files into the environment

pyo3 works in rust by creating an environment to run in. Python doesn't have lifetimes and all the other fun and wonderful memory safety we have in rust. So all the python variables and memory gets put inside the pyo3 environment where it gets the lifetime of the environment. If you don't understand this, don't worry, its rust memory model stuff, you probably aren't the intended audience for this (sorry)

I'm going to expect you've got a rust environment set up and a python environment setup. I'm on an old debian currently and have python 3.11.2 but I've not seen anything that should break using different versions. I've just not tested it. 

Starting with a fresh `cargo new` project and adding `pyo3` to the dependencies. 

```
pyo3 = {version="0.27", features=["auto-initialize"]}
```

Now lets create the python environment inside our rust main.

```rust
use pyo3::prelude::*;
fn main() -> PyResult<()> {
    Python::attach(|py| {
        // do something in python here
    });
}
```

Inside that attached python block we can do things in python, see the pyo3 docs for [examples](https://pyo3.rs/v0.15.0/python_from_rust) Here I'm going to jump right into loading the users code from a folder somewhere.

pyo3 has a function to create a python module from a string. We can reasonably easily load a file into a string. One wrinkle is we need to use `CString` rather than the usual rust `String` because python operates in c land and we have to give it strings it will be able to comprehend.

```rust
fn attach_module<'l>(
    py: Python<'l>,
    root_dir: &Path,
    input_path: &Path,
) -> PyResult<(CString, pyo3::Bound<'l, pyo3::types::PyModule>)> {
    let file_name = CString::new(input_path.file_name().unwrap().as_encoded_bytes())?;
    let module_name = CString::new(
        input_path
            .strip_prefix(root_dir)
            .unwrap()
            .to_str()
            .unwrap()
            .replace("/", ".")
            .strip_suffix(".py")
            .unwrap(),
    )?;

    let content = CString::new(fs::read_to_string(input_path)?)?;
    let module = PyModule::from_code(py, &content, &file_name, &module_name)?;

    Ok((module_name, module))
}
```
The first thing is the mess of a type signature. The lifetime parameter `'l` tells rust that the life time of the py parameter is going to be the life time of our results as well. We are going to be returning a `PyResult`, which is like a built in `Result` but it always returns a `PyErr` on errors. In the good case we are going to return a tuple of a CString (the module name) and a `pyo3::Bound` which acts like a rust `Rc` to the module with a lifetime. 

Then there is a fair chunk of mess in there to convert the file path to a python module name. I'm making the assumption that you are using an operating system that cleanly converts from `OsString` to `String`. I've worked on windows, linux, and macs and I've not seen that conversion fail so for this code I'm ok with this assumption. It also assumes that its been given a string with `.py` extension, you'll see why I'm ok with this in a minute. 

Reading the file and converting it to a `CString` is a single line. (Its fun how some really complex things are easy in rust but some things are a pain like taking a file path and replacing all the / with spaces :) )

Once we've got the text of the file, the filename and the module name we can call the `PyModule::from_code` to create the module. We have to hand it a reference to the python environment we are using so it knows where to create the module. 

This handles a single file for us. Easy enough. Lets go one more level and load all the files in a directory, even if they reference each other.

To read all the files in a directory, including sub directories we can use the [WalkDir](https://docs.rs/walkdir/latest/walkdir/) crate. This can give us an iterator over a directory and we can use the filter functions to find only the things we are interested in.

```rust
fn attach_modules<'l>(
    py: Python<'l>,
    input_path: &str,
) -> PyResult<HashMap<String, pyo3::Bound<'l, pyo3::types::PyModule>>> {
    let mut result = HashMap::new();
    
    let root_path = Path::new(input_path);
    let py_str = OsStr::new("py");
    
    for entry in WalkDir::new(input_path)
        .into_iter()
        .filter_map(|e| e.ok())
        .filter(|e| e.metadata().unwrap().is_file())
        .filter(|e| e.path().extension().unwrap_or(OsStr::new("")) == py_str)
    {
        let path = entry.path();
        let (name, module) = attach_module(py, &root_path, path)?;
        result.insert(name.into_string()?, module);
    }

    Ok(result)
}

```

This time our function takes a single path, where we are going to work from to find our python files. We create a HashMap (like a python dictionary) to use as the result then start using `WalkDir` First we filter out anything that the OS didn't let us read (permissions fun etc) then we filter for only files, then for paths which have the "py" extension. We then use the entries that gives us to call the `attach_module` function we created just now. 

So now we've been able to create all our modules. However they aren't actually available to each other. They are just Module objects, they are not really part of the python "class path" to mix my language concepts. Python manages its modules by keeping a dictionary in the built in `sys` module called `modules` when ever you try to import something in your code it looks in that dictionary to find it and errors if its not there.

We can do this in rust with a few modifications to the above function

```rust
fn attach_modules<'l>(
    py: Python<'l>,
    input_path: &str,
) -> PyResult<HashMap<String, pyo3::Bound<'l, pyo3::types::PyModule>>> {
    let mut result = HashMap::new();

    let sys = PyModule::import(py, "sys")?;
    let py_modules: Bound<'l, PyDict> = sys.getattr("modules")?.cast_into()?;

    let root_path = Path::new(input_path);
    let py_str = OsStr::new("py");
    for entry in WalkDir::new(input_path)
        .into_iter()
        .filter_map(|e| e.ok())
        .filter(|e| e.metadata().unwrap().is_file())
        .filter(|e| e.path().extension().unwrap_or(OsStr::new("")) == py_str)
    {
        let path = entry.path();
        let (name, module) = attach_module(py, &root_path, path)?;

        py_modules.set_item(&name, &module)?;

        result.insert(name.into_string()?, module);
    }

    Ok(result)
}
```

We import the "sys" module then get hold of the modules dictionary using the getattr method. We'll be using this a lot. It returns a `PyResult` that contains a reference to the python version of the value we asked for. This can be any attribute of the python object in question, in this case a global variable in the sys module. We can then call `py_modules.set_item` to insert the entry into the dictionary. Now all the modules should be able to find each other when we call them.

We've successfully loaded all the python files in the directory. So on to Step 2.

## Step 2: Create a rust config struct that can be constructed in python.

What we want here is a rust struct that we can do something like this to

```python
from somewhere import Config
def construct():
    c = Config(name: "foo")
    c.something = "a value"
    
    return c
```

Unfortunately we can't just import a rust type and have it work. That would be too much magic. There are some hoops we need to jump though.

The first is creating a new python module that we can import. I've created a new rust file to put the config related stuff in. `config.rs` pyo3 has a macro to do this for us.


```rust
use pyo3::prelude::*;

#[pymodule]
pub mod pyconfigmod {


}
```

Though again, like with us loading the python code this doesn't add it to the python world so we have do that separately. This needs a line before we attach to python, back in the `main.rs` file.

```rust
pub mod config;

use config::pyconfigmod;

fn main() -> PyResult<()> {
    pyo3::append_to_inittab!(pyconfigmod);
    Python::attach(|py| {
```

Now we can create the class, it looks like a normal rust struct but with some extra macros, back in the `config.rs` we can add the struct. 

``` rust
use pyo3::prelude::*;

#[pymodule]
pub mod pyconfigmod {
    
    use pyo3::prelude::*
    
    #[pyclass(set_all, get_all)]
    #[derive(Debug, Default, Clone)]
    pub struct Config {
        pub name: String,
    }
    
}
```

Note: We need to import the prelude again inside the module. (Don't ask me why, this is a bit of rust I've not figured out yet, and don't really need to understand right now. This works.)

By default pyo3 treats python classes as immutable. It makes it easier to reason about the thread safety of everything. However if you add the `set_all` and `get_all` parameters to the `pyclass` macro then you get generated getters and setters that take care of that synchronization mess for you. The final thing we need for this to work how we want is a constructor. Currently if we try to construct this thing in python it will tell us that it can't be constructed.

This also shows us how to add custom methods to our python class.

```rust
use pyo3::prelude::*;

#[pymodule]
pub mod pyconfigmod {
    
    use pyo3::prelude::*
    
    #[pyclass(set_all, get_all)]
    #[derive(Debug, Default, Clone)]
    pub struct Config {
        pub name: String,
    }
    
    #[pymethods]
    impl Config {
        #[new]
        fn __new__(name: String) -> Config {
            let result = Config{name: name};
                
            result
        }
    }
}
```

While this is a very simple function we need to define it our selves. 

As a result we are able to create a python file that looks like this

```python
from pyconfigmod import Config

def something():
    print("configuring something")
    c = Config("my name")
    # Eventually do some thing more complex to create this here.
    return c
```

Back in main we should run this for the grand payoff of seeing it print out something from our Config class. Back in our `main.rs` file we've got something like this.

```rust
pub mod config;

use config::pyconfigmod;

// ... module loading code here ...

fn main() -> PyResult<()> {
    pyo3::append_to_inittab!(pyconfigmod);
    Python::attach(|py| {
        modules = attach_modules("/home/emily/Projects/my_config");
    });
}
```

With the little python script in the previous code block in `/home/emily/Projects/my_config` we now have the module. So we can test this and see it working we will loop through each of the module and see if we can find an attribute called `something` and if its callable we will call it.

```rust
pub mod config;

use config::pyconfigmod;

// ... module loading code here ...

fn main() -> PyResult<()> {
    pyo3::append_to_inittab!(pyconfigmod);
    Python::attach(|py| {
        modules = attach_modules("/home/emily/Projects/my_config");
        
        for (module_name, module) in modules {
            if module.hasattr("something")? {
                let func_something = module.getattr("something")?;
                println!("Found something in {}", module_name);
                if func_something.is_callable() {
                    let result: Config = func_something.call((), None)?.extract()?;
                    println!("Config received! Name: {}", result.name);
                }
            }
        }
    });
}
```

If we run this with `cargo run` we should see something like

```shell
emily@diamondslab:~/Projects/breezeblock$ cargo run
   Compiling breezeblock v0.1.0 (/home/emily/Projects/breezeblock)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.44s
     Running `target/debug/breezeblock`
Found something in something
configuring something
Config received! Name: my name
```

This is a good start. We've got the ability to load a folder full of python files into rust. Create a rust class in python, and call functions in python from rust and get back a rust class. Next time we will work on finding python functions with a custom decorator and then working out what parameters those functions take.
