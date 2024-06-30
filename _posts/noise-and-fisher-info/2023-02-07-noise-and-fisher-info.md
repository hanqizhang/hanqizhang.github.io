---
title: Noise and two simplified forms of Fisher Information
date: 2023-02-07
modified: 2024-06-25
tags: [neuroscience, perception]
description: Orientation efficient coding.
usemathjax: true
---

Fisher information (FI) is a measure of information (though not a strict [information quantity](https://en.wikipedia.org/wiki/Quantities_of_information)) and is commonly used in neuroscience. But for those new to the concept (like I was at some point), this post is a quick introduction to it. I assume that you already understand what neural tuning curves/functions and firing rate noise are.

<!-- For example, when I read that FI takes the form of \\(f'(\theta)^2\\) if noise is Gaussian, it wasn't clear to me why that is, or what \\(f(\theta)\\) is, even with the footnote explaining that it is the tuning function.  -->
At the end of the post I provide derivations for 2 basic results: 1) \\(J(\theta) = f'(\theta)\\) if firing rate noise is Gaussian and 2) \\(J(\theta) = f'(\theta)^2 / f(\theta)\\) when Poisson firing is assumed. This can serve as a primer to understanding the use of FI in [Kriegeskorte and Wei's 2021 review paper](https://www.nature.com/articles/s41583-021-00502-3).

<!-- On the way, I will also attempt to explain in simple words and illustrations how FI is related to concepts like neural tuning, representational geometry, perceptual discrimination, etc. -->

**The original formulation of Fisher Information**

In plain language and speaking generally, Fisher Information (FI) quantifies the amount of "information" an observation \\(r\\) of a random variable \\(R\\) contains about said random variable's parameter \\(\theta\\). The formal way that this quantity came to be defined by Ronald Fisher was:
$$
\begin{align}
  J(\theta) = Var(l'(\theta|r)) = \int{\left(-\frac{d}{d\theta}\log p(r|\theta)\right)^2p(r|\theta)dr}
\end{align}
$$
where the second equality is based on the equivalence relation between \\(Var(.)\\) and \\(E(.^2) - E^2(.)\\). The definition introduced in the paper is yet another equivalent form:
$$
\begin{align}
  J(\theta) = \int{-\frac{d^2}{d\theta}\log p(r|\theta)p(r|\theta)dr}
\end{align}
$$

**Fisher Information in neuroscience**

In neuroscience the situation where we care about this information is when \\(r\\) is the internal representation of the stimulus \\(\theta\\) and we want to measure how much information a reading of this internal representation (typically in the form of neural firing rates) conveys about the stimulus \\(\theta\\) that caused this much neural activity in the first place.

\\(r\\) can be a vector of activities of multiple neurons, but for simplicity let's assume it is just one neuron. The neuron reacts to different stimulus values differently following a tuning function \\(f(\theta)\\), and on top of that, it is also noisy. Generatively speaking, given a stimulus value \\(\theta_0\\), the neuron's activity \\(r\\) is \\(f(\theta_0) + \eta\\) where \\(\eta\\) can be modeled as a stimulus-independent Gaussian noise. In some cases, it can be more accurate to model the noise as stimulus-dependent Gaussian, or simply to model \\(r\\) as following a Poisson process with \\(f(\theta_0)\\) as the mean.

To visualize the tuning transform and the noise in the same picture:
<!-- ![tuning-noise](/images/tuning-noise.png) -->
<figure>
<img src="{{site.url}}/_posts/noise-and-fisher-info/tuning-noise.png"
     alt="tuning and noise"
     style="clear: both; margin-center: 0px;" />
<figcaption>Fig 1. Tuning and noise.</figcaption>
</figure>

**Two simplified forms of FI**

So far the basic definitions may have appeared complicated, especially the definition of FI. But as the authors mention in the paper, specific noise assumptions actually simplify the expression of FI.

1) If we assume stimulus-independent Gaussian noise with variance 1, then FI is simply:
$$
\begin{align}
  J(\theta) = f'(\theta)^2, \label{eq1}\tag{1}
\end{align}
$$
&nbsp;&nbsp;&nbsp;&nbsp; where \\(f(\theta)\\) is the tuning function.

2) If we assume Poisson spiking, then FI can be shown to have the following form instead:
$$
\begin{align}
  J(\theta) = \frac{f'(\theta)^2}{f(\theta)} \label{eq2}\tag{2}
\end{align}
$$
Here I provide the proofs. Hopefully with the background introduction on what the tuning \\(f(\theta)\\) is and where the noise comes in, these become easy to understand.

Proof for \eqref{eq1}:
When \\(p(r|\theta)\\) is Gaussian:
$$
\begin{align}
  \log p(r|\theta) &= -\frac{(r-f(\theta))^2}{2\sigma^2} + const \label{eq3}\tag{3}\\ 
  J(\theta) &= -E_{r|\theta}[\frac{d^2}{d\theta^2}\log p(r|\theta)] \label{eq4}\tag{4}\\
            &= -E_{r|\theta}[\frac{d^2}{d\theta^2}(\frac{rf(\theta)}{\sigma^2}-\frac{f(\theta)^2}{2\sigma^2})] \label{eq5}\tag{5}\\
            &= -E_{r|\theta}[\frac{rf''(\theta)}{\sigma^2}-(\frac{2f(\theta)f'(\theta)}{2\sigma^2})'] \label{eq6}\tag{6}\\
            &= -E_{r|\theta}[\frac{rf''(\theta)}{\sigma^2}-\frac{f'(\theta)^2}{\sigma^2}-\frac{f(\theta)f''(\theta)}{\sigma^2}] \label{eq7}\tag{7}\\
            &= -E[-\frac{1}{\sigma^2}f'(\theta)^2] \label{eq8}\tag{8}\\
  J(\theta) &= f'(\theta)^2 \label{eq9}\tag{9}
\end{align}
$$

\eqref{eq7} to \eqref{eq8} is because \\(E_{r|\theta}(r) = f(\theta)\\),
and \eqref{eq8} to \eqref{eq9} is assuming \\(\sigma = 1\\).

Proof for \eqref{eq2} <!--([2](#mjx-eqn-eq2))--> follows the same procedure. If \\(p(r|\theta)\\) is Poisson:
$$
\begin{align}
  \log p(r|\theta) &= \log \frac{f(\theta)^re^{-f(\theta)}}{r!} = r\log f(\theta) - f(\theta) - \log r! \label{eq10}\tag{10}\\
  J(\theta) &= -E_{r|\theta}[r(\frac{f'(\theta)}{f(\theta)})'-f''(\theta)] \label{eq11}\tag{11}\\
            &= -E_{r|\theta}[r \frac{f''(\theta)f(\theta)-f'(\theta)^2}{f(\theta)^2}-f''(\theta)] \label{eq12}\tag{12}\\
            &= -[f''(\theta)-\frac{f'(\theta)^2}{f(\theta)}-f''(\theta)] \label{eq13}\tag{13}\\
  J(\theta) &= \frac{f'(\theta)^2}{f(\theta)} \label{eq14}\tag{14}
\end{align}
$$

Now so far we have been assuming that \\(r\\) is a scalar, namely that only one neuron encodes \\(\theta\\). When multiple neurons encode the parameter together, as long as the neurons have independent noise, the FI is just the sum of the FI for individual neurons.

Finally, if \\(\theta\\) itself is multidimensional, then \\(J(\theta)\\) becomes a matrix, e.g., \\( \mathbf{J}(\theta_1, \theta_2) \\) where
$$
\begin{align}
    \mathbf{J}_{11} &= J(\theta_1, \theta_2=\theta_2) \\ 
    \mathbf{J}_{22} &= J(\theta_1=\theta_1, \theta_2) \\ 
    \mathbf{J}_{12} &= \mathbf{J}_{21} = J(\theta_1, \theta_2)
\end{align}
$$
