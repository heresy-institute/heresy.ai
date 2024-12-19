+++
title = "Making PPLs More Useful With Two New Operators"
date = 2024-12-20
draft = false

[taxonomies]
tags = ["Probabilistic Programming", "Opinion", "AI/ML"]

[extra]
author = "Baxter Eaves"
type = "post"
image = "/img/muddy-science.webp"
+++

Probabilistic programming is a counter play to black box machine learning. Probabilistic programming practitioners seek to build interpretable models of phenomena and to capture (correctly calibrated and representative) uncertainty in their models' understanding of those phenomena given the data observed. Generally speaking, building probabilistic models is a from-scratch affair. Whereas neural models with different architectures can easily be snapped together like Legos, precise probabilistic models often require developing novel hierarchies of novel distributions that are not always drag-and-drop. Probabilistic Programming Languages (PPLs) are domain-specific languages (DSLs) designed make building certain classes of Bayesian probabilistic models easier. But *easier* does not mean easy.

In practice, PPLs are not all that useful, generally, which is why we don't see them much in practice.

I argue that PPLs are not all that useful because they are hard to use, and that they are hard to use because they are not that useful. PPLs do not do inference over model structure, forcing users to do their own model comparison outside the model framework. Comparing models requires fitting many models for comparison. Because users now must rely on fitting many models, users require that models fit quickly, so PPLs optimize for speed of fit. In optimizing for speed of fit PPLs have had to make ease-of-use sacrifices. This has created an ecosystem of tools that both require highly-sophisticated users and have limited practical use. This is why PPLs have seen such poor adoption compared with black box models.

# PPLs are hard to use so people don't use them

Originally PPLs were pitched as a tool to help **domain experts** build principled Bayesian probabilistic models instead of relying on uninformative standard statistical tests. Clearly we've fallen far short of that goal. PPLs place a number of strict constraints on their users that have effectively made PPLs tools for PPL people. PPL users must

1. understand Bayesian statistics and probability distributions to the level that they can fully specify a parametric model of the phenomena in which they are interested,
2. understand the nuances of the inference routines underlying the PPL to determine when their model may cause the inference routine to degenerate,
3. understand how to transform or truncate their model to work with the inference routine (often involves calculus),
4. be advanced programmers in the host language of their PPL,
5. and some PPLs, mostly in python, require users to think in tensors.

Point 1 is only sensible. However requiring that the user fully specify their model down to the parametric form is probably unreasonable in a scientific context where the big questions are often of the form "are these related" or "what form does this relationship take".

Points 2 and 3 are interrelated and the source of many of the usability troubles. 


# Why PPLs aren't actually that useful

Black box methodology is winning the ML wars because it is easy. It is easier to use, it is easier to research, it is easier to publish, and is easier to get funding for. Popularity comes from ease. But despite its ease, [black box technology is not delivering when we need it to](@/2024-11-07-ai-leans-evil.md). We, as a society, need what Bayes has to offer &mdash; representativeness, realism, transparency, uncertainty/risk quantification &mdash; but while we can just drop data into a black box model and learn *something*, we cannot do the same thing with PPLs because PPLs require full specification of the model.

There are a number of levels of uncertainty we must address when fully specifying a model:

1. Structural: How many variables are there, which are dependent, and which are independent?
2. Relational: Given a group of statistically dependent, what is the parametric form defining their relationship?
3. Parametric: Given a parametric distribution, what are the likely values of the parameters?

