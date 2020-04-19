---
date: 2020-04-19
title: "Boundary Value Problems (BVP) with Fourier Transforms"
markup: mmark
---

Everyone has their own mental triggers when hearing specific words. Let's take the word "Engineering", which has many triggers, that can range all the way from "Tech shop that writes software" to "Aerospace Manufacturing". Even though I write code for living, one word comes to mind when I hear the word "Engineering", that is nothing but Differential Equations.

Lately, I've revisited Boundary Value Problems (BVP) and their relationship to Fourier Series. I'll attempt to simplify the topic a bit, but I can never do it justice.

A quick refresher on Fourier Series, the core idea is to take a periodic signal $$u(t)$$ and describes it as a sum of sine and cosine waves.(If you asked, why? the answer because it's easier to solve that way)

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

You can write a function to find the fourier coefficients and reconstruct them as necessary.

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

![Plot 2](/images/reconstructed.png#center)

All this can be visualized as follows (courtesy of wikipedia)

![Wiki Animation](https://upload.wikimedia.org/wikipedia/commons/5/50/Fourier_transform_time_and_frequency_domains.gif)

So how can this help solve Boundary Value Problems (BVP)

