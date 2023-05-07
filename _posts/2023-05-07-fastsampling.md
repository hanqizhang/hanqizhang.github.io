---
title: 'Fast neural sampling - a paper review'
date: 2023-05-07
permalink: /posts/2023/05/blog-post-1/
tags:
  - neuroscience
  - neural sampling
  - Bayesian inference
---


The original paper was published at the NeurIPS 2014 conference by Hennequin, Aitchison and Lengyel. Last semester, 
I reviewed this paper together with Jeong Woo Kim, a colleague at NYU, with the goal of reproducing all simulation results 
provided in the main figures except for the part regarding Dale's Law. We also aim to explain key concepts in a manner that 
elucidates the high-level structure of the paper and clarifies the key logical connections among sections of the paper. On a 
lower level, we fill in the intermediate steps of derivations in the paper where possible.

**1. Introduction**

In probabilistic inference problems, sampling-based methods simplify computations on complex distributions, as we have seen in 
algorithms like particle filtering. Not surprisingly, in neuroscience, the neural sampling hypothesis states that the brain also 
performs inference by generating sequences of samples from high-dimensional posterior distributions, implemented through neural 
circuitry. In this context, each sample is represented by the instantaneous activity levels (e.g. a vector of membrane potentials or 
firing rates) of a population of neurons. 

However, existing neurophysiologically-plausible sampling methods suffer from low sampling speed, failing to account for the brain's 
ability to perform relatively accurate inference within a behavioral time scale. For sampling to be fast, the neuronal population needs 
to produce uncorrelated samples within a small number of time steps, so that the sequence of samples generated throughout a short time 
window is able to well approximate the target distribution. With this goal in mind, Hennequin et al set out to improve the an existing 
sampling method, Langevin Sampling (LS), and were able to achieve significantly faster sampling.

This report will explain the key concepts and derivations of the original paper, drawing connections to concepts learned in class wherever 
helpful. The code for the paper is not publicly available, so this makes it a more meaningful practice for us to write our own simulations 
and optimizations in Python to reproduce all the figures in the paper. 

**2. Problem Definition and Algorithm**

In short, the objective is to increase the sampling speed of a neuronal network which generates samples from the target posterior distribution 
of an underlying inference problem. We first describe the underlying inference problem, and then explain how to construct 
the recurrent neuronal network that can sample from the posterior given the evidence as network input. We define sampling speed and describe the 
algorithms for speed optimization. At the end of this section, we also list the algorithms for sampling simulations.

***Inference under a linear Gaussian model.*** 

The inference problem itself does not have temporal dynamics, since we are interested only in 
the short time frame within which the brain quickly carries out an inference from some observed input that can be considered static within that i
nstant. Hence we denote the latent variable $${\mathbf{r}_\bullet}$$ and the observation or input $${\mathbf{h}_\bullet}$$. The generative model is as follows, 
where $\mathbf{C}$ and $\mathbf{A}$ are taken as given:
$$
\begin{align}
    \mathbf{r_\bullet} &\sim \mathcal{N}(0, \mathbf{C})\\
    \mathbf{h_\bullet} &= \mathbf{A}\mathbf{r_\bullet} + \mathbf{v_\bullet}\\
    \mathbf{v_\bullet} &\sim \mathcal{N}(0, \sigma^2\mathbf{I})
