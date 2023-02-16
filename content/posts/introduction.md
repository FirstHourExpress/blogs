# Examples


## Structs and Traits

Structs and traits are nifty tools that can help us organize our code and make it more modular.

In Rust, a struct is a way to group related pieces of data together. A trait defines a set of behaviors that a type can have. It's similar to an interface in other languages but with added features. Traits are used extensively in Rust for writing generic, reusable code that works with various types. When a type implements a trait, it must provide implementations for all the trait's methods. This allows code to call those methods on any type that implements the trait, without knowing anything about the specific type.

In this example, we define a Point struct with latitude, longitude and name fields. By using a trait to define the distance and duration methods, the code can be reused for any structs that conform to the Distance trait. The distance is calculated using Haversine formula.

```rust
struct Point {
    name: String,
    latitude: f64,
    longitude: f64,
}

trait Distance {
    fn distance(&self, other: &Self) -> f64;
    fn duration(&self, other: &Self, speed: f64) -> f64;
}

impl Distance for Point {
    fn distance(&self, other: &Self) -> f64 {
        let d_lat = (other.latitude - self.latitude).to_radians();
        let d_lon = (other.longitude - self.longitude).to_radians();
        let lat1 = self.latitude.to_radians();
        let lat2 = other.latitude.to_radians();

        let a = (d_lat / 2.0).sin().powi(2) + lat1.cos() * lat2.cos() * (d_lon / 2.0).sin().powi(2);
        let c = 2.0 * a.sqrt().atan2((1.0 - a).sqrt());
        let r = 6371.0; // Earth's radius in km

        r * c
    }

    fn duration(&self, other: &Self, speed: f64) -> f64 {
        self.distance(other) / speed
    }
}

fn main() {
    let p1 = Point { name: "Wellington".to_string(), latitude: -41.2923, longitude: 174.7788 };
    let p2 = Point { name: "Cologne".to_string(), latitude: 50.9375, longitude: 6.9603 };
    let speed = 80.0; // km/h
    println!("The distance between {} and {} is {:.2} km.", p1.name, p2.name, p1.distance(&p2));
    println!("The time to travel from {} to {} at {:.0} km/h is {:.2} hours.", p1.name, p2.name, speed, p1.duration(&p2, speed));
}

```

`self` and `other` are parameters that refer to the current instance of Point or the other instance being compared or operated on.

`&Self` is a reference to the struct instance that is being passed in as a parameter. It's commonly used as a parameter type for trait methods so that the method can be called on any type that implements that trait. The & indicates that it's a reference to the instance rather than a copy, which can save memory and improve performance.

## Generics

Generics allow for code to be written that can work with different types without specifying the exact type. This reduces code duplication and improves reusability. In Rust, generics are defined using angle brackets (`<>)`.


```rust
use std::ops::Mul;

struct Measurement<T> {
    value: T,
    unit: String,
}

trait ConvertTo<T> {
    fn convert(&self) -> Measurement<T>;
}

impl<T: Mul<f64, Output = T> + Clone> ConvertTo<T> for Measurement<T> {
    fn convert(&self) -> Measurement<T> {
        let converted_value = self.value.clone().mul(2.54);
        Measurement {
            value: converted_value,
            unit: "cm".to_string(),
        }
    }
}

fn main() {
    let inches = Measurement {
        value: 10.0,
        unit: "inch".to_string(),
    };
    let centimeters: Measurement<f64> = inches.convert();
    println!(
        "{} {} is equal to {} {}.",
        inches.value, inches.unit, centimeters.value, centimeters.unit
    );
}

```

The Measurement struct uses a generic type T to allow for the value to be of any numeric type. We implement ConvertTo for Measurement<f64> to convert inches to centimeters, but we could implement it for any other type as well.

`<T: Mul<f64, Output = T> + Clone>` is a generic type constraint that specifies that the generic type T must implement the Mul trait, with f64 as the input type and T as the output type. We also add the Clone trait as a bound on the type parameter T.

## Handling Errors

`Ok` and `Err` are used to handle and propagate errors.

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Cannot divide by zero".to_string())
    } else {
        Ok(a / b)
    }
}

