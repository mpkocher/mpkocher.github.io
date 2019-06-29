---
title: "Series: Functional Programming Techniques In Python"
date: 2019-03-01 00:35:52
tags: 
    - python 
    - functional
    - FPT
    - series
---

This is a 4 Part Series that explores functional centric design style and patterns in Python.

{% gist 9896022 %}

[Part 1](https://mpkocher.github.io/2019/01/30/Functional-Python-Part-1/) ([notebook](https://gist.github.com/mpkocher/d1948e54c7863b1548ec4639df44b954)) starts with different mechanisms of how to define functions in Python and quickly moves to using [closures](https://en.wikipedia.org/wiki/Closure_(computer_programming)) and `functools.partial`. We then move on to adding `functools.reduce` and  composition with [compose](https://gist.github.com/mpkocher/9896022) to our toolbox. Finally, we conclude with adding lazy `map` and `filter` to our toolbox and create a data pipeline that takes a stream of records to compute common statics using a max heap as the reducer. 

In [Part 2](https://mpkocher.github.io/2019/02/02/Functional-Python-Part-2/) ([notebook](https://gist.github.com/mpkocher/517d9e72536346de505bff47199a9b24)), we explore building a REST client using functional-centric design style. Using an iterative approach, we build up a REST client using small functions leveraging clsoures and passing functions as first class citizens to methods. To conclude, we wrap the API and expose the REST client via a simple Python class.

[Part 3](https://mpkocher.github.io/2019/02/28/Functional-Python-Part-3/) ([notebook](https://gist.github.com/mpkocher/4e261b38d8b1c76a7f6dd8e9a95a4873)) follows similarly to spirit to Part 2. We build a commandline interface leveraging `argparse` from the Python standard library. Sometimes libraries such as argparse can have rough edges or friction points in the API that introduce duplication or design issues. Part 3 focuses on iterative building up an expressive commandline interface to a subparser commandline tool using closures and functions to smooth out the rough edges of `argparse.`. There's also examples of using a Strategy-ish pattern with type annotated functions to enable configuring logging as well as custom error handling. 

[Part 4](https://mpkocher.github.io/2019/03/01/Functional-Python-Part-4/) ([notebook](https://gist.github.com/mpkocher/88f01a6237df44a8a01b51fb58cbb544)) concludes with some gotchas with regards to scope in closures, a brief example of decorators and a few suggestions for leverging function-centric designs in your APIs or applications. 

If you're a OO wizard, a Data Scientist/Analysist, or a backend dev, this series can be useful to add another design approach in your toolbelt to designing APIs or programs. 

Best to you and your Python'ing!