\end{align}
$$
As the latent variable is static and the noise $\mathbf{v}$ is spherical, this is essentially a sensible PCA problem. We can compute the posterior u
sing Bayes' Rule: $p(\mathbf{r}|\mathbf{h}) \propto p(\mathbf{h}|\mathbf{r})p(\mathbf{r})$. Specifically, we need to first write 
$p(\mathbf{h}|\mathbf{r}) = \mathcal{N}(\mathbf{h}; \mathbf{Ar}, \sigma_h^2\mathbf{I})$ in terms of $\mathbf{r}$ before we can multiply the two densities:
$$
\begin{align}
    p(\mathbf{h}|\mathbf{r}) &= \mathcal{N}(\mathbf{h}; \mathbf{Ar}, \sigma_h^2\mathbf{I}) = \mathcal{N}(\mathbf{Ar}; \mathbf{Ar}, \sigma_h^2\mathbf{I})\\
    &= \mathcal{N}(\mathbf{r}; (\mathbf{A}^\top \mathbf{A})^{-1}\mathbf{A}^\top \mathbf{h}, (\mathbf{A}^\top \mathbf{A})^{-1}\mathbf{A}^\top \sigma_h^2\mathbf{I} [(\mathbf{A}^\top \mathbf{A})^{-1}\mathbf{A}^\top]^\top)\\
    &= \mathcal{N}(\mathbf{r}; (\mathbf{A}^\top \mathbf{A})^{-1}\mathbf{A}^\top \mathbf{h}, (\mathbf{A}^\top \mathbf{A})^{-1}\sigma_h^2)
\end{align}
$$
With this, we can multiply the multiply the densities to obtain $p(\mathbf{r}|\mathbf{h})$ using the Gaussian identity for the multiplication of two 
Gaussian pdfs:
$$
\begin{align}
    p(\mathbf{r}|\mathbf{h}) &\propto p(\mathbf{h}|\mathbf{r})p(\mathbf{r}) = \mathcal{N}(\mathbf{\mu},\mathbf{\Sigma})\nonumber\\
    \mathbf{\mu} &= \mathbf{\Sigma}(\mathbf{A}^\top \mathbf{A}/\sigma_h^2)(\mathbf{A}^\top \mathbf{A})^{-1}\mathbf{A}^\top \mathbf{h} \nonumber\\
        &= \mathbf{\Sigma} \mathbf{A}^\top \mathbf{h}/\sigma_h^2 \label{eqn: mu-target}\\
    \mathbf{\Sigma} &= (\mathbf{C}^{-1}+ \mathbf{A}^\top \mathbf{A}/\sigma_h^2)^{-1} \label{eqn: Sigma-target}
\end{align}
$$
Here, it is important to note that only the mean of the posterior depends on $\mathbf{h}$, while the covariance matrix does not. When analyzing 
the sampling speed, this property will allow us to set $\mathbf{h}$ to zero without loss of generality.

The next goal is to construct a neuronal network whose activity in the recurrent layer has a stationary distribution that is equal to the 
posterior distribution $p(\mathbf{r}|\mathbf{h})$ independent of the input $\mathbf{h}$. This network serves as a plausible circuit model for how the 
brain samples from the posterior distribution.

***Linear Stochastic Recurrent Dynamics.*** 

As shown in Fig.1, we construct a linear network of recurrently connected neurons, $\mathbf{r}$, 
where the recurrent weights are $\mathbf{W}$. They receive feed-forward input from $\mathbf{h}$ with weight matrix $\mathbf{F}$. At the same time, 
$\mathbf{r}$ also receives external noise $\mathbf{\xi}$. The corresponding network dynamics is described in Eq.\eqref{sde1}. 
<!-- \begin{figure}[h]
\begin{center}
  \includegraphics[width=1.5 in]{network.png}   
\end{center}
  \caption{Recurrent neuronal network}
  \label{fig:network}
\end{figure} -->
$$
\begin{equation}
    d\mathbf{r} = \frac{dt}{\tau_m} \left( -\mathbf{r}(t) + \mathbf{Wr}(t) + \mathbf{Fh} \right) + \sigma_{\xi} \sqrt{\frac{2}{\tau_m}} d\mathbf{\xi} (t)
    \label{sde1}
\end{equation}
$$
A nuanced distinction between the notation for the network and the notation in the inference problem description is that we will design 
the network such that the recurrent neurons $\mathbf{r}$ have a stationary distribution matching the posterior distribution $p(\mathbf{r}|\mathbf{h})$ 
described in section \ref{sec: inference}, not the marginal distribution $p(\mathbf{r})$. We will explain how to design the weights to achieve 
this in section \ref{sec: control}.