fn main() {
    let result = divide(10.0, 5.0);
    match result {
        Ok(x) => println!("Result: {}", x),
        Err(e) => println!("Error: {}", e),
    }
}
```

The divide function takes two arguments `a` and `b`, and returns a Result<f64, String> indicating either the result of the division or an error message if the second argument is zero. In the main function, we call divide with arguments 10.0 and 5.0, and then use a match expression to handle the result. If the result is `Ok`, we print the value using `println!("Result: {}", x)`, otherwise we print the error message using `println!("Error: {}", e)`.


## HashMap iterator adaptors

A HashMap is a collection type in Rust that stores key-value pairs.`filter` and `filter_map` operate on the HashMap by consuming it and producing a new iterator with only the elements that satisfy the condition specified in the closure.

- An example for `filter`:

```rust

use std::collections::HashMap;

#[derive(Debug)]
struct Requirement {
    name: String,
    version: String,
}

impl Requirement {
    fn new(name: &str, version: &str) -> Requirement {
        Requirement {
            name: name.to_string(),
            version: version.to_string(),
        }
    }
}

fn main() {
    let mut software_requirements = HashMap::new();
    software_requirements.insert("Language", Requirement::new("Rust", "1.56.0"));
    software_requirements.insert("Database", Requirement::new("PostgreSQL", "13.4"));
    software_requirements.insert("Web Framework", Requirement::new("Rocket", "0.5.0"));
    software_requirements.insert("Template Engine", Requirement::new("Handlebars", "4.5.3"));

    // Filter for requirements with a version number greater than or equal to "1.0"
    let filtered_requirements: HashMap<_, _> = software_requirements
        .into_iter()
        .filter(|(_, requirement)| requirement.version >= "1.0")
        .collect();

    println!("{:?}", filtered_requirements);
}

```

The `into_iter()` method is a consuming iterator, which means that it takes ownership of the HashMap and allows iteration over its key-value pairs. This method is used to create an iterator that can be passed to the `filter()` method.

The `filter()` method is a method of the Iterator trait, and it returns a new iterator that only includes elements that satisfy a given predicate. In this case, the predicate is a closure that checks if the version of a requirement is greater than or equal to "1.0".

The `collect()` method is then called on the filtered iterator, which collects the remaining key-value pairs into a new HashMap. The `collect()` method is also a method of the Iterator trait, and it collects the results of the iterator into a collection type.

- An example for `filter_map`:

```rust
use std::collections::HashMap;

#[derive(Debug)]
struct Requirement {
    name: String,
    version: String,
}

impl Requirement {
    fn new(name: &str, version: &str) -> Requirement {
        Requirement {
            name: name.to_string(),
            version: version.to_string(),
        }
    }
}

fn main() {
    let mut software_requirements = HashMap::new();
    software_requirements.insert("Language", Requirement::new("Rust", "1.56.0"));
    software_requirements.insert("Database", Requirement::new("PostgreSQL", "13.4"));
    software_requirements.insert("Web Framework", Requirement::new("Rocket", "0.5.0"));
    software_requirements.insert("Template Engine", Requirement::new("Handlebars", "4.5.3"));

    // Use filter_map to apply a transformation and filter the results
    let filtered_requirements: HashMap<_, _> = software_requirements
        .into_iter()
        .filter_map(|(_, requirement)| {
            if requirement.version >= "1.0" {
                Some((
                    requirement.name.to_uppercase(),
                    Requirement {
                        name: requirement.name,
                        version: requirement.version,
                    },
                ))
            } else {
                None
            }
        })
        .collect();

    println!("{:?}", filtered_requirements);
}
```

The `filter_map` method applies a transformation to each element in the iterator and then filters the results based on whether the transformation returns `Some `or `None`. In this case, we transform each (key, value) pair into a tuple of the form (`uppercase_name`, `requirement`) where `uppercase_name` is the uppercase version of the requirement's name. We only keep the transformed pairs where the version number is greater than or equal to "1.0".