+++
title = "Ephesus: a PPL for hybrid graph-relational databases"
date = 2025-06-13
draft = false

[taxonomies]
tags = ["Probabilistic Programming", "AI/ML", "Ephesus", "Rust"]

[extra]
author = "Baxter Eaves"
type = "post"
image = "/img/robot-looking-down-on-library.webp"
+++

[Redpoll](https://redpoll.ai/) is a data quality company. Finding and monitoring for errors and anomalies is our bread and butter, and it's also something that AI/ML-at-large has had a really hard time with. The reason that AI/ML has such a hard time is that an anomaly or an error is data that "doesn't make sense", and to know what doesn't make sense you have to know what does. You have to *understand* the process behind the data. And, trad AI/ML, like boosted trees and neural nets, doesn't do understanding. Understanding is something explicit process models are awesome at. We define a model of the process that generates the data; then data that are inconsistent with that model are errors or anomalies. Bayesian hierarchical modeling allow us to assign likelihoods to observations under a generative process model, which is even better. Now "erroneous-ness" and "anomalous-ness" are on a continuum. But there are problems: what if we don't know how to model the data, or the data we have don't fit into the constraints of modeling tools, or we have more data than modeling tools can handle? The vast majority of data we deal with at Redpoll sit in giant relational/graph databases with many billions of records of sparse and heterogenous data that nobody has any clue how to model holistically--it's not really something you can just throw into [STAN](https://mc-stan.org/) or [Lace](https://www.lace.dev/). So, what now? There's only one thing to do: build a paradigm-shifting [probabilistic programming language (PPL)](https://en.wikipedia.org/wiki/Probabilistic_programming) backed by Bayesian Nonparameterics that scales to petabyte data.

# Enter Ephesus
Ephesus is a rust-inspired PPL for hybrid relational/graph databases that--through the magic of Bayesian nonparametrics--builds a joint Bayesian model over the entire database without having to define an explicit generative process.

Ephesus is written in [rust](https://www.rust-lang.org/), using [pest](https://pest.rs/) for parsing, and backed by [polars](https://pola.rs/) for data storage and streaming. It's very early days for Ephesus, but here's what we have.

Ephesus builds on the idea that a database schema is a model specification describing how the data flow and interact. Our goal with Ephesus is to create a modeling framework the keeps the internals of the model as close as possible to the original data representation. So, Ephesus models are specified in a schema and built/fit/queried in rust. For Example, to define a model of an image using two tables, we'd define the schema like this:

```
// schema.ephesus
model: MLaplace;

table X[Id] {
  dens x: Float,
}

table Y[Id] {
  fk Id: X.Id,
  dens y: Float,
}
```

We specify the model name, `MLaplace`, which will be converted into a rust module (you'll see this later). Then we define the tables. `table X[Id]` says, we have a table called `X` that has an index column called `Id`. The table `X` has one feature: `dens x: Float`, which says that feature `x` is never missing (`dens`), and takes on floating point values. The table `Y` is basically the same, but it has this `fk Id: X.Id` bit, which says that table `Y` has a foreign key called `Id` that indexes into table `X` by table `X`'s `Id` column. That's it. We could have just modeled it as one table like this

```
model: MLaplace;

table Coords[Id] {
  dens x: Float,
  dens y: Float,
}
```

but using two tables is more fun.

Now what? We use rust procedural macros to turn the schema into rust code and interact with the model like anything else in rust.

```rust
use ephesus::codegen::load_schema;
use ephesus::utils::load_parquet;
use ephesus::ColumnIndex;

// Use a procedural macro to parse the schema and generate the model code
load_schema!("schema.ephesus");

fn main() {
  let mut rng = rand::thread_rng();

  // Load in the data for each table
  let df = load_parquet("m-laplace.parquet").unwrap();
  // Both columns are in the one parquet file
  let data = MLaplace::TableData {
    X: df.clone(),
    Y: df,
  };

  // Initialize the model
  let mut model = MLaplace::Model::init(data, &mut rng);

  // Run MCMC (fitting) for 1000 steps
  model.fit(1_000, &mut rng);

  // Save our work
  model.write_metadata("m-laplace.metadata").unwrap();

  let coords = [
    ("X", vec!["x"]),  // 'x' from table 'X'
    ("Y", vec!["y"]),  // 'y' from table 'Y'
  ];

  // Draw from the joint distribution of x and y
  let simulated_output = model.simulate(
    coords.try_into().unwrap(), // Output variables
    None,                       // No "given" conditions
    100_000,                    // Number of samples to generate
    &mut rng
  ).unwrap();

  // ...save off the output

  // compute the (log) likelihood that X.x = 12.0 given that Y.y = 4.2
  let logp = model.logp(
    [("X", vec!["x".equals(12.0)]).try_into().unwrap(),
    Some([      
      ("Y", vec!["y".equals(4.2)]),
    ].try_into().unwrap())
  ).unwrap();
}
```

The above rust code builds the model code from the schema, initializes the model, fits the model, saves the model metadata, simulates a bunch of synthetic data from the model, and computes the conditional likelihood of some data given some observation. Similar to Lace, Ephesus allows you to query (via `simulate` and `logp`) *any* conditional distribution of the form `p(features|other_features)` without any extra modeling or training.

What about inference quality? Here is an example of the simulated output from the `MLaplace` model:

![Ephesus recovering the joint distribution of an image.](laplace.gif)

On the left we have the original data, and on the other left (right) we have the simulated output at different steps of the fitting procedure.

### Aside: Why build a language? Why not just use rust or JSON or something?

1. Using code generation allows us to build fast, cache-efficient models. The alternative is using dynamically sized containers and enums all over the place, which would both considerably slow down the code and increase the memory footprint. Code generation also means less fiddling with trait bounds.
2. We could of course build models using specs written in something like JSON or YAML, but using the custom schema language saves thousands of lines of text and is infinitely easier to read. Customer integration is already miserable work, and we'd rather not spend weeks writing 100,000 line JSON files.
3. I could have probably gotten away with using JSON schemas that deserialize into rust structs that convert into dynamic model objects, but I'm a bad startup founder and I'm [doing things that scale](https://paulgraham.com/ds.html) just because they're fun (and make better screenshots).

### Aside: Why not use deep learning?

Mainstream AI based on Artificial Neural Networks (ANNs) is notoriously bad at structured data problems. Apart from being opaque and uninterpretable, popular ANN-based approaches to graph and relational data like Graph Neural Networks (GNNs) and Relational Deep Learning (RDL) have a number of limitations:
- They use embeddings that distort the structure of the original data.
- They're expensive to scale, requiring special hardware like GPUs and TPUs to run in reasonable time.
- They're inflexible in the sense that each question you want to ask requires additional model building.
- They're bad at quantifying (aleatoric and epistemic) uncertainty, because 1) they don't do posterior sampling and 2) they're not guaranteed to build sane models of the data generating process.


## Doing more in the script

You might have noticed that, other than defining the data types, we didn't do any "real" modeling in the schema file. That's one of the great thing about Bayesian nonparametrics--it does inference over the form and complexity of the underlying model so you can get away with specifying basically nothing. That said, it still needs priors, so if we don't define them Ephesus will generate empirical priors by default. However, we often we know a little more than nothing about our data. In that case, Ephesus gives power users the ability to dig in a little deeper.

```
// schema.ephesus
model: MLaplace;

#[dataframe(path="m-laplace.parquet")]
table X[Id] {
  dens x: Float
    | Gaussian
    | Nix(m=0.5, k=1.0, v=1.0, s2=1.0),
}

#[dataframe(path="m-laplace.parquet")]
table Y[Id] {
  fk Id: X.Id,
  dens y: Float  
    | Gaussian
    | Nix
    | @ {
      m ~ Gaussian(m=0.5, s=1.0),
      k ~ Gamma(shape=1.0, rate=1.0),
      v ~ Gamma(shape=1.0, rate=1.0),
      s2 ~ InvGamma(shape=1.0, scale=1.0),
    },
}
```

We did a couple of things. Rustaceans will first notice the attribute `#[dataframe]`. This tells Ephesus just to load that tables data from a specific parquet file so we can skip steps when we build the model in the rust code. The next thing you'll notice is that we've added a lot more code to each column spec.

```
  dens x: Float
    | Gaussian
    | Nix(m=0.5, k=1.0, v=1.0, s2=1.0),
```

says that `x` is Gaussian distributed and that the parameters for the Gaussian have a Normal-Inverse-Chi-Squared (`Nix`) prior with the given parameters. In the `Y` table we took things a bit further and defined a hyperprior.

```
  dens y: Float  
    | Gaussian
    | Nix
    | @ {
      m ~ Gaussian(m=0.5, s=1.0),
      k ~ Gamma(shape=1.0, rate=1.0),
      v ~ Gamma(shape=1.0, rate=1.0),
      s2 ~ InvGamma(shape=1.0, scale=1.0),
    },
```

We said, we don't know what the parameters of the `Nix` prior are, so we'll defined an anonymous prior `@` over the `Nix` parameters. Not that anything we chose here is any good--it's just supposed to be illustrative.

## Other-than-continuous data

Great, we can make a model of continuous data. But Ephesus supports a lot more data types. I guess, technically, it supports *infinite* data types. In healthcare the vast majority of the data we deal with are timestamps or categorical. Take this table from our schema for the [MIMIC-III dataset](https://mimic.mit.edu/docs/iii/tables/admissions/)

```
table Admissions[hadm_id] {
  fk subject_id: Patients.subject_id,
  mcar admittime: Datetime,
  mcar dischtime: Datetime,
  mcar deathtime: Datetime,
  mcar admission_type: Categorical[AdmissionType],
  mcar admission_location: Categorical[AdmissionLocation],
  mcar discharge_location: Categorical[DischargeLocation],
  mcar insurance: Categorical[Insurance],
  mcar language: Categorical[Language],
  mcar religion: Categorical[Religion],
  mcar marital_status: Categorical[MaritalStatus],
  mcar ethnicity: Categorical[Ethnicity],
  mcar edregtime: Datetime,
  mcar edouttime: Datetime,
  mcar diagnosis: List[Diagnosis],
  mcar hospital_expire_flag: Binary,
  mcar has_chartevents_data: Binary,
}
```

We see there are `Categorical` columns, `Datetime` columns, `Binary` columns, and even a `List` column. And all the columns, other than the foreign key are `mcar`, which is "missing completely at random".

`Categorical` columns take a support argument. We can say `Categorical[3]` for a category that takes on values in {0, 1, 2}, or we can say `Categorical[Language]` where `Language` is an `enum` defined like this:

```
enum Language {
  English,
  Spanish,
  Arabic,
  French,
  "Ancient Greek",
  // .. etc
}
```

Explicitly defining variants like this is good for when you have an `enum` with like five variants. But what if there are thousands? We can infer them from the data.

```
#[infer(path="ADMISSIONS.csv", field="language")]
enum Language { .. }
```

The above will create the `enum` with all the unique variants in the `language` column of `ADMISSIONS.csv`.

We can also just store them in a different text file, which is a bit more explicit.

```
#[infer(path="language-variants.txt")]
enum Language { .. }
```

## Compound Data Types

So what about `Datetime`? How does that work? How does one model a `Datetime`? The short answer is "however you want". A compound type is a column that expands into a number of other columns. For example, we can define `Datetime` using the `struct` keyword

```
struct Datetime[DateTime(format="ISO-8601")] {
  dens day_of_week: Categorical[7],
  dens hour: Int[0..24]
    | Periodic,
  dens minute: Int[0..60]
    | Periodic,
  dens second: Int[0..60]
    | Periodic,
}
```

The key piece here is in the brackets,

```
[DateTime(format="ISO-8601")]
```

This says that this compound column is extracted using the (built-in) `DateTime` extractor with the `format` argument set to `"ISO-8601"` (format can take other common formats or a format string like `"%Y-%m-%d"`). The extractor expands the base column into a number of columns that we can select and specify models for in the body of the struct. In this case, we select `day_of_week`, `hour`, `minute`, and `second`. We model `day_of_week` as categorical, and model `hour`, `minute`, and `second` as periodic (wrapping) integers.

Now we can do things like

```
struct Datetime[DateTime(format="ISO-8601")] {
  dens day_of_week: Categorical[7],
  dens hour: Int[0..24]   | Periodic,
  dens minute: Int[0..60] | Periodic,
  dens second: Int[0..60] | Periodic,
}

enum EventType {
  Post,
  Delete,
  Edit,
}

table Event[Id] {
  dens event_type: Categorical[EventType],
  dens dt: Datetime,
}
```

We can also define anonymous compound types inline like this:

```
enum EventType {
  Post,
  Delete,
  Edit,
}

table Event[Id] {
  dens event_type: Categorical[EventType],
  dens dt: struct @[DateTime(format="ISO-8601")] {
    dens day_of_week: Categorical[7],
    dens hour: Int[0..24]   | Periodic,
    dens minute: Int[0..60] | Periodic,
    dens second: Int[0..60] | Periodic,
  },
}
```

And I forgot to mention it earlier, but you can do the same thing with enums:

```
table Event[Id] {
  dens event_type: Categorical[enum @{
      Post,
      Delete,
      Edit,
    }],
  dens dt: struct @[DateTime(format="ISO-8601")] {
    dens day_of_week: Categorical[7],
    dens hour: Int[0..24]   | Periodic,
    dens minute: Int[0..60] | Periodic,
    dens second: Int[0..60] | Periodic,
  },
}
```

which is kind of ugly IMO, but it's there if you want it.

### Custom compound data types

Here is my favorite part of the API. What if we have a new compound type that Ephesus doesn't have a built-in extractor for? We define one ourselves using rust! Procedural macros just expand to code in place. So, if we implement the `Extractor` trait for a rust struct matching the name of the extractor passed to our compound type, we're good to go.

Imagine that, instead of coming in two float-type columns, our image pixel coordinates came in a single string column defining coordinate pairs, e.g., `"1.2,3.1"`. We could do this:

```rust
use std::collections::BTreeMap;

use ephesus::Arg;
use ephesus::codegen::inline_schema;
use ephesus::extractors::Extractor;
use ephesus::extractors::ToDataFrame;
use ephesus::polars::frame::DataFrame;
use ephesus::polars::series::Series;

#[derive(ToDataFrame)]
struct Coords {
  x: f64,
  y: f64,
}

impl Extractor for Coords {
  type Base = String;

  fn extract(coords: Self::Base, _args: &BTreeMap<String, Arg>) -> Self {
      let Some((x, y)) = coords.split_once(',') else {
        panic!("Invalid coords '{coords}'")
      };

      Self {
        x: x.parse::<f64>().unwrap(),            
        y: y.parse::<f64>().unwrap(),            
      }
  }
}

inline_schema!{
  r#"
    model: Image;

    struct Coords[Coords()] {
      dens x: Float,
      dens y: Float,
    }

    table Xy[Id] {
      corods: Coords,
    }
  "#
}

fn main() {
  // ...
}
```

Simple as that. Behind the scenes, the `Base` associate type in `Extractor` tells Ephesus how to convert the polars `Series` to an iterator over the proper type. The `ToDataFrame` derive macro is required for converting an iterator over `Coords` to a polars `DataFrame`. Other than that, it's rust as usual.

# Does it scale?

"But Bax," you say, "Bayesian models are *slow*. General models are *slow*. Nonparameteric models are *slow*. So surely Ephesus is prohibitively slow." *Au contraire*. Using a single-table Ephesus model to cluster bivariate continuous data, we can go from 0 to 0.98 [ARI](https://en.wikipedia.org/wiki/Rand_index) (0 is random; 1.0 is perfect) on 1 Billion rows in under 12 seconds on an M4 Macbook Pro. Ephesus models can be also be backed with Memory Mapped files, and even remote data stores, when the data and model are too big to fit in memory.

For example,

```
#[data(store = "MmVec")]
table Xy[Id] {
  coords: Coords,
}
```

the above backs large vectors with memory mapped files. If we couple this with file systems like [lustre](https://www.lustre.org/), we can go really, really big.

Also, also, Ephesus is *embarrassingly parallel* and computation can be split across many machine--not just computation across multiple tables, but computations within single tables as well.

# The future & how to get involved

I apologize if I've gotten you terribly excited to try this. At present Ephesus is locked down under heavy development as part of our core [data QA/QC](https://redpoll.ai/) and **new** [secure federated inference tech](https://iixx.io). If you have a use case that Ephesus might help with, like quickly building Bayesian models across many structured datasets, please reach out to me on [linkedin](https://www.linkedin.com/in/baxtereaves/). I'm perfectly friendly and happy to help--whether it be with Ephesus or by pointing you down other avenues that might be a better fit.