The Wiener process, $d\mathbf{\xi}$, of unit variance can be rewritten as, $d\mathbf{\xi}(t) = \sqrt{dt} \mathcal{N}(0,1)$. Then, the stationary 
distribution of $\mathbf{r}$ is when $\mathbb{E}(d\mathbf{r}) = 0$:
$$
\begin{align}
    0 &= \mathbb{E} \left( \frac{dt}{\tau_m} \left( -\mathbf{r}(t) + \mathbf{Wr}(t) + \mathbf{Fh} \right) + \sigma_{\xi} \sqrt{\frac{2}{\tau_m}} \cdot \sqrt{dt} \mathcal{N} (0,1) \right) \\
    0 &= \frac{dt}{\tau_m} \mathbb{E} \left( -\mathbf{r}(t) + \mathbf{Wr}(t) + \mathbf{Fh} \right) + \sigma_{\xi} \sqrt{\frac{2}{\tau_m}} \cdot \sqrt{dt} \cdot \mathbb{E} \left( \mathcal{N}(0,1) \right) \\
    0 &= \frac{dt}{\tau_m} \mathbb{E} \left( (-\mathbf{I} + \mathbf{W}) \mathbf{r}(t) + \mathbf{Fh} \right) + \sigma_{\xi} \sqrt{\frac{2}{\tau_m}} \cdot \sqrt{dt} \cdot 0 \\
    0 &= \frac{dt}{\tau_m} \left( (-\mathbf{I}+\mathbf{W})\mathbb{E}(\mathbf{r}(t)) + \mathbf{Fh} \right) \\
    (-\mathbf{I}+\mathbf{W}) \mathbf{\mu}^{\mathbf{r}} &= -\mathbf{Fh} \\
    \mathbf{\mu}^\mathbf{r}(\mathbf{h}) &= (\mathbf{I}-\mathbf{W})^{-1}\mathbf{Fh} \label{eqn: mu}
\end{align}
$$
And the covariance of the stationary distribution is $\mathbf{\Sigma^r} = \langle(\mathbf{r}(t)-\mathbf{\mu^r})(\mathbf{r}(t)-\mathbf{\mu^r})^\top \rangle_t $.

***Simplifying the Control Theory Intuition Using a Discrete-time Analog.*** 

Realizing that Eq.\eqref{sde1}'s discrete-time version is essentially 
an AR1 process is greatly helpful for intuitively understanding the dynamics of the network. It also simplifies our explanation of the control 
theory references in the paper that the authors used to explain how the network's stationary mean $\mathbf{\mu^r}$ and covariance $\mathbf{\Sigma^r}$ are 
able to match the target posterior  $\mathbf{\mu}$ and $\mathbf{\Sigma}$. Therefore, here we will base our explanation in the discrete-time context, and 
the conclusions will be convertible to the continuous-time version by following a simple process outlined [here]().

The AR1 process that corresponds to Eq.\eqref{sde1} is:
$$
\begin{align}
    \mathbf{r}(t+1) - \mathbf{r}(t) &= -\mathbf{r}(t) + \mathbf{Wr}(t) + \mathbf{Fh} + \sigma_{\xi} \mathbf{\eta}(t)\nonumber\\
    \mathbf{r}(t+1) &= \mathbf{Wr}(t) + \mathbf{Fh} + \sigma_{\xi} \mathbf{\eta}(t)
