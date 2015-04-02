---
layout: post
category : lessons
tagline: collection comprehensions and other stuff
---
{% include JB/setup %}
##Stuff that pleasantly surprised me about python coming from C    

Note: the examples here are from ipython. ```In [<line no>]``` means input and ```Out [<line no>]``` is output.

###Conditional Expressions

C style tenary operator(```condition?value_if_true : value_if_false```), but in English. 
The conditional expression returns one value if a condition is true or else it returns something else.


some arbitrary examples.    
{% highlight python %} 

    In [10]: 5 if 10%2 else 10
    Out[10]: 10

    In [11]: 5 if 11%2 else 10
    Out[11]: 5

    In [12]: vowels='aeiou'

    In [13]: 'vowel' if 'x' in vowels else 'consonant'
    Out[13]: 'consonant'

    In [14]: 'vowel' if 'e' in vowels else 'consonant'
    Out[14]: 'vowel'
    
{% endhighlight %}


### List Comprehensions    
This is a way to construct lists on the fly.
Say you want to make a list that contains the squares of first 10 natural numbers

###What you know:  

    In [1]: squares=[]

    In [2]: for number in range(1,11):
       ...:     squares.append(number*number)
       ...:

    In [3]: squares
    Out[3]: [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

###List comprehension way     

    In [4]: squares=[number*number for number in range(1,11)]

    In [5]: squares
    Out[5]: [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

It takes an if statement too!    

    In [6]: squares_of_even=[num* num for num in range(1,11) if not num%2]

    In [7]: squares_of_even
    Out[7]: [4, 16, 36, 64, 100]

You can construct a list of anything with this.    
list of tuples of numbers and their squares for first 10 natural numbers:

    In [8]: tuples=[(num,num*num)for num in range(1,11)]

    In [9]: tuples
    Out[9]:
    [(1, 1),
     (2, 4),
     (3, 9),
     (4, 16),
     (5, 25),
     (6, 36),
     (7, 49),
     (8, 64),
     (9, 81),
     (10, 100)]

###Anatomy of list comprehensions, and optional bling
![listcomp.png](http://i.imgur.com/DhooQ7E.png)

### More comprehensions - dicts, sets and generator expressions.    
**Dictionary Comprehension** : ```{<key>:<value> for <item> in <iterable> }```     
This lets you construct dictionaries.
For example, to find how many times each character in a string repeats, one can do this:

    In [2]: s='This is a string with many letters repeating.'

    In [3]: freq_table={letter:s.count(letter) for letter in s }

    In [4]: freq_table
    Out[4]:
    {' ': 7,
     '.': 1,
     'T': 1,
     'a': 3,
     'e': 4,
     'g': 2,
     'h': 2,
     'i': 5,
     'l': 1,
     'm': 1,
     'n': 3,
     'p': 1,
     'r': 3,
     's': 4,
     't': 5,
     'w': 1,
     'y': 1}

**Set comprehensions**: ```{<member> for <item> in <iterable>}```     
This lets you create a set. A set is a data structure that do not allow repeats(only unique values) and are optimized for fast membership tests(which of the elements of SET A are also members of SET B?) and other related computations.     
Say, if we have a list of strings ```['parrot','dead','argument','spam','eggs','spam']``` and we need to find out the unique lengths. ie there are strings of length 8,4 and 6 in the list and no other length strings.

    In [6]: l=['parrot','dead','argument','spam','eggs','spam']

    In [7]: {len(word) for word in l}
    Out[7]: set([8, 4, 6])

**Generator expressions**: ```(<item to yield> for <item> in <iterable>)```      
A generator expression is a lazy list comprehension. What this means is that the entire list is not stored in memory and when you iterate through it, the values are calculated just when they are needed. the difference between list comprehensions and generator expressions is also the difference between ```range()``` and ```xrange()``` from the standard library. A lot of the times, you do not need every item in a list at the same time, and using generator expressions at these times can save quite a bit of memory. For example , if you need the sum of the squares of the first 100 natural numbers, you do not need to store all the squares at any point. You only need an accumulator variable to store the total and the current square in the iteration.

    In [15]: generator=(num*num for num in xrange(100))

    In [16]: sum(generator)
    Out[16]: 328350

Note that ```sum()``` does not convert the argument into a list and works lazily.

###splat(```*```) and double splat(```**```)     
**splat**:     
While passing arguments in a function call, the splat ```*``` operator is a signal to unpack.
Say you have a function that needs 3 arguments ```def foo(arg1,arg2,arg3)```, but while calling it, you have the arguments to send in a list, you can do the following   

    In [33]: def foo(arg1,arg2,arg3): # accepts 3 arguments and return them as a tuple
       ....:     return arg1,arg2,arg3
       ....:

    In [34]: args=[12,'a string',23.7]

    In [35]: foo(*args) # send a single list as argument with a splat, which is a signal to unpack
    Out[35]: (12, 'a string', 23.7) 

It works the other way around too, and allows you to send an arbitary number of arguments to a function.

    In [36]: def foo(*args): #pack all the incoming arguments into a tuple called args
       ....:     return args 
       ....:

    In [37]: foo(1,2,3,'a string') # sending multiple arguments
    Out[37]: (1, 2, 3, 'a string') # arguments get returned fine

You are encouraged to experiment with this stuff till you get a solid idea.

**double splat**:     
the double splat(```**```) is similar to the splat, just that it allows keyword arguments. You can arbitrarily pass keyword arguments and the function receives it as a dictionary.
    
    In [39]: def foo(**kwargs):
       ....:     return kwargs
       ....:

    In [40]: foo(name='aritra',lastname='das')
    Out[40]: {'lastname': 'das', 'name': 'aritra'}

other way round:

    In [44]: def foo(name,lastname):
       ....:     return name,lastname
       ....:

    In [45]: args={'name':'aritra','lastname':'das'}

    In [46]: foo(**args) #send a dictionary with unpack signal
    Out[46]: ('aritra', 'das')


###[zip](https://docs.python.org/2/library/functions.html#zip)
Docstring:
```zip(seq1 [, seq2 [...]]) -> [(seq1[0], seq2[0] ...), (...)]```

Return a list of tuples, where each tuple contains the i-th element
from each of the argument sequences.  The returned list is truncated
in length to the length of the shortest argument sequence.

    In [48]: zip([1,2,3],['a','b','c'])
    Out[48]: [(1, 'a'), (2, 'b'), (3, 'c')]

The opposite operation of this, where you have a list of tuples and you want a tuple of lists where each list 
    
    In [55]: zipped=zip([1,2,3],['a','b','c'])

    In [56]: unzipped=zip(*zipped)

    In [57]: unzipped
    Out[57]: [(1, 2, 3), ('a', 'b', 'c')]

The combination of all this lets you do a lot of cool stuff.

###lambdas    
A lambda is simply a function maker which lets you make oneline functions that return one expression. This expression can be conditional expressions or list comprehensions or whatever.

    In [58]: foo=lambda x: sum(xrange(x))

    In [59]: foo(10)
    Out[59]: 45

Is equivalent to

    In [60]: def foo(x):
       ....:     return sum(xrange(x))
       ....:

    In [61]: foo(10)
    Out[61]: 45

you can also call it directly without assigning to a variable.

    In [63]: (lambda x: sum(xrange(x)))(10)
    Out[63]: 45


###random example
Suppose you have a bunch of 2D coordinates as a list of tuples ```[(x1,y1),(x2,y2),(x3,y3)....]``` and you want to find out the minimum x, maximum x, minimum y and maximum y.

    In [75]: coords=[(1,3),(2,5),(6,4),(2,9),(0,0)]

    In [76]: (maxx,minx),(maxy,miny)=map(lambda p:(max(p),min(p)), zip(*coords))

    In [77]: print maxx,minx,maxy,miny
    6 0 9 0

```zip(*coords)``` unzips coords into ```[(x1,x2,x3,x4...),(y1,y2,y3,y4...)]```     
```map(<function or lambda>,iterable)``` returns a list of new values by applying the function on each value of the iterable
```lambda p:(max(p),min(p))``` takes an iterable and returns a tuple of the maximum and the minimum item in the list    
Automatic unpacking causes the output of map to be assigned properly to the tuple of tuples in the left hand side.


    In [87]: l=[('A1','B1'),('A2','B2')]

    In [88]: (a1,b1),(a2,b2)=l

    In [89]: a1,b1,a2,b2
    Out[89]: ('A1', 'B1', 'A2', 'B2')