Of these uncertainties, PPLs address only parametric uncertainty which constitutes the least amount of learning. To build a model in a PPL, we must 1) come in with a great deal of existing domain knowledge, 2) be willing and able to develop it, 3) or be willing to guess (or [throw science at the wall to see what sticks](https://www.youtube.com/watch?v=UM-wKQqBBnY)). Put another way: **PPLs require users to have already done most of the learning**. 

Ideally, we want to define exactly (and only) what we know and let the machine figure out everything else. We want to apply our domain knowledge where we have it &mdash; whether that is a fully specified parametric model for certain parts of the larger model, or a looser understanding of which variables are causally related without understanding their parametric form.

# Modeling faster by fitting slower

Because PPLs only address parametric uncertainty, users must fit many models to address structural and relational uncertainty through a manual, and often *ad hoc*, model comparison process external to the PPL. This means that users must fit many models, which means that model fitting is now the bottleneck to model building, which means that PPLs must fit models quickly. Fitting models quickly often requires brutal optimization which narrows scope. Optimization and generality are often [conflicting goals](https://en.wikipedia.org/wiki/No_free_lunch_in_search_and_optimization).

To improve fit speed, many PPLs use inference routines that

1. optimize for unimodal posteriors (don't handle complex real-world models well)
2. optimize for continuous variables (are slow on categorical variables)
3. use variational approximations that hurt uncertainty quantification and introduce approximation error

PPLs have optimized for speed of fit at the cost of speed of modeling. I argue that using more general inference/fitting routines that address all levels of uncertainty will make modeling faster by greatly reducing iteration. Using samplers that better support multi-modal posteriors and categorical variables will be slower, but will require less tweaking and transformation. Similarly, by using samplers that do inference of model structure and parametric forms, we further reduce the burden on the user.

# Fixing PPLs with two new operators

PPL code is littered with `~`. For example `x ~ Normal(m, s)` means `x` (whatever it is) is distributed according to a Normal distribution with mean `m` and standard deviation `s`. We have defined a parametric form (Normal) of the data, or variable, `x`. We can read `~` as "distributed as" or "simulated from" or "is a random variable of the form".

Using this notation we can define a simple linear regression:

```
data {
  x: [float]
  y: [float]
}

eps ~ Gamma(1, 1)
m ~ Normal(0, 1)
b ~ Normal(0, 1)
y ~ Normal(m * x + b , eps)
```

The `data` block defines our data, `x` and `y`, which are both vectors of floats. We define an errors, `eps`, which comes from a Gamma distribution. We define `m` and `b`, the slope and intercept of the line. The result is that `y = m * x + b` plus some Gaussian noise.

To add support for structural and relational uncertainty we can add two new operators. To address structural uncertainty, we can add a `~?` operator, which we can read as "maybe distributed as". Now to determine whether two variables are linearly related we can do

```
data {
  x: [float]
  y: [float]
}

eps ~ Gamma(1, 1)
m ~ Normal(0, 1)
b ~ Normal(0, 1)
y ~? Normal(m * x + b , eps)
```

We made one change on the last line. Now we are asking, "Is `y` distributed linearly according to `x`, and if so, what are the likely slope, intercept, and error?" 

There are number of ways we could handle this under the hood. A straightforward way, but one that doesn't scale well, is to treat `~?` like a component in a mixture model and do [hypothesis testing via a mixture model](https://arxiv.org/abs/1412.2044).

If we know `x` and `y` are related, but don't know whether it's through a line or any other easily defined shape, we can tackle relation uncertainty with the `?` distribution:

```
data {
  x: [float]
  y: [float]
}

y ~ ?(x)
```

Which says that `y` is distributed according to some distribution relating to `x`, but we don't know what. This is easily addressed with Bayesian nonparametrics.

And of course, we can just ask about everything: "Are `x` and `y` related, and if so, how?"

```
data {
  x: [float]
  y: [float]
}

y ~? ?(x)
```

The awesome thing is that models that admit structural and relational uncertainty are easier to define that models that only admit parametric uncertainty.

# Ephesus

So far this has all be very theoretical and hand-wavy. How do we build and scale such a system? This type of mixed inference underlies our (Redpoll's) *Ephesus* modeling framework. We're still in early days, but I will sporadically drop posts discussing Ephesus components as supplements to `arXiv`ed manuscripts. Stay tuned!
