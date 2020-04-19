---
date: 2020-04-19
title: "Boundary Value Problems (BVP) with Fourier Transforms"
markup: mmark
---

Everyone has their own mental triggers when hearing specific words. Let's take the word "Engineering", which has many triggers, that can range all the way from "Tech shop that writes software" to "Aerospace Manufacturing". Even though I write code for living, one word comes to mind when I hear the word "Engineering", that is nothing but Differential Equations.

Lately, I've revisited Boundary Value Problems (BVP) and their relationship to Fourier Series. I'll attempt to simplify the topic a bit, but I can never do it justice.

A quick refresher on Fourier Series, the core idea is to take a periodic signal $$u(t)$$ and describes it as a sum of sine and cosine waves.

A classical example, imagine you have the square wave function, the produces a signal like in the plot below:

``` python
import matplotlib.pyplot as plt
import numpy as np

fs = 100
square_wave_function = lambda t: (abs((t % 1) - 0.25) < 0.25).astype(float) - (
    abs((t % 1) - 0.75) < 0.25
)

t = np.arange(-2, 2, 1 / fs)
plt.plot(t, square_wave_function(t))
plt.xlabel("$t$")
plt.ylabel("$u(t)$")
plt.title("Square Wave Function")
plt.show()
```

![Plot 1](/images/square_wave_func.png#center)

Now, you can decompose that function in it's $$sin$$ and $$cos$$ coefficients. The math is pretty complex, but you can write a quick python function to do the heavy lifting, first let's look at the math; 

$$
u(t)=\frac{a_0}{2}+\sum_{n=1}^{\infty}a_n\cos(2\pi \frac{nt}{L})+b_n\sin(2\pi\frac{nt}{L}), 
$$

where $$a_n$$ and $$b_n$$ are the coefficients of the Fourier series. They can be calculated by

$$
a_n=\frac{2}{L}\int_0^Lu(t)\cos(2\pi \frac{nt}{L})dt
$$

$$
b_n=\frac{2}{L}\int_0^Lu(t)\sin(2\pi \frac{nt}{L})dt
$$

As for python:

``` python
def fourier_series(period, N):
    result = []
    T = len(period)
    t = np.arange(T)
    for n in range(N + 1):
        an = 2 / T * (period * np.cos(2 * np.pi * n * t / T)).sum()
        bn = 2 / T * (period * np.sin(2 * np.pi * n * t / T)).sum()
        result.append((an, bn))
    return np.array(result)

def reconstruct_signal(P, anbn):
    result = 0
    t = np.arange(P)
    for n, (a, b) in enumerate(anbn):
        if n == 0:
            a = a / 2
        result = (
            result

            + a * np.cos(2 * np.pi * n * t / P)
            + b * np.sin(2 * np.pi * n * t / P)

        )
    return result

t_period = np.arange(0, 1, 1 / fs)
f = fourier_series(square_wave_function(t_period), fs)
plt.plot(t_period, square_wave_function(t_period), label="Original Signal")

plt.plot(
    t_period,
    reconstruct_signal(len(t_period), f[:30, :]),
    label="Reconstructed with 30 Harmonics",
)

plt.ylabel("$x(t)$")
plt.xlabel("$t$")
plt.legend(fontsize=8)
plt.savefig("reconstructed.png")
plt.show()
```

This is the approximation produced by reconstructing the signal just as sum of $$\sin$$ and $$\cos$$

![Plot 2](/images/reconstructed.png#center)

You can explain the whole process in one visualization (courtesy of wikipedia)

![Wiki Animation](https://upload.wikimedia.org/wikipedia/commons/5/50/Fourier_transform_time_and_frequency_domains.gif)

Now, how can this help solve Boundary Value Problems (BVP). If you are familiar with the topic, you will recommend using, [The Shooting Method](https://en.wikipedia.org/wiki/Shooting_method).
Basically, instead of tying a general solution to a boundary value problem down at both points, one only ties it down at one end. This leaves a free parameter (in the case of a second order problem). For a given value of this free parameter, one then integrates out a solution to the differential equation. Specifically, you start at the tied-down boundary point and integrate out just like an initial value problem. When you get to the other boundary point, the error between your solution and the true boundary condition tells you how to adjust the free parameter. Repeat this process until a solution matching the second boundary condition is obtained.

But for unbounded regions, this becomes tricky, we can apply the same ideas from Fourier transforms, For example:

$$
x =\begin{cases} u_{t}=4u_{xx}+\sin(t), &  -\infty  \leq  x \leq \infty, t>0\\u(x, 0)=e^{-x^2} \sin(x), & -\infty \leq x \leq \infty \end{cases}
$$

Then sum these solutions using an integral, 

$$
u(x, t) = \int_{-\infty}^{\infty}c(s)e^{-4s^2t}e^{isx}ds, 
$$

where $$c(x)$$ are the coefficients in the sum, analogous to discrete sums. The coefficient function is determined by the initial condition

$$
e^{-x^2}\sin(x) = \int_{-\infty}^{\infty}c(s)e^{isx}ds.
$$

The solution is then

$$
u(x, t) = \frac{1}{\sqrt{2\pi}i}\int_{-\infty}^{\infty}\{ \hat{f}(s-1)-\hat{f}(s+1)\}e^{-4s^2 t}e^{isx}ds.
$$

Python code on github, [https://github.com/zeyaddeeb/boundry-value-problems](https://github.com/zeyaddeeb/boundry-value-problems)

The Fourier transform applications are endless, it can still pull a good trick even in Ordinary Differential Equations.

#### References

[https://math.stackexchange.com/questions/2063595/using-fourier-transforms-to-solve-the-boundary-value-problem](https://math.stackexchange.com/questions/2063595/using-fourier-transforms-to-solve-the-boundary-value-problem)