\end{align}
$$
where $\mathbf{\eta}(t)$ is white noise with covariance $\mathbf{I}$. As long as a stationary distribution exists for  $\mathbf{r}$, its covariance 
matrix $\mathbf{\Sigma^r}$ is related to the covariance matrix $$\sigma^2_{\xi}\mathbf{I}$$ of the noise input as follows:
$$
\begin{align}
    \mathbf{\Sigma^r} &= \mathbb{E}(\mathbf{r}(t+1)\mathbf{r}(t+1)^\top)\\
    &= \mathbb{E}((\mathbf{Wr}(t)+\mathbf{Fh} + \sigma_{\xi} \mathbf{\eta}(t))(\mathbf{Wr}(t)+\mathbf{Fh} + \sigma_{\xi} \mathbf{\eta}(t))^\top)\\
    &= \mathbb{E}((\mathbf{Wr}(t)+\mathbf{Fh})(\mathbf{Wr}(t)+\mathbf{Fh})^\top) + \mathbb{E}((\mathbf{Wr}(t)+\mathbf{Fh})\sigma_{\xi} \mathbf{\eta}(t)^\top) + ...\\
    &\ \ \ \ \ \ \ \ \ \ \ \mathbb{E}(\sigma_{\xi} \mathbf{\eta}(t)(\mathbf{Wr}(t)+\mathbf{Fh})^\top) + \sigma^2_{\xi}\mathbb{E}(\mathbf{\eta}(t)\mathbf{\eta}(t)^\top)\\
    &= \mathbf{W\Sigma^rW}^\top + 0 + 0 + \sigma^2_{\xi}\mathbf{I} \\
    \mathbf{\Sigma^r} &= \mathbf{W\Sigma^rW}^\top + \sigma^2_{\xi}\mathbf{I} \label{eqn: lyap-discrete}
\end{align}
$$
Eq.\eqref{eqn: lyap-discrete}, called the discrete Lyapunov equation, is thus derived. This can be converted to the continuous-time version by 
the derivation discussed in \cite{wikipedia}, and then the Lyapunov equation becomes:
$$
\begin{equation}
    (\mathbf{W}-\mathbf{I})^\top \mathbf{\Sigma^r} + \mathbf{\Sigma^r}(\mathbf{W}-\mathbf{I}) + 2\sigma^2_{\xi} \mathbf{I} = 0 \label{eqn: lyap-cont}
\end{equation}
$$
Now, through $\mathbf{W}$, $\mathbf{F}$ and $\sigma_{\xi}$, we have control over the stationary distribution covariance as in Eq.\eqref{eqn: lyap-cont} 
as well as over the stationary distribution mean as in Eq.\eqref{eqn: mu}. For the network to fulfill its promise, our goal is to find $\mathbf{W}$, 
$\mathbf{F}$ and $\sigma_{\xi}$ such that the solutions $\mathbf{\Sigma^r}$ and $\mathbf{\mu^r}(\mathbf{h})$ to Eq.\eqref{eqn: mu} and Eq.\eqref{eqn: lyap-cont} 
satisfy $\mathbf{\Sigma^r} = \mathbf{\Sigma}$ and $\mathbf{\mu^r}(\mathbf{h}) = \mathbf{\mu}(\mathbf{h})$ for any $\mathbf{h}$.

***Langevin Sampling Corresponds to One Special Solution.*** 

One way to set $\mathbf{W}$, $\mathbf{F}$ (while leaving $\sigma_{\xi}$ undetermined) is
$$
\begin{align}
    \mathbf{F} = (\sigma_{\xi}/\sigma_{h})^2\mathbf{A}^\top \text{ and } \mathbf{W} =\mathbf{I} - \sigma_{\xi}^2\mathbf{\Sigma}^{-1}
    \label{eqn: LS-solution}
\end{align}
$$
And it can be easily shown that Eq.\eqref{sde1} with these parameters is equivalent to a common sampling technique -- Langevin Sampling (LS), 
which is often expressed as a "noisy gradient ascent of the posterior":
$$
\begin{align}
    d\mathbf{r} = \frac{1}{2}\frac{\partial}{\partial\mathbf{r}}\log p(\mathbf{r}|\mathbf{h})dt + d\mathbf{\xi}
\end{align}
$$
as long as $\mathbf{r}|\mathbf{h}$ is Gaussian. This is verifiable by plugging in the Gaussian pdf for $p(\mathbf{r}|\mathbf{h})$ to the above equation.

However, the authors were able to derive that LS is very slow. We will skip the derivations as they are not the main focus of the paper and do 
not hinder our understanding of sampling speed. In \ref{sec: speed} we will discuss what slowness means and how it is measured. In the Experimental 
Evaluation section, we will show numerically evaluated estimations and simulation results that demonstrate how slow LS is.

