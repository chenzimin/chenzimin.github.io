---
layout: post
title: Explaining automatic program repair using concrete example
visible: 1
---

When I started working with automatic program repair (Feb 2018). My first question was: "How the heck can we automatically repair bugs?". Much like other technologies, once you understand what is going on under the hood. You realize that it is nothing magical, we can solve the problem by simplifying the problem with empirically verified assumptions. So this blog post is intended for people that are interesting in the question, "How the heck can we automatically repair bugs?". This by no means the state-of-the-art solution in the field, rather than just a very simple approach to illustrate the idea.


What is automatic program repair? Martin Monperrus defined it in his paper ["Automatic Software Repair: a Bibliography"](https://arxiv.org/abs/1807.00515) as:

> Automatic repair is the transformation of an unacceptable behavior of a program execution into an acceptable on according to a specification.

So how do automatic program repair transform the program? What is unacceptable/acceptable behavior? How do we define the specification? I will try to give answers to these questions in this blog post by guiding you through a concrete example. Note that I will only discuss **automatic program repair at source code level**, and I will also trade **accuracy for simplicity** when explaining concepts in automatic program repair.

----

## Automatic Program Repair

Currently, nearly all automatic program repair tools follow three steps to repair the program:
1. Fault localization
2. Patch generation
3. Patch validation

The first step, fault localization, localize the cause of the unacceptable behavior in the program. The patch generation step then will transform the program by modifying the source code at the specified position. And in the end, the patch validation step will validate the transformed program according to a specification.

Until now, all descriptions of automatic program repair are too abstract. So let's discuss how each step can be achieved in reality. I will be using the following toy example throughout this blog to help me explain each step:

A function that is supposed to return the sum of two integers:
```python
def sum(a, b):
  if(a >= 10):
    return a
  else:
    return a+b
```

Two test cases, "`test_1`" respective "`test_2`", for the "`sum`" function:
```python
assert(sum(1, 2) == 3)
```
```python
assert(sum(10, 20) == 30)
```

### Fault localization
The fault localization step tries to localize the cause of the unacceptable behavior in the program. But how do we know that some behavior is unacceptable? In most of the times, we use the **test suite to define acceptable behavior**. Meaning that if some test cases failed, we have an unacceptable behavior. In other words, there are one/more bug(s) in the program. As a result, we say that the test suite is the program specification. Automatic program repair that uses test suite as tge specification is called **test suite based automatic program repair**. Test suite based automatic program repair require at least one failing and one passing test case. We need failing test cases to expose the bug, and passing test cases acts as regression test, to make sure we didn't break anything.

Now we have decided that test suite is the program specification, how do we localize the bug? Here I will present **spectrum-based fault localization**, the dominant technique to localize the fault. It is built on top of a simple idea, that code components (e.g. statements) executed by falling test cases have a higher probability to be the cause of the bug.

Let's use "`sum`", "`test_1`" and "`test_2`" as example. We will record the result of each test cases and log statements that are executed by each test cases. First, "`test_1`" is executed:

```python
def sum(a, b):
  if(a >= 10):     # test_1
    return a
  else:            # test_1
    return a+b     # test_1
```

"`test_1`" passed and statements that are executed by "`test_1`" are marked by in-line comment "`# test_1`". Now let's execute "`test_2`" and follow the same procedure:

```python
def sum(a, b):
  if(a >= 10):     # test_1 test_2
    return a       #        test_2
  else:            # test_1
    return a+b     # test_1
```

"`test_2`" failed and statements that are executed by "`test_2`" are marked by in-line comment "`# test_2`".

In this toy example, we can clearly see that "`return a`" is only executed by the failing test case. Therefore the bug is most likely in "`return a`". In reality, we use a formula to calculate a **suspicious score**, between 0 and 1, for each statement. 0 means it is unlikely to be faulty and 1 mean it is likely to be faulty, the formula that we will uses is:

![Tarantula formula]({{ site.baseurl }}/images/2019-3-14/tarantula_formula.png "Tarantula formula")

Where "`failed(s)`" is number of times that statement "`s`" is executed by failing test cases, "`passed(s)`" is the number of times that statement "`s`" is executed by passing test cases, "`totalpassed`" is the number of passing test cases and "`totalfailed`" is the number of failing test cases.

If we apply the formula on our toy example, we get the following result:
```python
def sum(a, b):
  if(a >= 10):     # suspicious(s) = (1/1) / ((1/1) + (1/1)) = 1/2
    return a       # suspicious(s) = (1/1) / ((1/1) + (0/1)) = 1
  else:            # suspicious(s) = (1/1) / ((1/1) + (1/1)) = 1/2
    return a+b     # suspicious(s) = (1/1) / ((1/1) + (1/1)) = 1/2
```

The formula is called [Tarantula](https://dl.acm.org/citation.cfm?id=1101949) and it is one of most used formula to calculate the suspicious score. Note that higher suspicious score does not necessarily mean anything more than that the statements were executed more times by the failing test cases. Therefore we can trust it and sort the statements based on the suspicious value, or do a weighted random selection, or just random selection.

### Patch generation

In the patch generation step, the source code will be modified and patches are generated. Most of the research within automatic program repair is focusing on the patch generation step, since here is where the magic is happening. I will only present one very simple approach here, I may discuss more approaches in the future.

 Let's us say that we trust the fault localization step, meaning that we sort the statements based on its suspicious value and process them sequentially. Our first candidate in the list is "`return a`" with a suspicious value of 1. How do we modify it? Should we change the keyword "`return`"? Should we replace "`a`" with "`b`"? Should we add a statement before it? Or should we just delete this statement? Clearly, there are infinitely many ways to change this statement, and it is only one of many suspicious statements. Therefore we must reduce the search space in a clever way.

Here we make one assumption to help us to reduce the search space, that is **the correct bug fix may already exist somewhere else in the same file**. This assumption is often called [**redundancy assumption**](https://arxiv.org/abs/1403.6322), or [**plastic surgery assumption**](https://dl.acm.org/citation.cfm?id=2635898). What we assume is that we can reuse code component in the same file to fix existing bugs. The intuition is that the same function can be implemented several times, and we can use the code component from the correct implementation to fix the buggy version. We can even expand its scope to reuse code component from the same package, program or code base.

In the same time, I will also limit the operation that can be done on source component to only consider **add**, **remove** and **replace** operation. Meaning that we can only add something after the current code component, or remove the current code component, or replace the current code component with other code components. These operations are also called **mutation operation**.

| Mutation operation | Descriptions |
| ------------------ | ------------ |
| Add | Add code component after the current code component |
| Remove | Remove the current code component |
| Replace | Replace the current code component with other code component |

Now we are ready to modify the source code. We will define code component as expression, and we will reuse all expressions in the same file. Therefore all our expressions, or **repair ingredients**, in our toy example are:
```python
a
b
10
a+b
a >= 10
```

Our objective is to modify the only expression "`a`" in "`return a`" with add, remove or replace mutation operation. Since we are working with expression, we can skip the add operation (with statement it makes more sense), and the remove operation does not need any repair ingredients, since we just remove the current expression. The **search space** (the space where we search for the correct patch) of our repair algorithm for "`return a`" is therefore 5 + 1 = 6 patches. We have 5 repair ingredients where each ingredient can replace "`a`", and we have 1 more patch where "`a`" is simply removed. In short, we will generate the following patches:

patch_1
```diff
def sum(a, b):
  if(a >= 10):
-    return a
+    return a
  else:
    return a+b
```

patch_2
```diff
def sum(a, b):
  if(a >= 10):
-    return a
+    return b
  else:
    return a+b
```

patch_3
```diff
def sum(a, b):
  if(a >= 10):
-    return a
+    return 10
  else:
    return a+b
```

patch_4
```diff
def sum(a, b):
  if(a >= 10):
-    return a
+    return a+b
  else:
    return a+b
```

patch_5
```diff
def sum(a, b):
  if(a >= 10):
-    return a
+    return a >= 10
  else:
    return a+b
```

patch_6
```diff
def sum(a, b):
  if(a >= 10):
-    return a
+    return
  else:
    return a+b
```

Next, we will take the next candidate in the list, which is "`if(a >= 10)`" (breaks tie with line number). Again we have the same repair ingredients and mutation operators. And we will generate more patches. The same procedure applies for all candidates return by the fault localization step.

That is our patch generation step! If we included more repair ingredients by for example considering all expression at program scope, our search space will be much larger. We can also expand the search space by considering more mutation operations such as combining expression with AND, OR, addition, module etc. This patch generation technique is often called **mutation-based program repair**.

### Patch validation

In the patch validation step, we want to validate the patches generated by the patch generation step according to a specification. Since our program specification is our test suite, we will execute the test suite on these new transformed programs. This step is very straightforward and doesn't need much explanation. At the end of this step, only "`patch_4`" passed all test cases, and our program repair tool will output this patch as bug fix.

----

## Other variations
Here I will present some variations in automatic program repair, it is by no means an exhaustive list.

### Program specification (patch validation)
[Formal specification](https://en.wikipedia.org/wiki/Formal_specification) is another way to define the program specification. And then [formal verification](https://en.wikipedia.org/wiki/Formal_verification) can be used to validate the patched program.


We could also use compiler as program specification, then the types of error that can be repaired are compiler errors. And the patch validation step will be to simply compile the program.

### Fault localization
We have less choice with fault localization, we could use other formula such as [Ochiai](https://ieeexplore.ieee.org/document/4344104), which is defined as:

![Ochiai formula]({{ site.baseurl }}/images/2019-3-14/Ochiai_formula.png "Ochiaiformula")

But there are approach that replace values in statements executed by falling test cases ([Fault Localization Using Value Replacement](https://dl.acm.org/citation.cfm?id=1390652)). And we can even build graph and use casual inference to reason about the location ([Causal inference for statistical fault localization](https://dl.acm.org/citation.cfm?id=1831717)).

If we use compiler as program specification, the error message can be used as fault localization. For example:

```bash
Exception in thread "main" java.lang.ArithmeticException: / by zero
	at ArithmeticException_Demo.main(File.java:7)
```

We have a indication that the bug is located in line 7.

### Patch generation
Apart from automatic program repair based on the redundancy assumption. We have another big family based on symbolic execution or constraint solving. The idea is that we derived constraints that the correct program must satisfy, and we use constraint solver techniques (SAT-solver) to discover the correct patch. Probably the best known paper in this family is [Angelix: Scalable Multiline Program Patch Synthesis via Symbolic Analysis](https://dl.acm.org/citation.cfm?id=2884807)

In recent years, deep learning has also made its way into automatic program repair. And we have started to see more approaches that use recurrent neural network (encoder-decoder model) from neural machine translation for automatic program repair. Just to name a few:
- [DeepFix: Fixing Common C Language Errors by Deep Learning](https://www.aaai.org/ocs/index.php/AAAI/AAAI17/paper/viewPaper/14603)
- [sk p: A Neural Program Corrector for MOOCs](https://arxiv.org/abs/1607.02902)
- [SequenceR: Sequence-to-Sequence Learning for End-to-End Program Repair](https://arxiv.org/abs/1901.01808) (Disclaimer: I am one of the authors)
- [An empirical investigation into learning bug-fixing patches in the wild via neural machine translation](https://arxiv.org/abs/1812.08693)
- [Learning to Generate Corrective Patches using Neural Machine Translation](https://arxiv.org/abs/1812.07170)

----

## Last words

 If you prefer watching video instead of reading, I recommend this presentation from Claire Le Geous (["Automatic Patch Generation" by Claire Le Goues](https://www.youtube.com/watch?v=sRkfMe0_5cA)).

 By now you should have a pretty clear overview of how automatic program works. But remember that this is only the one toy example and it is only used to demonstrate the concepts in automatic program repair. We haven't touch more complex problem such as large search space, problem with redundancy assumption, overfitting patch and multi-location bug repair, which I may discuss in the future. Hopes that this blog helped you and feel free to contact me if you have any questions.
