---
title: "Floating-point representation"
date: 2021-10-17
updated_at: 2025-02-18
description: "What do Roman numerals, Tally marks, Decimal and Binary number systems have in common? They are all used to describe Quantity or the number of things, each with its unique style motivated by a particular usecase. "
draft: false
meta_image: floating_bits_images/realmap.png
hideToc: true
enableToc: false
math: true
libraries:
    - katex
enableTocContent: true
tags:
    - Science
    - float
    - Number system
    - Computer Science
    - Computational Physics
categories:
    - engineering
---

What do Roman numerals, Tally marks, Decimal and Binary number systems have in common? They are all used to describe **Quantity** or the **number of things**, each with its unique style motivated by a particular usecase.
Tally Marks exist from the primitive ages and were probably made to keep track of the stock of apples left and Roman numerals were invented to write relatively large quantities probably to ledger cash, it would not be really fun writing hundred or even a thousand in tally marks.

## Number System

Decimal system is the one with which you grew up with, atleast I did, which came to be after humanity invented `0` which comes really handy while simplifying fractions, it is also called the Base 10 system because we have 10 Basic **states** or **configurations** or **symbols** to work with, namely `0,1,2,3,4,5,6,7,8,9`.
The decimal number system has two main attributes indices(place), base.

{{< blockquote >}}
In the Decimal system, a number is represented by a list of digits from 0 to 9, where each digit represents the coefficient for a power of 10 determined by it's place.
Mathematically, if N is any number, its decimal equivalent $$x_b x_{b-1} \cdots x_1 x_0 . x_{-1} \cdots x_{a+1} x_{a}$$ has the relation,
$$N = \sum_{i=a}^b{x_i 10^i}$$
where $x_i = {0,1,\dots 9}$
{{< /blockquote >}}

look at the following examples,
$$114.5 = 1\cdot10^2 + 1\cdot10^1 + 4\cdot 10^0 + 5\cdot10^{-1}$$
$$11.64 = 1\cdot10^1 + 1\cdot 10^0 + 6\cdot10^{-1} + 4\cdot10^{-2}$$

Now, the binary (meaning "two") number system is defined similarly but with a base of 2,

{{< blockquote >}}
In the Binary system, a number is represented by 0 or 1, where each digit represents the coefficient for a power of 2 for it's place.
Mathematically, if N is a any number, its binary equivalent $$x_b x_{b-1} \cdots x_1 x_0 . x_{-1} \cdots x_{a+1} x_{a}$$ has the relation,
$$ N = \sum\_{i=a}^b{x_i 2^i} $$
where $x_i = {0,1}$
{{< /blockquote >}}

$$ (1001)\_2 = 1\cdot2^3 + 0\cdot 2^2 + 0\cdot2^1 + 1\cdot2^0 = 9 $$
$$ (1001.01)\_2 = 1\cdot2^3 + 0\cdot 2^2 + 0\cdot2^1 + 1\cdot2^0 + 0\cdot2^{-1} + 1\cdot2^{-2} = 9.25 $$

The Binary number system readily finds its way in the most useful Binary device (No pun intended) humans ever created, _**the transistor**_ . A transistor can either be **ON** (**if it outputs current**) or **OFF** otherwise. Computers are built on these small devices and therfore to work with such a binary system, the binary number system seemed as the perfect choice until.... we came short on the abilities of our memory devices.

A bit alone can attain only two states (ON or OFF) and therefore it can only represent two possible values based on its state, conventionally `0` and `1`, but a list of $n$ bits can collectively exist in $2^n$ unique combinations or states. This allows us to store $2^n$ different numbers in memory, each mapped directly from a unique combination of the list of stored bits.

{{< notice info >}}

- Information is stored in computer memory as lists of bits of fixed length (8-bit, 64-bit), each list is known as a **register**, fundamentally storing values in variables comes down to filling these **registers**.