Despite being slow, the LS solution provides a good starting point to find more general solutions that might make sampling faster. And indeed we can verify that any $\mathbf{W}$ of the following form is also a solution to Eq.\eqref{eqn: lyap-cont} with $\mathbf{\Sigma^r}$ set to $\mathbf{\Sigma}$ being the target posterior covariance matrix.
$$
\begin{equation}
    \mathbf{W}(\mathbf{S}) =\mathbf{I} - (\sigma_{\xi}^2\mathbf{I} + \mathbf{S})\mathbf{\Sigma}^{-1}
\end{equation}
$$
where $\mathbf{S}$ is an arbitrary skew-symmetric matrix ($\mathbf{S}^\top = -\mathbf{S}$).

***Measuring Sampling Speed.***

Assuming that generating a sample takes a fixed amount of time proportional to the membrane time constant $\tau_m$, sampling efficiently from a 
target distribution requires that subsequent samples be relatively uncorrelated with one another. Otherwise, the sequence of samples would tend 
to come from a focused region of the target distribution and fail to adequately approximate the entire distribution.

Therefore, sampling speed is measured by the autocorrelation of neuronal activity. Intuition from the AR1 process is again helpful here. We know 
that the decaying speed of the ACF of a stationary AR1 process is dominated by the largest eigenvalue $\mathbf{\lambda}_{\max}$ of the weight 
matrix $\mathbf{W}$. Since our goal is to make the ACF decay away with lag $k$ as fast as possible (unlike in the typical use of AR1 models 
where we hope autocorrelation stays large for the predictabilty of the time series), we want to optimize the weight matrix such that its largest 
eigenvalue, which is less than one, to be small, or as far away from one as possible.

In the continuous time version, we want $Re(\mathbf{\lambda}_{\max}^{\mathbf{W}-\mathbf{I}})$ to be as far away from zero as possible. However, 
it turns out that optimizing for the largest eigenvalue is not as easy as directly optimizing for shape of the ACF. For this, the authors defined 
a slowing cost function:
$$
\begin{equation}
    \psi_{slow} (\mathbf{W}) = \frac{1}{2\tau_m N^2} \int_{0}^\infty \Big\vert\Big\vert \mathbf{\Lambda}^{-\frac{1}{2}} \mathbf{K}(\mathbf{W},\tau) \mathbf{\Lambda}^{-\frac{1}{2}} \Big\vert\Big\vert_F^2 d\tau
    \label{10}
\end{equation}
$$
This cost function essentially measures the area under the ACF which, for processes with multidimensional variable, is defined through the Frobenius 
norm of the autocorrelation matrix $\mathbf{\Lambda}^{-\frac{1}{2}} \mathbf{K}(\mathbf{S},\tau) \mathbf{\Lambda}^{-\frac{1}{2}}$, where 
$\mathbf{K}(\tau) = \langle(\mathbf{r}(t+\tau)-\mathbf{\mu})(\mathbf{r}(t)-\mathbf{\mu})^\top \rangle_t$ is the matrix of lagged autocovariances, 
and $\mathbf{\Lambda} = \text{diag}(\mathbf{\Sigma})$ is the diagonal matrix with all the posterior variances. 

And since $\mathbf{W}$ is parameterized by $\mathbf{S}$, we can write the slowing cost as a function of $\mathbf{S}$ instead. Adding a $L_2$-norm 
penalty, the loss on which we will do gradient descent to optimize sampling speed is:
$$
\begin{equation}
    \mathcal{L}(\mathbf{S}) = \frac{1}{2\tau_m N^2} \int_{0}^\infty \Big\vert\Big\vert \mathbf{\Lambda}^{-\frac{1}{2}} \mathbf{K}(\mathbf{S},\tau) \mathbf{\Lambda}^{-\frac{1}{2}} \Big\vert\Big\vert_F^2 d\tau + \frac{\lambda_{L_2}}{2N^2}\Big\vert\Big\vert \mathbf{W}(\mathbf{S}) \Big\vert\Big\vert_F^2
