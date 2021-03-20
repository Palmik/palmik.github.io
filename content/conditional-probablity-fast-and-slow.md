+++
title = "Conditional Probability, Fast and Slow"
date = 2021-03-20

[taxonomies]
categories = []
tags = ["Math", "Probability", "Thinking, Fast and Slow"]
+++

Now more than ever, we're bombarded with terms like "sensitivity" or "specificity" in relation to Covid-19 tests. It turns out that to understand these, you first need to understand conditional probability. 

Because common sense intuition can be often misleading, I will walk you through a conditional probablity problem about Blue and Green cabbies from the book [Thinking, Fast and Slow](https://en.wikipedia.org/wiki/Thinking,_Fast_and_Slow).

<!-- more -->
<hr/>

Few weeks ago a friend reached out to me to help them with understanding the solution to a particular question about probablity from the book [Thinking, Fast and Slow](https://en.wikipedia.org/wiki/Thinking,_Fast_and_Slow) by [Daniel Kahneman](https://en.wikipedia.org/wiki/Daniel_Kahneman). The question goes like this:

> A cab was involved ina hit-and-run accident at night. Two cab companies, the Green and the Blue, operate in the city. You are given the following data:
> 
> * 85% of the cabs in the city are Green and 15% are Blue.
> * A witness identified the cab as Blue. The court tested the reliability of the witness user the circumstance that existed on the night of the accident and concluded that the witness correctly identified each one of the two colors 80% of the time and failed 20% of the time.
> 
> What is the probability that the cab involved in the accident was Blue rather than Green?

## Notation 

Let's introduce some useful notation for different "events":

* `G` = green caused the accident 
* `B` = blue caused the accident 
* `WG` = `!WB` = witness testified green = witness did not testify blue
* `WB` = `!WB` = witness testified blue = witness did not testify green


## Understanding the problem statement

What does it mean that the court tested the reliabilty of the witness? Easy way to think about this is that the court performed 100 tests where they recreated the situation of the accident (like weather and light conditions), in 85 of those tests the cab was Blue and in 15 the cab was Green [1]. This allows the court to compute:
* `P(WB | B)` -- the probability that the witness says the cab was Blue *given that it was really Blue*. In other words, the probability that the witness is right if the culprit is Blue.
* `P(WB | G)` -- the probability that the witness says the cab was Blue *given that it was really Green*. In other words, the probability that the witness is wrong if the culprit is Green.
* Similary for `P(WG | G)` and `P(WG | B)`.

Based on the problem statement:

* `P(WB | B)` = `P(WG | G)` = 0.8
* `P(WB | G)` = `P(WG | B)` = 0.2

We will also need to make an assumption [2], which is that drivers of the two companies are equally likely to commit hit-and-run accidents.

Based on the problem statement, since 85% of cabs are Green and 15% of them are Blue and each cab is equally likely to cause an accident:

* `P(B)` = 0.85
* `P(G)` = 0.15


Using our notation, the task is to compute `P(B | WB)` -- the probability that the cab involved in the accident was Blue **given that the witness identified the cab as Blue.**

## Solving the problem

We are to find `P(B | WB)`. We are given:

* `P(G) = 0.85`
* `P(B) = 0.15`
* `P(WB | B) = P(WG | G) = 0.8`
* `P(WB | G) = P(WG | B) = 0.2`

To solve this, we will refer to [the definition of conditional probability](https://en.wikipedia.org/wiki/Conditional_probability):

```
P(A | B) * P(B) = P(A and B)
```

Applying this to our problem:

1. `P(B | WB) * P(WB) = P(B and WB)` therefore `P(B | WB) = P(B and WB) / P(WB)`
2. `P(WB | B) * P(B) = P(B and WB)`

Based on (1), to compute `P(B | WB)` we need to know `P(B and WB)` and `P(WB)`.

To compute `P(B and WB)` we use (2). We already know `P(WB | B)` and `P(B)`, so we simply substitute: `P(WB | B) * P(B) = 0.8 * 0.15`.

To compute `P(WB)` we need to consider when `WB` can occur (i.e. the witness says that the car is Blue)? Either:

* The car is Blue and the witness is right: `P(B) * P(WB | B)` = `0.15 * 0.8`
* The car is Green and the witness is wrong: `P(B) * P(WB | G)` = `0.15 * 0.2`

Given that these two options are exclusive, we can sum up these two probabilities to obtain `P(WB)`.

Now that we know both `P(B and WB)` and `P(WB)`, we simply substitue into (2):

```math
P(B | WB) = P(B and WB) / P(WB)
          = (0.8 * 0.15) / P(WB)
          = (0.8 * 0.15) / (0.15 * 0.8 + 0.15 * 0.2)
          = 0.41379...
```

And voila! This is the answer.

## Bonus: Using the "tabular" method

|                         | Blue criminal                                 | Green criminal                                 |
|-------------------------|-----------------------------------------------|------------------------------------------------|
| Witness testified Blue  | `P(B and WB) =`<br/><code>P(B) * (WB &vert; B) =</code><br/>`0.15 * 0.8`  | `P(G and WB) =`<br/><code>P(G) * P(WB &vert; G) =</code><br/>`0.85 * 0.2` |
| Witness testified Green | `P(B and WG) =`<br/><code>P(B) * P(WG &vert; B) =</code><br/>`0.15 * 0.2` | `P(G and WG) =`<br/><code>P(G) * P(WG &vert; G) =</code><br/>`0.85 * 0.8` |


* All of these 4 different events `P(B and WB)`, `P(B and WG)`, etc. are clearly disjoint
* The sum of the “testified blue” row is equal to P(WB)
* The sum of the “testified green” row is equal to P(WG)
* The sum of the “blue criminal” column is equal to P(B)
* The sum of the “green criminal” column is equal to P(G)
* This table allows you to easily solve any of the conditional probabilities involving G or B and WG or WB

## Bonus: Using the Bayes' theorem


The [Bayes' theorem](https://en.wikipedia.org/wiki/Bayes%27_theorem) states:

```math
P(A | B) * P(B) = P(B | A) * P(A)
```

Alternatively, if `P(B)` is not zero, is:

```math
P(A | B) = (P(B | A) * P(A)) / P(B)
```

Using this, computing `P(B | WB)` we need `P(WB | B)`, `P(WB)` and `P(B)`, which we have already computed above.

## Bonus: Covid-19 tests 

In the context of vaccines, people often talk about *sensitivity* and *specificity*. What are these and how do they relate to conditional probability?

### Sensitivity

The probability that the test comes out as positive if the tested subject has the dissease. `P(tests positive | has disease)`.

### Specificity 

The probability that the test comes out as negative if the tested subject does not have the disease: `P(tests negative | does not have disease)`.

### Sample problem 

Assume that the coronavirus test has specificity of 0.85 and sensitivity of 0.95 and that 10% of the population has coronavirus at the moment.

What is the probability that you have coronavirus if:
* You tested positve
* You tested negative

## Footnotes

* [1] In practice, the color of the cab for each test should be selected at random. Try to think about why.
* [2] Ideally this would be explicitly specified in the problem statement.