- It is helpful to view number systems as a one-one linear mapping from the set of all possible states to the Real Number line. The idea being that if you have such a mapping at your disposal you can figure out the only unique real number corresponding to a given array of bits, which is what a computer needs to do in order to store numbers.
  {{< /notice >}}

## Fixed-point numbers

In a fixed-point representation, we fix our imaginary decimal point somewhere in middle of our list of bits and store numbers accordingly.

Let's first look at the idea in base 10, given that we allocate last 3 places for the difits after the decimal, we store the decimal number $14.78$ as $14.78 \times 10^3 = 14780$ and similarly

| Decimal | Fixed point representation |
| ------- | -------------------------- |
| 503.445 | 503445                     |
| 2.34    | 2340                       |
| 2300.3  | 2300300                    |
| 23.3305 | 23330                      |

any place after the 3rd place after the decimal is dropped.

For example, consider a 8-bit register where the last 3 bits are allotted to the value after the decimal point.
$ {(18.625 \times 2^{3} )}\_{10} = (149)\_{10}= (10010101)\_2 $

Here the number $18.625$ is stored in memory as

<div class="image_container !bg-primary bg-opacity-25">

![Floating point](./drawing.png)

</div>

<div>

|  Binary  | Internal Fixed point representation |
| :------: | :---------------------------------: |
| 101.110  |               101110                |
|  101.11  |               101110                |
|  101.10  |               101100                |
| 101.1101 |               101110                |

</div>

Although it is easier to perform arithemetic with fixed point numbers, there are numerous occassions when you would want to deal with numbers of very small magnitude in the order of $10^{-12}$ and to be able to multiply and divide them by numbers of relatively large magnitude in the order of $10^{12}$ ; Now, If you wish to use fixed-point representation you would require roughly 80+ bits which might not seem alot in the modern age but 30 years ago they were very expensive, this forced computer scientists to look for better ways of storing decimal numbers.

{{< notice info >}}
Negative fixed point numbers are generally stored here by alloting half of all possible states($2^n$) to negative numbers, they are obtained by taking the two's complement, that is, flipping all the bits and adding one to the obtained binary,
{{< /notice >}}

## There is not enough memory!!

Information that you store anywhere takes up space, quite literally, made available to you by your storage device. Allocating 64-bit to a variable means that you are reserving $64$ bits for the value of that variable in the memory. You can represent $2^{64}$ or `18,446,744,073,709,551,616` numbers at maximum with 64 bits. If you decide to represent numbers using fixed-point representaion you will not be able to get a large range and high precision both, you have to trade one for another,
since one might have to deal with:-

- Very large numbers like the speed of light $2.99,792,458 \times 10^{8}$ in SI units

- Very small numbers like the planck's constant $6.62607004 \times 10^{-34}$

- Boring numbers like $9.86$

If your number system is a one-one maping from the set of all combinations/states of 64-bits to the co-domain of real numbers, this is a good time to ask which all real numbers do you actually want to map to?

## Which all Numbers you might actually need?

If I gave you a million of your favourite fruits, will you mind if you lost 20 of them?

Since a $1 \times 10^6$ fruits minus 20 or even a 200 is still a million fruits, roughly speaking, you may not want to keep track of small differences when dealing with comparitively large numbers.

Similarly, when dealing with small quantities, say 5.89kg of fruits, every kilogram of fruit matters to you and so you can't probably afford to lose the count of even 0.1kg.

To put it simply, error is best recognized when it is compared to the magnitude of the quantity in which error is being noticed. Such error is called **relative error**, reducing absolute error should always be our priority but for good approximations we try to keep the relative error as low as possible.

On the same train of thought, notice that you must have more configurations in your number system reprensenting small real numbers than larger ones. This idea was at the core of all numerical work we did and led to the creation of several different **floating point** representations for binary,

{{< blockquote >}}
so many infact that it became difficult for programmers to make design software with all these different floating point hardware units floating around.
including the **IEEE's standard for floating-point arithmetic, the IEEE 754** which is a number system that became immesenly popular among computers after its establishment in 1985.
{{< /blockquote >}}