\end{equation}
$$

***Sampling Algorithms.***

Once we have optimized for $\mathbf{W}$, we can plug it in the sampling algorithm to simulate the samples generated from the recurrent network. 
The simulation results help visualize the sampling behavior and empirically analyzing how sampling is sped up, which will be shown in 
Section \ref{sec: Experimental Evaluation}. Here we describe the sampling algorithms.

***Sampling for Eq.\eqref{sde1}.*** Eq.\eqref{sde1} can be discretized using the Euler-Maruyama method, and be written as:
$$
\begin{equation}
\mathbf{r}(t+dt) - \mathbf{r}(t) = \frac{dt}{\tau_m}[-\mathbf{r}(t) + \mathbf{Wr}(t) + \mathbf{Fh}] + \sigma_{\xi}\sqrt{\frac{2dt}{\tau_m}}\mathbf{\eta},
\end{equation}
$$
where $\mathbf{\eta} \sim \mathcal{N}(0,1)$ and $\mathbf{h}$ is zero without loss of generality.

Sampling from the posterior by linear stochastic recurrent dynamics
- **Input:** Skew-symmetric matrix $\mathbf{S}$ and the covariance matrix $\mathbf{\Sigma}$ of the target posterior.
- Compute the weight matrix $\mathbf{W} = \mathbf{I} + (-\sigma_{\xi}^2\mathbf{I} + \mathbf{S})\mathbf{\Sigma}^{-1}$
- Generate initial sample $\mathbf{r}(0) = 0$
- For $k=1,2,...,K$:
$$
\begin{align}
    &\text{Generate } \mathbf{\eta}(k) \sim \mathcal{N}(0,1)\\
    &\mathbf{r}(k) = \mathbf{r}(k-1) + \frac{dt}{\tau_m}[-\mathbf{r}(k-1) + \mathbf{Wr}(k-1)] + \sigma_{\xi}\sqrt{\frac{2dt}{\tau_m}}\mathbf{\eta}(k)
\end{align}
$$
- **Output:** K samples $${\{\mathbf{r}(1), ..., \mathbf{r}(K)\}}$$ from the posterior: $${\mathbf{r|h}=0 \sim \mathcal{N}(0, \mathbf{\Sigma})}$$


***Gibbs sampling.*** We also compare the above sampling technique with Gibbs Sampling. Gibbs Sampling samples the posterior 
$\mathbf{r}|\mathbf{h}$ by sampling $\mathbf{r}_i | \mathbf{r}_{j\neq i}, \mathbf{h}$ sequentially for each $\mathbf{r}_i$.

Since $$p(\mathbf{r}|\mathbf{h}) = \mathcal{N}(0, \mathbf{\Sigma})$$, thus $$p(\mathbf{r}_i|\mathbf{r}_{j\neq i}, \mathbf{h}) = \mathcal{N}(\mathbf{\mu}^{[i]}, \mathbf{\Sigma}^{[i]})$$ 
where:
$$
\begin{align}
    \mathbf{\mu}^{[i]} &= 0 + \mathbf{D} \mathbf{B}^{-1} (\mathbf{r}_{j\neq i} - 0) = \mathbf{D} \mathbf{B}^{-1} \mathbf{r}_{j\neq i}\\
    \mathbf{\Sigma}^{[i]} &= \sigma_i^2 - \mathbf{DB}^{-1}\mathbf{D}^\top
\end{align} 
$$
where $\sigma_i^2$ is the $i$-th diagonal element of $$\mathbf{\Sigma}$$, and $$\mathbf{D} = \mathbf{\Sigma}_{row=i,col\neq i}$$ and 
$$\mathbf{B} = \mathbf{\Sigma}_{row\neq i, col\neq i}$$. In other words, $$\mathbf{D}$$ is a 1$\times (N-1)$ matrix corresponding to the $i$-th row 
of $$\mathbf{\Sigma}$$ but excluding the $i$-th column. $$\mathbf{B}$$ is $$\mathbf{\Sigma}$$ excluding the $i$-th row and $i$-th column.