## Floating-point numbers (floats)

Aha !! we already have an representation for to deal with scientific quantities,
It's called the scientific notation and goes something like this,

{{< blockquote >}}
All decimal numbers $N$ can be written in the form,
$$ N = m \times 10^n $$

- where $n$ is called the **exponent** and helps "float" the decimal point around,
- $m$ is called the **significand** which is a nonzero decimal number such that $0<|m|<10$ and it simply ensures that each decimal number has only one scientific notation(one-one mapping), a notation with this feature is called a **normalized notation**.
  {{< /blockquote >}}

As we pointed out in our discussion of scientific notation for decimal numbers, the decimal point is no longer fixed but its position is determined by another integer called the **exponent**. Let's discuss more about floating point representation in binary by discussing the **IEEE 754** standard for floating point arithemetic followed while designing all modern computers to handle float numbers.

### IEEE 754

In 1985, **[Institute of Electrical and Electronics Engineers](https://www.ieee.org/)** or IEEE came up with a set of standard protocols for storing and performing arithemetic operations involving the class of floating-point numbers, these standards stand by the name **IEEE 754** and was established to ease the burden on computer programmers.(since in the early days various different number systems were implemented by hardware manufacturers.)

Analogous to the scientific notation,

{{< blockquote >}}
Any IEEE 754 standard float gets stored in accordance with this notation, binary number B can be represented as,
$$ B = (-1)^S (1.M)\times 2^{E-E_b} $$

- where S is called the **sign bit** and determines the sign of $B$, we no longer use 2's complement to find the additive inverse in this system, instead we just flip the sign bit.
- recall from earlier that (1.M) is nothing but the significand, since normalization demands that the leading digit is nonzero, we end up fixing a 1 there and therfore it need not be stored. If $(M)\_{2}$ is called the **mantissa** and gets read as, \\[ (M)\_{10} = \sum_{i=-c}^{-1} m\_i 2^i \\]
  where $c$ is the maximum number of bits $m_i$ allocated to M.
- The **exponent** $(E-E_b)_{10}$ does the job of moving the binary point of the significand around just like in the scientific notation. Here biased exponent $(E)_2$ gets stored in the memory and when being retrieved the bias $(E_b)_2$ is subracted from it to get the value of exponent.
  {{< /blockquote >}}

IEEE standards are classified on the bases of number of bits that are allocated to store the float.
| IEEE standard | No. of bits in memory |
|---------------| :--------------------- |
| half-precision or float16 | 1 sign bit + 5 bits exponent + 10 bits mantissa = 16 |
| single-precision or float32 | 1 sign bit + 8 bits exponent + 23 bits mantissa = 32 |
| double-precision or float64 | 1 sign bit + 11 bits exponent + 52 bits mantissa = 64 |

One more thing, some values of the biased exponent is reserved for special numbers like inf, 0 and nan as shown in the table down below. Also,
when E is $(0\dots0)_2$ the leading digit of significand is flipped from 1 to 0. Such a representation is called **subnormal** and IEEE 754 uses them when the value to be returned is smaller than the smallest possible normal float value (this situation is referred to as **underflow**).
On the other hand when the value to be returned is larger than the largest possible normal float value, inf is returned in accordance to the sign bit.

<div class="overflow-auto">

| E             | M(base 10) | S   | Value                   |
| ------------- | ---------- | --- | ----------------------- |
| $(1\dots1)_2$ | 0          | 0   | inf (positive infinity) |
| $(1\dots1)_2$ | 0          | 1   | -inf(negative infinity) |
| $(1\dots1)_2$ | non-zero   | 0,1 | nan (not a number)      |
| $(0\dots0)_2$ | 0          | 0,1 | 0                       |
| $(0\dots0)_2$ | non-zero   | 0,1 | Subnormal number        |

</div>

{{< blockquote >}}
NOTE : From the table you must have noticed that the number 0 has two representations, one for each sign bit, that is why you will see Python and other languages printing -0 during computations.
{{< /blockquote >}}

### The usable range and the machine epsilon

Largest possible float64 value can be easily computed,

$E$ gets stored in 11bits and therefore has $2^{11}$ states out of which $(00\dots 0)\_{2}$ and $(11\dots1)\_2$ are reserved, therefore $E\_{max} = 2046$ and $E\_{min} = 1$,

The bias $E\_b$ can be calculated as $2^{11-1}-1 = 1023$.

Therefore the unbiased exponent ranges between $1-1023 = -1022\leq \text{exponent} \leq 2046-1023 = 1023$

```Python
>>> import numpy as np
>>> largest_positive = (1+np.sum(2**np.arange(-52,0,1,dtype= float)))*2**(2046-1023)
>>> smallest_positive = (1+0)*2**(1-1023)
>>> largest_positive,smallest_positive
(1.7976931348623157e+308, 2.2250738585072014e-308)
```

The interval between these two numbers gives us the **usable range** on the positive side of the number line.

<div class="image_container [.dark_&]!bg-primary">

![Image](./realmap.png)

</div>

Each float value is at a certain **gap** from it's neighbouring values if we view the values on the real number line. That is what allows us to cover such a large range of values.

If $N_1$ is a float then it's smallest increment $N_2$ can be produced by adding $(00\dots01)_2 = 2^{-52}$ to the mantissa.

<div class="content_container">
$$ N_1 = (1.M_1) \times 2^{E-E_b}$$
$$ N_2 = (1.M_1 + 2^{-52}) \times 2^{E-E_b}$$
$$ \epsilon(E-E_b) = N_2 - N_1 = 2^{-52} \times 2^{E-E_b} = 2^{E-E_b-52} $$
$$ \text{Machine epsilon} = \epsilon(1) = 2^{-52} = 2.220446049250313e-16 $$
</div>

The gap for any float value belonging different IEEE precisions can be obtained using `numpy.spacing` function.

```Python
>>> x = np.single(1)
>>> np.spacing(x)
1.1920929e-07
>>> y = np.double(1)
>>> np.spacing(y)
2.220446049250313e-16
>>> z = np.half(1)
>>> np.spacing(z)
0.000977
```

Although $\epsilon(E-E_b)$ which represents the **gap** varies with the exponent, the relative error is roughly equal to the **machine epsilon**. This constant is useful as it represents the relative error in every real value represented using inexact floats.

You can directly get the information about the float using,

```Python
>>> import sys
>>> sys.float_info
sys.float_info(max=1.7976931348623157e+308, max_exp=1024, max_10_exp=308, min=2.2250738585072014e-308, min_exp=-1021, min_10_exp=-307, dig=15, mant_dig=53, epsilon=2.220446049250313e-16, radix=2, rounds=1)
```

### Rounding

Real numbers which cannot be representated exactly by the IEEE 754 standard lie between two exactly representable values and therefore end up getting rounded while being stored, IEEE 754 recommends use of one of the following methods for rounding such numbers:-

- **Round to nearest** :
  The system chooses the nearer of the two possible neighbourhood values. If the exact value answer is exactly halfway between the two, the system chooses to store the one where the least significant bit of mantissa is zero. This behavior (round-to-even) prevents various undesirable effects.

    This is the default rounding rule followed by python float and numpy double datatypes.

    ```Python
    >>> 2.7 + np.spacing(2.7)*0.5
    2.7
    >>> 2.7 + np.spacing(2.7)*0.4
    2.7
    >>> 2.7 + np.spacing(2.7)*0.55
    2.7000000000000006
    ```

- **Round up, or round toward plus infinity** :
  The system chooses the larger of the two possible values (that is, the one further from zero if they are positive, and the one closer to zero if they are negative).
- **Round down, or round toward minus infinity** :
  The system chooses the smaller of the two possible values (that is, the one closer to zero if they are positive, and the one further from zero if they are negative).
- **Round toward zero, or chop, or truncate** :
  The system chooses the value that is closer to zero, in all cases.
