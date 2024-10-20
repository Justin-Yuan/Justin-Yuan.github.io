---
layout: distill
title: 'Generative Modeling by Estimating Gradients of the Data Distribution'
date: 2021-05-05 11:12:00-0400
description: 'This blog post focuses on a promising new direction for generative modeling. We can learn score functions (gradients of log probability density functions) on a large number of noise-perturbed data distributions, then generate samples with Langevin-type sampling. The resulting generative models, often called <em>score-based generative models</em>, has several important advantages over existing model families: GAN-level sample quality without adversarial training, flexible model architectures, exact log-likelihood computation, and inverse problem solving without re-training models. In this blog post, we will show you in more detail the intuition, basic concepts, and potential applications of score-based generative models.'
authors:
  - name: Yang Song  
    affiliations: 
      name: Stanford University
bibliography: blogs.bib
comments: true
hidden: true
_meta: >
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:site" content="@YSongStanford">
  <meta name="twitter:creator" content="@YSongStanford">
  <meta name="twitter:title" content="Generative Modeling by Estimating Gradients of the Data Distribution">
  <meta name="twitter:description" content="An introduction to the intuition, basic concepts and applications of score-based generative modeling">
  <meta name="twitter:image" content="http://yang-song.github.io/assets/img/score/sde_schematic_21.jpg">
---

<d-contents>

  <nav class="l-text figcaption">
  <h3>Contents</h3>
    <div><a href="#introduction"> Introduction </a></div>
    <div><a href="#the-score-function-score-based-models-and-score-matching">The score function, score-based models, and score matching</a></div>
    <div><a href="#langevin-dynamics">Langevin dynamics</a></div>
    <div><a href="#naive-score-based-generative-modeling-and-its-pitfalls">Naive score-based generative modeling and its pitfalls</a></div>
    <div><a href="#score-based-generative-modeling-with-multiple-noise-perturbations">Score-based generative modeling with multiple noise perturbations</a></div>
    <div><a href="#score-based-generative-modeling-with-stochastic-differential-equations-sdes">Score-based generative modeling with stochastic differential equations (SDEs)</a>			</div>
    <ul>
      <li><a href="#perturbing-data-with-an-sde">Perturbing data with an SDE</a></li>
      <li><a href="#reversing-the-sde-for-sample-generation">Reversing the SDE for sample generation</a></li>
      <li><a href="#estimating-the-reverse-sde-with-score-based-models-and-score-matching">Estimating the reverse SDE with score-based models and score matching</a></li>
      <li><a href="#how-to-solve-the-reverse-sde"> How to solve the reverse SDE </a></li>
      <li><a href="#probability-flow-ode">Probability flow ODE</a></li>
      <li><a href="#controllable-generation-for-inverse-problem-solving">Controllable generation for inverse problem solving</a></li>
    </ul>
    <div><a href="#connection-to-diffusion-models-and-others">Connection to diffusion models and others</a></div>
    <div><a href="#concluding-remarks">Concluding remarks</a></div>      
  </nav>
</d-contents>

## Introduction

Existing generative modeling techniques can largely be grouped into two categories based on how they represent probability distributions. 

1. **likelihood-based models**, which directly learn the distribution's probability density (or mass) function via (approximate) maximum likelihood. Typical likelihood-based models include autoregressive models <d-cite key="larochelle2011neural,germain2015made,van2016pixel"></d-cite>, normalizing flow models <d-cite key="dinh2014nice,dinh2016density"></d-cite>, energy-based models (EBMs)<d-cite key="lecun2006tutorial,song2021train"></d-cite>, and variational auto-encoders (VAEs) <d-cite key="kingma2013auto,rezende2014stochastic"></d-cite>. 
2. **implicit generative models** <d-cite key="mohamed2016learning"></d-cite>, where the probability distribution is implicitly represented by a model of its sampling process. The most prominent example is generative adversarial networks (GANs) <d-cite key="goodfellow2014generative"></d-cite>, where new samples from the data distribution are synthesized by transforming a random Gaussian vector with a neural network.

<div class='l-body'>
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/likelihood_based_models.png">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Bayesian networks, Markov random fields (MRF), autoregressive models, and normalizing flow models are all examples of likelihood-based models. All these models represent the probability density or mass function of a distribution. </figcaption>
</div>

<div class='l-body'>
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/implicit_models.png">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> GAN is an example of implicit models. It implicitly represents a distribution over all objects that can be produced by the generator network. </figcaption>
</div>

Likelihood-based models and implicit generative models, however, both have significant limitations. Likelihood-based models either require strong restrictions on the model architecture to ensure a tractable normalizing constant for likelihood computation, or must rely on surrogate objectives to approximate maximum likelihood training. Implicit generative models, on the other hand, often require adversarial training, which is notoriously unstable <d-cite key="salimans2016improved"></d-cite> and can lead to mode collapse <d-cite key="metz2017unrolled"></d-cite>.

In this blog post, I will introduce another way to represent probability distributions that may circumvent several of these limitations. The key idea is to model *the gradient of the log probability density function*, a quantity often known as the (Stein) **score function** <d-cite key="chwialkowski16,liu2016kernelized"></d-cite>. Such **score-based models** are not required to have a tractable normalizing constant, and can be directly learned by **score matching** <d-cite key="hyvarinen2005estimation,vincent2011connection"></d-cite>.

<div class="l-page">
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/score_contour.jpg" style="display: block; width: max(30%, 200px); margin-left: auto; margin-right: auto;">
   <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Score function (the vector field) and density function (contours) of a mixture of two Gaussians.  </figcaption>
</div>

Score-based models have achieved state-of-the-art performance on many downstream tasks and applications. These tasks include, among others, image generation <d-cite key="song2019generative,song2020improved,ho2020denoising,song2021scorebased,dhariwal2021diffusion,ho2021cascaded"></d-cite> (Yes, better than GANs!), audio synthesis <d-cite key="chen2021wavegrad,kong2021diffwave,popov2021grad"></d-cite>, shape generation<d-cite key="ShapeGF"></d-cite>, and music generation<d-cite key="mittal2021symbolic"></d-cite>. Moreover, score-based models have connections to [normalizing flow models](https://blog.evjang.com/2018/01/nf1.html), therefore allowing exact likelihood computation and representation learning. Additionally, modeling and estimating scores facilitates [inverse problem](https://en.wikipedia.org/wiki/Inverse_problem#:~:text=An%20inverse%20problem%20in%20science,measurements%20of%20its%20gravity%20field) solving, with applications such as image inpainting <d-cite key="song2019generative,song2021scorebased"></d-cite>, image colorization <d-cite key="song2021scorebased"></d-cite>, [compressive sensing](https://en.wikipedia.org/wiki/Compressed_sensing), and medical image reconstruction (e.g., CT, MRI) <d-cite key="jalal2021robust"></d-cite>.

<div class='l-body'>
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/ffhq_samples.jpg">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> 1024 x 1024 samples generated from score-based models <d-cite key="song2021scorebased"></d-cite> </figcaption>
</div>

This post aims to show you the motivation and intuition of score-based generative modeling, as well as its basic concepts, properties and applications.


## The score function, score-based models, and score matching

Suppose we are given a dataset $$ \{\mathbf{x}_1, \mathbf{x}_2, \cdots, \mathbf{x}_N\} $$, where each point is drawn independently from an underlying data distribution $$p(\mathbf{x})$$. Given this dataset, the goal of generative modeling is to fit a model to the data distribution such that we can synthesize new data points at will by sampling from the distribution. 

In order to build such a generative model, we first need a way to represent a probability distribution. One such way, as in likelihood-based models, is to directly model the [probability density function](https://en.wikipedia.org/wiki/Probability_density_function) (p.d.f.) or [probability mass function](https://en.wikipedia.org/wiki/Probability_mass_function) (p.m.f.). Let $$f_\theta(\mathbf{x}) \in \mathbb{R} $$ be a real-valued function parameterized by a learnable parameter $$\theta$$. We can define a p.d.f. <d-footnote> Hereafter we only consider probability density functions. Probability mass functions are similar. </d-footnote> via
\begin{align}
p_\theta(\mathbf{x}) = \frac{e^{-f_\theta(\mathbf{x})}}{Z_\theta}, \label{ebm}
\end{align}
where $$Z_\theta > 0$$ is a normalizing constant dependent on $$\theta$$, such that $$\int p_\theta(\mathbf{x}) \textrm{d} \mathbf{x} = 1$$. Here the function $$f_\theta(\mathbf{x})$$ is often called an unnormalized probabilistic model, or energy-based model <d-cite key="song2021train"></d-cite>.

We can train $$p_\theta(\mathbf{x})$$ by maximizing the log-likelihood of the data
\begin{align}
\max_\theta \sum_{i=1}^N \log p_\theta(\mathbf{x}_i). \label{mle}
\end{align}
However, equation \eqref{mle} requires $$p_\theta(\mathbf{x})$$ to be a normalized probability density function. This is undesirable because in order to compute $$p_\theta(\mathbf{x})$$, we must evaluate the normalizing constant $$Z_\theta$$—a typically intractable quantity for any general $$f_\theta(\mathbf{x})$$. Thus to make maximum likelihood training feasible, likelihood-based models must either restrict their model architectures (e.g., causal convolutions in autoregressive models, invertible networks in normalizing flow models) to make $$Z_\theta$$ tractable, or approximate the normalizing constant (e.g., variational inference in VAEs, or MCMC sampling used in contrastive divergence<d-cite key="hinton2002training"></d-cite>) which may be computationally expensive.

By modeling the score function instead of the density function, we can sidestep the difficulty of intractable normalizing constants. The **score function** of a distribution $$p(\mathbf{x})$$ is defined as 
\begin{equation} 
\nabla_\mathbf{x} \log p(\mathbf{x}), \notag
\end{equation} 
and a model for the score function is called a **score-based model** <d-cite key="song2019generative"></d-cite>, which we denote as $$\mathbf{s}_\theta(\mathbf{x})$$. The score-based model is learned such that $$\mathbf{s}_\theta(\mathbf{x}) \approx \nabla_\mathbf{x} \log p(\mathbf{x})$$, and can be parameterized without worrying about the normalizing constant. For example, we can easily parameterize a score-based model with the energy-based model defined in equation \eqref{ebm} , via

$$
\begin{equation}
  \mathbf{s}_\theta (\mathbf{x}) = \nabla_{\mathbf{x}} \log p_\theta (\mathbf{x} ) = -\nabla_{\mathbf{x}}  f_\theta (\mathbf{x}) - \underbrace{\nabla_\mathbf{x} \log Z_\theta}_{=0} = -\nabla_\mathbf{x} f_\theta(\mathbf{x}).
\end{equation}
$$

Note that the score-based model $$\mathbf{s}_\theta(\mathbf{x})$$ is independent of the normalizing constant $$Z_\theta$$ ! This significantly expands the family of models that we can tractably use, since we don't need any special architectures to make the normalizing constant tractable.
<div class="row l-body">
	<div class="col-sm">
	  <img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/ebm.gif">
   <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Parameterizing probability density functions. No matter how you change the model family and parameters, it has to be normalized (area under the curve must integrate to one). </figcaption>
	</div>
	<div class="col-sm">
  <img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/score.gif">
   <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px">Parameterizing score functions. No need to worry about normalization. </figcaption>
  </div>
</div>
Similar to likelihood-based models, we can train score-based models by minimizing the **Fisher divergence** <d-footnote>Fisher divergence is typically between two distributions p and q, defined as $$\begin{equation}
\mathbb{E}_{p(\mathbf{x})}[\| \nabla_\mathbf{x} \log p(\mathbf{x}) - \nabla_\mathbf{x}\log q(\mathbf{x})  \|_2^2].
\end{equation}$$ Here we slightly abuse the term as the name of a closely related expression for score-based models. </d-footnote> between the model and the data distributions, defined as
$$
\begin{equation}
\mathbb{E}_{p(\mathbf{x})}[\| \nabla_\mathbf{x} \log p(\mathbf{x}) - \mathbf{s}_\theta(\mathbf{x})  \|_2^2]\label{fisher}
\end{equation}
$$

Intuitively, the Fisher divergence compares the squared $$\ell_2$$ distance between the ground-truth data score and the score-based model. Directly computing this divergence, however, is infeasible because it requires access to the unknown data score $$\nabla_\mathbf{x} \log p(\mathbf{x})$$. Fortunately, there exists a family of methods called **score matching** <d-footnote>Commonly used score matching methods include denoising score matching <d-cite key="vincent2011connection"></d-cite> and sliced score matching <d-cite key="song2019sliced"></d-cite>. Here is an introduction to <a href="{{ site.baseurl }}/blog/2019/ssm/">score matching and sliced score matching</a>. </d-footnote> <d-cite key="hyvarinen2005estimation,vincent2011connection,song2019sliced"></d-cite> that minimize the Fisher divergence without knowledge of the ground-truth data score. Score matching objectives can directly be estimated on a dataset and optimized with stochastic gradient descent, analogous to the log-likelihood objective for training likelihood-based models (with known normalizing constants). We can train the score-based model by minimizing a score matching objective, **without requiring adversarial optimization**.

Additionally, using the score matching objective gives us a considerable amount of modeling flexibility. The Fisher divergence itself does not require $$\mathbf{s}_\theta(\mathbf{x})$$ to be an actual score function of any normalized distribution---it simply compares the $$\ell_2$$ distance between the ground-truth data score and the score-based model, with no additional assumptions on the form of $$\mathbf{s}_\theta(\mathbf{x})$$. In fact, the only requirement on the score-based model is that it should be a vector-valued function with the same input and output dimensionality, which is easy to satisfy in practice.

As a brief summary, we can represent a distribution by modeling its score function, which can be estimated by training a score-based model of free-form architectures with score matching.

## Langevin dynamics

Once we have trained a score-based model $$\mathbf{s}_\theta(\mathbf{x}) \approx \nabla_\mathbf{x} \log p(\mathbf{x})$$, we can use an iterative procedure called [**Langevin dynamics**](https://en.wikipedia.org/wiki/Metropolis-adjusted_Langevin_algorithm)<d-cite key="parisi1981correlation,grenander1994representations"></d-cite> to draw samples from it.

Langevin dynamics provides an MCMC procedure to sample from a distribution $$p(\mathbf{x})$$ using only its score function $$\nabla_\mathbf{x} \log p(\mathbf{x})$$. Specifically, it initializes the chain from an arbitrary prior distribution $$\mathbf{x}_0 \sim \pi(\mathbf{x})$$, and then iterates the following

$$
\begin{align}
\mathbf{x}_{i+1} \gets \mathbf{x}_i + \epsilon \nabla_\mathbf{x} \log p(\mathbf{x}) + \sqrt{2\epsilon}~ \mathbf{z}_i, \quad i=0,1,\cdots, K, \label{langevin}
\end{align}
$$

where $$\mathbf{z}_i \sim \mathcal{N}(0, I)$$. When $$\epsilon \to 0$$ and $$K \to \infty$$, $$\mathbf{x}_K$$ obtained from the procedure in \eqref{langevin} converges to a sample from $$p(\mathbf{x})$$ under some regularity conditions. In practice, the error is negligible when $$\epsilon$$ is sufficiently small and $$K$$ is sufficiently large. 

<div class="l-page">
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/langevin.gif" style="display: block; width: max(30%, 300px); margin-left: auto; margin-right: auto;">
   <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;">Using Langevin dynamics to sample from a mixture of two Gaussians.  </figcaption>
</div>

Note that Langevin dynamics accesses $$p(\mathbf{x})$$ only through $$\nabla_\mathbf{x} \log p(\mathbf{x})$$. Since $$\mathbf{s}_\theta(\mathbf{x}) \approx \nabla_\mathbf{x} \log p(\mathbf{x})$$, we can produce samples from our score-based model $$\mathbf{s}_\theta(\mathbf{x})$$ by plugging it into equation \eqref{langevin}.

## Naive score-based generative modeling and its pitfalls

So far, we've discussed how to train a score-based model with score matching, and then produce samples via Langevin dynamics. However, this naive approach has had limited success in practice—we’ll talk about some pitfalls of score matching that received little attention in prior works<d-cite key="song2019generative"></d-cite>.
<div class='l-body'>
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/smld.jpg">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Score-based generative modeling with score matching + Langevin dynamics. </figcaption>
</div>
The key challenge is the fact that the estimated score functions are inaccurate in low density regions, where few data points are available for computing the score matching objective. This is expected as score matching minimizes the Fisher divergence 

$$
\mathbb{E}_{p(\mathbf{x})}[\| \nabla_\mathbf{x} \log p(\mathbf{x}) - \mathbf{s}_\theta(\mathbf{x})  \|_2^2] = \int p(\mathbf{x}) \| \nabla_\mathbf{x} \log p(\mathbf{x}) - \mathbf{s}_\theta(\mathbf{x})  \|_2^2 \mathrm{d}\mathbf{x}.
$$

Since the $$\ell_2$$ differences between the true data score function and score-based model are weighted by $$p(\mathbf{x})$$, they are largely ignored in low density regions where $$p(\mathbf{x})$$ is small. This behavior can lead to subpar results, as illustrated by the figure below:

<div class='l-body'>
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/pitfalls.jpg">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Estimated scores are only accurate in high density regions. </figcaption>
</div>

When sampling with Langevin dynamics, our initial sample is highly likely in low density regions when data reside in a high dimensional space. Therefore, having an inaccurate score-based model will derail Langevin dynamics from the very beginning of the procedure, preventing it from generating high quality samples that are representative of the data.

## Score-based generative modeling with multiple noise perturbations

How can we bypass the difficulty of accurate score estimation in regions of low data density? Our solution is to **perturb** data points with noise and train score-based models on the noisy data points instead. When the noise magnitude is sufficiently large, it can populate low data density regions to improve the accuracy of estimated scores. For example, here is what happens when we perturb a mixture of two Gaussians perturbed by additional Gaussian noise.

<div class='l-body'>
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/single_noise.jpg">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Estimated scores are accurate everywhere for the noise-perturbed data distribution due to reduced low data density regions. </figcaption>
</div>

Yet another question remains: how do we choose an appropriate noise scale for the perturbation process? Larger noise can obviously cover more low density regions for better score estimation, but it over-corrupts the data and alters it significantly from the original distribution. Smaller noise, on the other hand, causes less corruption of the original data distribution, but does not cover the low density regions as well as we would like.

To achieve the best of both worlds, we use multiple scales of noise perturbations simultaneously <d-cite key="song2019generative,song2020improved"></d-cite>. Suppose we always perturb the data with isotropic Gaussian noise, and let there be a total of $$L$$ increasing standard deviations $$\sigma_1 < \sigma_2 < \cdots < \sigma_L$$. We first perturb the data distribution $$p(\mathbf{x})$$ with each of the Gaussian noise $$\mathcal{N}(0, \sigma_i^2 I), i=1,2,\cdots,L$$ to obtain a noise-perturbed distribution 

$$
p_{\sigma_i}(\mathbf{x}) = \int p(\mathbf{y}) \mathcal{N}(\mathbf{x}; \mathbf{y}, \sigma_i^2 I) \mathrm{d} \mathbf{y}.
$$

Note that we can easily draw samples from $$p_{\sigma_i}(\mathbf{x})$$ by sampling $$\mathbf{x} \sim p(\mathbf{x})$$ and computing $$\mathbf{x} + \sigma_i \mathbf{z}$$, with $$\mathbf{z} \sim \mathcal{N}(0, I)$$.

Next, we estimate the score function of each noise-perturbed distribution, $$\nabla_\mathbf{x} \log p_{\sigma_i}(\mathbf{x})$$, by training a **Noise Conditional Score-Based Model** $$\mathbf{s}_\theta(\mathbf{x}, i)$$ (also called a Noise Conditional Score Network, or NCSN<d-cite key="song2019generative,song2020improved,song2021scorebased"></d-cite>, when parameterized with a neural network) with score matching, such that $$\mathbf{s}_\theta(\mathbf{x}, i) \approx \nabla_\mathbf{x} \log p_{\sigma_i}(\mathbf{x})$$ for all $$i= 1, 2, \cdots, L$$.

<div class='l-body'>
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/multi_scale.jpg">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> We apply multiple scales of Gaussian noise to perturb the data distribution (<strong>first row</strong>), and jointly estimate the score functions for all of them (<strong>second row</strong>).</figcaption>
</div>

<div class='l-body'>
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/duoduo.jpg">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px"> Perturbing an image with multiple scales of Gaussian noise.</figcaption>
</div>



The training objective for $$\mathbf{s}_\theta(\mathbf{x}, i)$$ is a weighted sum of Fisher divergences for all noise scales. In particular, we use the objective below:

$$
\begin{equation}
\sum_{i=1}^L \lambda(i) \mathbb{E}_{p_{\sigma_i}(\mathbf{x})}[\| \nabla_\mathbf{x} \log p_{\sigma_i}(\mathbf{x}) - \mathbf{s}_\theta(\mathbf{x}, i)  \|_2^2],\label{ncsn_obj}
\end{equation}
$$

where $$\lambda(i) \in \mathbb{R}_{>0}$$ is a positive weighting function, often chosen to be $$\lambda(i) = \sigma_i^2$$. The objective \eqref{ncsn_obj} can be optimized with score matching, exactly as in optimizing the naive (unconditional) score-based model $$\mathbf{s}_\theta(\mathbf{x})$$.

After training our noise-conditional score-based model $$\mathbf{s}_\theta(\mathbf{x}, i)$$, we can produce samples from it by running Langevin dynamics for $$i = L, L-1, \cdots, 1$$ in sequence. This method is called **annealed Langevin dynamics** (defined by Algorithm 1 in <d-cite key="song2019generative"></d-cite>, and improved by <d-cite key="song2020improved,jolicoeur-martineau2021adversarial"></d-cite>), since the noise scale $$\sigma_i$$ decreases (anneals) gradually over time. 
<div class="l-body">
	<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/ald.gif">
	<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Annealed Langevin dynamics combine a sequence of Langevin chains with gradually decreasing noise scales. </figcaption>
</div>

<div class="container">
<div class="row l-body">	
		<div class="col-sm">
		<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/celeba_large.gif">
	</div>
		<div class="col-sm">
		<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/cifar10_large.gif">
	</div>
</div>
<div class="row l-body">
	<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Annealed Langevin dynamics for the Noise Conditional Score Network (NCSN) model (from ref.<d-cite key="song2019generative"></d-cite>) trained on CelebA (<strong>left</strong>) and CIFAR-10 (<strong>right</strong>). We can start from unstructured noise, modify images according to the scores, and generate nice samples. The method achieved state-of-the-art Inception score on CIFAR-10 at its time. </figcaption>
</div>
</div>

Here are some practical recommendations for tuning score-based generative models with multiple noise scales: 

* Choose $$\sigma_1 < \sigma_2 < \cdots < \sigma_L$$ as a [geometric progression](https://en.wikipedia.org/wiki/Geometric_progression#:~:text=In%20mathematics%2C%20a%20geometric%20progression,number%20called%20the%20common%20ratio.), with $$\sigma_1$$ being sufficiently small and $$\sigma_L$$ comparable to the maximum pairwise distance between all training data points <d-cite key="song2020improved"></d-cite>. $$L$$ is typically on the order of hundreds or thousands.

* Parameterize the score-based model $$\mathbf{s}_\theta(\mathbf{x}, i)$$ with U-Net skip connections <d-cite key="song2019generative,ho2020denoising"></d-cite>.

* Apply exponential moving average on the weights of the score-based model when used at test time <d-cite key="song2020improved,ho2020denoising"></d-cite>.

With such best practices, we are able to generate high quality image samples with comparable quality to GANs on various datasets, such as below:

<div class="l-body">
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/ncsnv2.jpg">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Samples from the NCSNv2<d-cite key="song2020improved"></d-cite> model. From left to right: FFHQ 256x256, LSUN bedroom 128x128, LSUN tower 128x128, LSUN church_outdoor 96x96, and CelebA 64x64.</figcaption>
</div>

## Score-based generative modeling with stochastic differential equations (SDEs)

As we already discussed, adding multiple noise scales is critical to the success of score-based generative models. By generalizing the number of noise scales to infinity <d-cite key="song2021scorebased"></d-cite>, we obtain not only **higher quality samples**, but also, among others, **exact log-likelihood computation**, and **controllable generation for inverse problem solving**.

In addition to this introduction, we have tutorials written in [Google Colab](https://colab.research.google.com/) to provide a step-by-step guide for training a toy model on MNIST. We also have more advanced code repositories that provide full-fledged implementations for large scale applications.

| Link | Description|
|:----:|:-----|
|[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1SeXMpILhkJPjXUaesvzEhc3Ke6Zl_zxJ?usp=sharing)  | Tutorial of score-based generative modeling with SDEs in JAX + FLAX |
|[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1dRR_0gNRmfLtPavX2APzUggBuXyjWW55?usp=sharing)  | Load our pretrained checkpoints and play with sampling, likelihood computation, and controllable synthesis (JAX + FLAX)|
|[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/120kYYBOVa1i0TD85RjlEkFjaWDxSFUx3?usp=sharing)| Tutorial of score-based generative modeling with SDEs in PyTorch |
|[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/17lTrPLTt_0EDXa4hkbHmbAFQEkpRDZnh?usp=sharing) | Load our pretrained checkpoints and play with sampling, likelihood computation, and controllable synthesis (PyTorch) |
|[Code in JAX](https://github.com/yang-song/score_sde) | Score SDE codebase in JAX + FLAX |
|[Code in PyTorch](https://github.com/yang-song/score_sde_pytorch)| Score SDE codebase in PyTorch |

### Perturbing data with an SDE
When the number of noise scales approaches infinity, we essentially perturb the data distribution with continuously growing levels of noise. In this case, the noise perturbation procedure is a continuous-time [stochastic process](https://en.wikipedia.org/wiki/Stochastic_process#:~:text=A%20stochastic%20process%20is%20defined,measurable%20with%20respect%20to%20some), as demonstrated below

<div class="l-body">
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/perturb_vp.gif">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Perturbing data to noise with a continuous-time stochastic process.</figcaption>
</div>

How can we represent a stochastic process in a concise way? Many stochastic processes ([diffusion processes](https://en.wikipedia.org/wiki/Diffusion_process) in particular) are solutions of stochastic differential equations (SDEs). In general, an SDE possesses the following form:

$$
\begin{align}
\mathrm{d}\mathbf{x} = \mathbf{f}(\mathbf{x}, t) \mathrm{d}t + g(t) \mathrm{d} \mathbf{w},\label{sde}
\end{align}
$$

where $$\mathbf{f}(\cdot, t): \mathbb{R}^d \to \mathbb{R}^d$$ is a vector-valued function called the drift coefficient, $$g(t)\in \mathbb{R}$$ is a real-valued function called the diffusion coefficient, $$\mathbf{w}$$ denotes a standard [Brownian motion](https://en.wikipedia.org/wiki/Brownian_motion), and $$\mathrm{d} \mathbf{w}$$ can be viewed as infinitesimal white noise. The solution of a stochastic differential equation is a continuous collection of random variables $$\{ \mathbf{x}(t) \}_{t\in [0, T]}$$. These random variables trace stochastic trajectories as the time index $$t$$ grows from the start time $$0$$ to the end time $$T$$. Let $$p_t(\mathbf{x})$$ denote the (marginal) probability density function of $$\mathbf{x}(t)$$. Here $$t \in [0, T]$$ is analogous to $$i = 1, 2, \cdots, L$$ when we had a finite number of noise scales, and $$p_t(\mathbf{x})$$ is analogous to $$p_{\sigma_i}(\mathbf{x})$$. Clearly, $$p_0(\mathbf{x}) = p(\mathbf{x})$$ is the data distribution since no perturbation is applied to data at $$t=0$$. After perturbing $$p(\mathbf{x})$$ with the stochastic process for a sufficiently long time $$T$$, $$p_T(\mathbf{x})$$ becomes close to a tractable noise distribution $$\pi(\mathbf{x})$$, called a **prior distribution**. We note that $$p_T(\mathbf{x})$$ is analogous to $$p_{\sigma_L}(\mathbf{x})$$ in the case of finite noise scales, which corresponds to applying the largest noise perturbation $$\sigma_L$$ to the data.

The SDE in \eqref{sde} is **hand designed**, similarly to how we hand-designed $$\sigma_1 < \sigma_2 < \cdots < \sigma_L$$ in the case of finite noise scales. There are numerous ways to add noise perturbations, and the choice of SDEs is not unique. For example, the following SDE

$$
\begin{align}
\mathrm{d}\mathbf{x} = e^{t} \mathrm{d} \mathbf{w}
\end{align}
$$

perturbs data with a Gaussian noise of mean zero and exponentially growing variance, which is analogous to perturbing data with $$\mathcal{N}(0, \sigma_1^2 I), \mathcal{N}(0, \sigma_2^2 I), \cdots, \mathcal{N}(0, \sigma_L^2 I)$$ when $$\sigma_1 < \sigma_2 < \cdots < \sigma_L$$ is a [geometric progression](https://en.wikipedia.org/wiki/Geometric_progression#:~:text=In%20mathematics%2C%20a%20geometric%20progression,number%20called%20the%20common%20ratio.). Therefore, the SDE should be viewed as part of the model, much like $$\{\sigma_1, \sigma_2, \cdots, \sigma_L\}$$. In <d-cite key="song2021scorebased"></d-cite>, we provide three SDEs that generally work well for images: the Variance Exploding SDE (VE SDE), the Variance Preserving SDE (VP SDE), and the sub-VP SDE.

### Reversing the SDE for sample generation

Recall that with a finite number of noise scales, we can generate samples by reversing the perturbation process with **annealed Langevin dynamics**, i.e., sequentially sampling from each noise-perturbed distribution using Langevin dynamics. For infinite noise scales, we can analogously reverse the perturbation process for sample generation by using the reverse SDE.

<div class="l-body">
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/denoise_vp.gif">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Generate data from noise by reversing the perturbation procedure.</figcaption>
</div>

Importantly, any SDE has a corresponding reverse SDE <d-cite key="anderson1982reverse"></d-cite>, whose closed form is given by

$$
\begin{equation}
\mathrm{d}\mathbf{x} = [\mathbf{f}(\mathbf{x}, t) - g^2(t) \nabla_\mathbf{x} \log p_t(\mathbf{x})]\mathrm{d}t + g(t) \mathrm{d} \mathbf{w}.\label{rsde}
\end{equation}
$$

Here $$\mathrm{d} t$$ represents a negative infinitesimal time step, since the SDE \eqref{rsde} needs to be solved backwards in time (from $$t=T$$ to $$t = 0$$). In order to compute the reverse SDE, we need to estimate $$\nabla_\mathbf{x} \log p_t(\mathbf{x})$$, which is exactly the **score function** of $$p_t(\mathbf{x})$$.


<div class="l-body">
<img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/sde_schematic.jpg">
<figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Solving a reverse SDE yields a score-based generative model. Transforming data to a simple noise distribution can be accomplished with an SDE. It can be reversed to generate samples from noise if we know the score of the distribution at each intermediate time step. </figcaption>
</div>

### Estimating the reverse SDE with score-based models and score matching

Solving the reverse SDE requires us to know the terminal distribution $$p_T(\mathbf{x})$$, and the score function $$\nabla_\mathbf{x} \log p_t(\mathbf{x})$$. By design, the former is close to the prior distribution $$\pi(\mathbf{x})$$ which is fully tractable. In order to estimate $$\nabla_\mathbf{x} \log p_t(\mathbf{x})$$, we train a **Time-Dependent Score-Based Model** $$\mathbf{s}_\theta(\mathbf{x}, t)$$, such that $$\mathbf{s}_\theta(\mathbf{x}, t) \approx \nabla_\mathbf{x} \log p_t(\mathbf{x})$$. This is analogous to the noise-conditional score-based model $$\mathbf{s}_\theta(\mathbf{x}, i)$$ used for finite noise scales, trained such that $$\mathbf{s}_\theta(\mathbf{x}, i) \approx \nabla_\mathbf{x} \log p_{\sigma_i}(\mathbf{x})$$.

Our training objective for $$\mathbf{s}_\theta(\mathbf{x}, t)$$ is a continuous weighted combination of Fisher divergences, given by

$$
\begin{equation}
\mathbb{E}_{t \in \mathcal{U}(0, T)}\mathbb{E}_{p_t(\mathbf{x})}[\lambda(t) \| \nabla_\mathbf{x} \log p_t(\mathbf{x}) - \mathbf{s}_\theta(\mathbf{x}, t) \|_2^2],
\end{equation}
$$

where $$\mathcal{U}(0, T)$$ denotes a uniform distribution over the time interval $$[0, T]$$, and $$\lambda: \mathbb{R} \to \mathbb{R}_{>0}$$ is a positive weighting function. Typically we use $$\lambda(t) \propto 1/ \mathbb{E}[\| \nabla_{\mathbf{x}(t)} \log p(\mathbf{x}(t) \mid \mathbf{x}(0))\|_2^2]$$ to balance the magnitude of different score matching losses across time.

As before, our weighted combination of Fisher divergences can be efficiently optimized with score matching methods, such as denoising score matching <d-cite key="vincent2011connection"></d-cite> and sliced score matching <d-cite key="song2019sliced"> </d-cite>. Once our score-based model $$\mathbf{s}_\theta(\mathbf{x}, t)$$ is trained to optimality, we can plug it into the expression of the reverse SDE in \eqref{rsde} to obtain an estimated reverse SDE.

$$
\begin{equation}
\mathrm{d}\mathbf{x} = [\mathbf{f}(\mathbf{x}, t) - g^2(t) \mathbf{s}_\theta(\mathbf{x}, t)]\mathrm{d}t + g(t) \mathrm{d} \mathbf{w}.
\end{equation}
$$

We can start with $$\mathbf{x}(T) \sim \pi$$, and solve the above reverse SDE to obtain a sample $$\mathbf{x}(0)$$. Let us denote the distribution of $$\mathbf{x}(0)$$ obtained in such way as $$p_\theta$$. When the score-based model $$\mathbf{s}_\theta(\mathbf{x}, t)$$ is well-trained, we have $$p_\theta \approx p_0$$, in which case $$\mathbf{x}(0)$$ is an approximate sample from the data distribution $$p_0$$.

When $$\lambda(t) = g^2(t)$$, we have an important connection between our weighted combination of Fisher divergences and the KL divergence from $$p_0$$ to $$p_\theta$$ under some regularity conditions <d-cite key="durkan2021maximum"></d-cite>:

$$
\begin{multline}
\operatorname{KL}(p_0(\mathbf{x})\|p_\theta(\mathbf{x})) \leq \frac{T}{2}\mathbb{E}_{t \in \mathcal{U}(0, T)}\mathbb{E}_{p_t(\mathbf{x})}[\lambda(t) \| \nabla_\mathbf{x} \log p_t(\mathbf{x}) - \mathbf{s}_\theta(\mathbf{x}, t) \|_2^2] \\+ \operatorname{KL}(p_T \mathrel\| \pi).
\end{multline}
$$

Due to this special connection to the KL divergence and the equivalence between minimizing KL divergences and maximizing likelihood for model training, we call $$\lambda(t) = g(t)^2$$ the **likelihood weighting function**. Using this likelihood weighting function, we can train score-based generative models to achieve very high likelihoods, comparable or even superior to state-of-the-art autoregressive models<d-cite key="durkan2021maximum"></d-cite>.


### How to solve the reverse SDE

By solving the estimated reverse SDE with numerical SDE solvers, we can simulate the reverse stochastic process for sample generation. Perhaps the simplest numerical SDE solver is the [Euler-Maruyama method](https://en.wikipedia.org/wiki/Euler%E2%80%93Maruyama_method). When applied to our estimated reverse SDE, it discretizes the SDE using finite time steps and small Gaussian noise. Specifically, it chooses a small negative time step $$\Delta t \approx 0$$, initializes $$t \gets T$$, and iterates the following procedure until $$t \approx 0$$:

$$
\begin{aligned}
\Delta \mathbf{x} &\gets [\mathbf{f}(\mathbf{x}, t) - g^2(t) \mathbf{s}_\theta(\mathbf{x}, t)]\Delta t + g(t) \sqrt{\vert \Delta t\vert }\mathbf{z}_t \\
\mathbf{x} &\gets \mathbf{x} + \Delta \mathbf{x}\\
t &\gets t + \Delta t,
\end{aligned}
$$

Here $$\mathbf{z}_t \sim \mathcal{N}(0, I) $$. The Euler-Maruyama method is qualitatively similar to Langevin dynamics—both update $$\mathbf{x}$$ by following score functions perturbed with Gaussian noise.

Aside from the Euler-Maruyama method, other numerical SDE solvers can be directly employed to solve the reverse SDE for sample generation, including, for example, [Milstein method](https://en.wikipedia.org/wiki/Milstein_method), and [stochastic Runge-Kutta methods](https://en.wikipedia.org/wiki/Runge%E2%80%93Kutta_method_(SDE)). 
In <d-cite key="song2021scorebased"></d-cite>, we provided a reverse diffusion solver similar to Euler-Maruyama, but more tailored for solving reverse-time SDEs. More recently, authors in <d-cite key="jolicoeur2021gotta"></d-cite> introduced adaptive step-size SDE solvers that can generate samples faster with better quality.

In addition, there are two special properties of our reverse SDE that allow for even more flexible sampling methods:
* We have an estimate of $$\nabla_\mathbf{x} \log p_t(\mathbf{x})$$ via our time-dependent score-based model $$\mathbf{s}_\theta(\mathbf{x}, t)$$.
* We only care about sampling from each marginal distribution $$p_t(\mathbf{x})$$. Samples obtained at different time steps can have arbitrary correlations and do not have to form a particular trajectory sampled from the reverse SDE.

As a consequence of these two properties, we can apply MCMC approaches to fine-tune the trajectories obtained from numerical SDE solvers. Specifically, we propose **Predictor-Corrector samplers**. The **predictor** can be any numerical SDE solver that predicts $$\mathbf{x}(t + \Delta t) \sim p_{t+\Delta t}(\mathbf{x})$$ from an existing sample $$\mathbf{x}(t) \sim p_t(\mathbf{x})$$. The **corrector** can be any MCMC procedure that solely relies on the score function, such as Langevin dynamics and Hamiltonian Monte Carlo. 

At each step of the Predictor-Corrector sampler, we first use the predictor to choose a proper step size $$\Delta t < 0$$, and then predict $$\mathbf{x}(t + \Delta t)$$ based on the current sample $$\mathbf{x}(t)$$. Next, we run several corrector steps to improve the sample $$\mathbf{x}(t + \Delta t)$$ according to our score-based model $$\mathbf{s}_\theta(\mathbf{x}, t + \Delta t)$$, so that $$\mathbf{x}(t + \Delta t)$$ becomes a higher-quality sample from $$p_{t+\Delta t}(\mathbf{x})$$. 

With Predictor-Corrector methods and better architectures of score-based models, we can achieve **state-of-the-art** sample quality on CIFAR-10 (measured in FID <d-cite key="FID"></d-cite> and Inception scores <d-cite key="salimans2016improved"></d-cite>), outperforming the best GAN model to date (StyleGAN2 + ADA <d-cite key="Karras2020ada"></d-cite>).

| Method | FID $$\downarrow$$ | Inception score $$\uparrow$$ |
| :------:|:-----:|:-----:|
| StyleGAN2 + ADA <d-cite key="Karras2020ada"></d-cite>| 2.92 | 9.83 |
| Ours <d-cite key="song2021scorebased"></d-cite> | **2.20** | **9.89** |

The sampling methods are also scalable for extremely high dimensional data. For example, it can successfully generate high fidelity images of resolution $$1024\times 1024$$.

<div class="l-body">
  <img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/ffhq_1024.jpeg">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> 1024 x 1024 samples from a score-based model trained on the FFHQ dataset.  </figcaption>
</div>

Some additional (uncurated) samples for other datasets (taken from this [GitHub repo](https://github.com/yang-song/score_sde)):

<div class="l-body">
  <img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/bedroom.jpeg">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> 256 x 256 samples on LSUN bedroom.  </figcaption>
</div>

<div class="l-body">
  <img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/celebahq_256.jpg">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> 256 x 256 samples on CelebA-HQ.  </figcaption>
</div>


### Probability flow ODE

Despite capable of generating high-quality samples, samplers based on Langevin MCMC and SDE solvers do not provide a way to compute the exact log-likelihood of score-based generative models. Below, we introduce a sampler based on ordinary differential equations (ODEs) that allow for exact likelihood computation.

In <d-cite key="song2021scorebased"></d-cite>, we show t is possible to convert any SDE into an ordinary differential equation (ODE) without changing its marginal distributions $$\{ p_t(\mathbf{x}) \}_{t \in [0, T]}$$. Thus by solving this ODE, we can sample from the same distributions as the reverse SDE. The corresponding ODE of an SDE is named **probability flow ODE** <d-cite key="song2021scorebased"></d-cite>, given by

$$
\begin{equation}
\mathrm{d} \mathbf{x} = \bigg[\mathbf{f}(\mathbf{x}, t) - \frac{1}{2}g^2(t) \nabla_\mathbf{x} \log p_t(\mathbf{x})\bigg] \mathrm{d}t.\label{prob_ode}
\end{equation}
$$

The following figure depicts trajectories of both SDEs and probability flow ODEs. Although ODE trajectories are noticeably smoother than SDE trajectories, they convert the same data distribution to the same prior distribution and vice versa, sharing the same set of marginal distributions $$\{ p_t(\mathbf{x}) \}_{t \in [0, T]}$$. In other words, trajectories obtained by solving the probability flow ODE have the same marginal distributions as the SDE trajectories.

<div class="l-body">
  <img class="img-fluid rounded z-depth-1" src="{{ site.baseurl }}/assets/img/score/teaser.jpg">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;">  We can map data to a noise distribution (the prior) with an SDE, and reverse this SDE for generative modeling. We can also reverse the associated probability flow ODE, which yields a deterministic process that samples from the same distribution as the SDE. Both the reverse-time SDE and probability flow ODE can be obtained by estimating score functions.  </figcaption>
</div>

This probability flow ODE formulation has several unique advantages.

<!-- #### Exact log-likelihood computation -->
When $$\nabla_\mathbf{x} \log p_t(\mathbf{x})$$ is replaced by its approximation $$\mathbf{s}_\theta(\mathbf{x}, t)$$, the probability flow ODE becomes a special case of a neural ODE<d-cite key="neural_ode"></d-cite>. In particular, it is an example of continuous normalizing flows<d-cite key="grathwohl2018scalable"></d-cite>, since the probability flow ODE converts a data distribution $$p_0(\mathbf{x})$$ to a prior noise distribution $$p_T(\mathbf{x})$$ (since it shares the same marginal distributions as the SDE) and is fully invertible.

As such, the probability flow ODE inherits all properties of neural ODEs or continuous normalizing flows, including exact log-likelihood computation. Specifically, we can leverage the instantaneous change-of-variable formula (Theorem 1 in <d-cite key="neural_ode"></d-cite>, Equation (4) in <d-cite key="grathwohl2018scalable"></d-cite>) to compute the unknown data density $$p_0$$ from the known prior density $$p_T$$ with numerical ODE solvers. 

In fact, our model achieves the **state-of-the-art** log-likelihoods on uniformly dequantized <d-footnote>It is typical for normalizing flow models to convert discrete images to continuous ones by adding small uniform noise to them. </d-footnote> CIFAR-10 images <d-cite key="song2021scorebased"></d-cite>, **even without maximum likelihood training**.

| Method | Negative log-likelihood (bits/dim) $$\downarrow$$ |
|:-----------:|:------------------------------------------------:|
| RealNVP | 3.49 |
| iResNet | 3.45 |
| Glow | 3.35 |
| FFJORD | 3.40 |
| Flow++ | 3.29 |
| Ours | **2.99** |

When training score-based models with the **likelihood weighting** we discussed before, and using **variational dequantization** to obtain likelihoods on discrete images, we can achieve comparable or even superior likelihood to the state-of-the-art autoregressive models (all without any data augmentation) <d-cite key="durkan2021maximum"></d-cite>.

| Method | Negative log-likelihood (bits/dim) $$\downarrow$$ on CIFAR-10 | Negative log-likelihood (bits/dim) $$\downarrow$$  on ImageNet 32x32|
|:-------------:|:-------------------------:|:----------------------------:|
| Sparse Transformer | **2.80** | - |
| Image Transformer | 2.90 | 3.77 | 
| Ours | 2.83 | **3.76** |


<!-- #### Manipulating latent representations

Similarly as in normalizing flow models, we can obtain a latent representation for any input by integrating the probability flow ODE from $$t=0$$ to $$t=T$$. We can then manipulate the latent space, and reconstruct the edited image by integrating the probability flow ODE backwards from $$t=T$$ to $$t=0$$. This can provide continuous interpolations between two images, as shown below

<div class="l-body">
  <img class="img-fluid rounded z-depth-1 center" src="{{ site.baseurl }}/assets/img/score/interpolation.png" style="display: block; margin-left: auto; margin-right: auto;">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;">  Interpolating between the top left figure and the bottom right figure by doing spherical linear interpolation in the latent space. </figcaption>
</div>

#### Uniquely identifiable encoding

But what sets apart our method from your typical normalizing flow models is that our latent representations are **uniquely identifiable** <d-cite key="roeder2020linear"></d-cite>. This means that any input will be uniquely mapped to the same latent code, given sufficient data, model capacity and optimization accuracy. By contrast, normalizing flow models may map the same input into different latent codes when using different model architectures or training with different random seeds.

Our latent codes are uniquely identifiable because the probability flow ODE in equation \eqref{prob_ode} does not have trainable parameters. Instead, the probability flow ODE, as well as latent codes obtained from it, is fully determined by the data distribution itself and the forward SDE. As long as $$\mathbf{s}_\theta(\mathbf{x}, t) \approx \nabla_\mathbf{x} \log p_t(\mathbf{x})$$, we will always have roughly the same latent code for the same input, no matter how $$\mathbf{s}_\theta(\mathbf{x}, t)$$ was parameterized or trained.

<div class="l-body">
  <img class="img-fluid rounded z-depth-1 center" src="{{ site.baseurl }}/assets/img/score/identifiability.jpg">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;">  Sanity check on identifiable encoding. We compare the first 100 dimensions of the latent code obtained for a random CIFAR-10 image for two different score-based models, where “Model A” and “Model B” are separately trained with different architectures. The two latent codes are very close to each other despite using different score-based models. </figcaption>
</div>

#### More efficient sample generation

We can obtain samples from the probability flow ODE by sampling from the prior noise distribution $$p_T(\mathbf{x})$$ and then integrating the ODE from $$t=T$$ to $$t=0$$. This procedure is exactly the same as sampling from continuous normalizing flow models. 

Just like in continuous normalizing flows, we can exploit highly-optimized adaptive ODE solvers to integrate the probability flow ODE for sample generation. These ODE solvers not only require fewer steps than numerical SDE solvers in many cases, but also allow trading off sample quality for sampling speed by tuning the numerical precision.

<div class="l-body">
  <img class="img-fluid rounded z-depth-1 center" src="{{ site.baseurl }}/assets/img/score/ode_sampling.png" style="display: block; margin-left: auto; margin-right: auto;">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;">  Probability flow ODE enables fast sampling with adaptive stepsizes as the numerical precision is varied (from left to right: 1e-1, 1e-3, 1e-5), and reduces the number of score function evaluations (NFE) without harming quality. In comparison, numerical SDE solvers may require 1000 NFE for generating samples of similar quality.</figcaption>
</div>

Although ODE solvers may be more efficient, numerical SDE solvers still have their own advantages. They typically provide samples of higher quality as measured by quantitative metrics like FID and Inception scores. They are also easier to incorporate in Predictor-Corrector methods, which can lead to even better sample quality. In addition, it is often easier to modulate their sampling procedure for controllable sample generation, a point on which we elaborate below.
-->

### Controllable generation for inverse problem solving

Score-based generative models are particularly suitable for solving inverse problems. At its core, inverse problems are same as Bayesian inference problems. Let $$\mathbf{x}$$ and $$\mathbf{y}$$ be two random variables, and suppose we know the forward process of generating $$\mathbf{y}$$ from $$\mathbf{x}$$, represented by the transition probability distribution $$p(\mathbf{y} \mid \mathbf{x})$$. The inverse problem is to compute $$p(\mathbf{x} \mid \mathbf{y})$$. From Bayes' rule, we have $$p(\mathbf{x} \mid \mathbf{y}) = p(\mathbf{x}) p(\mathbf{y} \mid \mathbf{x}) / \int p(\mathbf{x}) p(\mathbf{y} \mid \mathbf{x}) \mathrm{d} \mathbf{x}$$. This expression can be greatly simplified by taking gradients with respect to $$\mathbf{x}$$ on both sides, leading to the following Bayes' rule for score functions:

$$
\begin{equation}
\nabla_\mathbf{x} \log p(\mathbf{x} \mid \mathbf{y}) = \nabla_\mathbf{x} \log p(\mathbf{x}) + \nabla_\mathbf{x} \log p(\mathbf{y} \mid \mathbf{x}).\label{inverse_problem}
\end{equation}
$$

Through score matching, we can train a model to estimate the score function of the unconditional data distribution, i.e., $$\mathbf{s}_\theta(\mathbf{x}) \approx \nabla_\mathbf{x} \log p(\mathbf{x})$$. This will allow us to easily compute the posterior score function $$\nabla_\mathbf{x} \log p(\mathbf{x} \mid \mathbf{y})$$ from the known forward process $$p(\mathbf{y} \mid \mathbf{x})$$ via equation \eqref{inverse_problem}, and sample from it with Langevin-type sampling <d-cite key="song2021scorebased"></d-cite>.

<!--
However, we know that score-based generative models work best when trained on a sequence of noise-perturbed data distributions. How can we solve inverse problems when perturbing data with an SDE and training time-dependent score-based models? With a perturbation SDE, we can first convert the posterior distribution $$p(\mathbf{x} \mid \mathbf{y}) $$ to a known noise distribution $$p_T(\mathbf{x})$$ by setting $$p_0(\mathbf{x}) = p(\mathbf{x} \mid \mathbf{y})$$, and then reverse the stochastic process to convert a sample from the noise distribution $$\mathbf{x}(T) \sim p_T(\mathbf{x})$$ to a sample from the posterior distribution $$\mathbf{x}(0) \sim p_0(\mathbf{x}) = p(\mathbf{x} \mid \mathbf{y})$$. This procedure is illustrated below:

<div class="l-body">
  <img class="img-fluid rounded z-depth-1 center" src="{{ site.baseurl }}/assets/img/score/controllable_gen.png" style="display: block; margin-left: auto; margin-right: auto;">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> We perturb the posterior distribution to a noise prior with an SDE, and then reverse it to sample from the posterior distribution, thereby solving the inverse problem. </figcaption>
</div>

The reverse stochastic process is described by a **conditional reverse-time SDE** <d-cite key="song2021scorebased"></d-cite>:

$$
\begin{equation}
\mathrm{d} \mathbf{x} = [\mathbf{f}(\mathbf{x}, t) - g^2(t) \nabla_\mathbf{x} \log p_t(\mathbf{y} \mid \mathbf{x})] \mathrm{d} t + g(t) \mathrm{d} \mathbf{w}.
\end{equation}
$$

With the Bayes' rule for score functions, we can rewrite $$\nabla_\mathbf{x} \log p_t(\mathbf{x} \mid \mathbf{y})$$ as below

$$
\begin{equation}
\nabla_\mathbf{x} \log p_t(\mathbf{x} \mid \mathbf{y}) = \nabla_\mathbf{x} \log p_t(\mathbf{x}) + \nabla_\mathbf{x} \log p_t(\mathbf{y} \mid \mathbf{x}).
\end{equation}
$$

Here the first term is an unconditional time-dependent score function, which can be approximated by our time-dependent score-based model since $$\mathbf{s}_\theta(\mathbf{x}, t) \approx \nabla_\mathbf{x} \log p_t(\mathbf{x})$$. The second term $$\nabla_\mathbf{x} \log p_t(\mathbf{y} \mid \mathbf{x})$$ can often be obtained in two ways:
* Training a separate model. Since we know the forward process $$p(\mathbf{y} \mid \mathbf{x})$$, and we have training data from the data distribution $$\{\mathbf{x}_1, \mathbf{x}_2, \cdots, \mathbf{x}_N\} \stackrel{\text{i.i.d.}}{\sim} p_0(\mathbf{x})$$, we can generate samples $$\{ (\mathbf{x}_1, y_1), \cdots, (\mathbf{x}_N, y_N) \} \stackrel{\text{i.i.d.}}{\sim} p_0(\mathbf{x}) p(\mathbf{y} \mid \mathbf{x})$$. From our forward SDE, we also know how to perturb any clean data sample $$\mathbf{x}_i$$ to a noisy one $$\mathbf{x}_i(t) \sim p_t(\mathbf{x} \mid \mathbf{x}_i)$$. This means we can easily get samples $$\{ (\mathbf{x}_1(t), y_1), \cdots, (\mathbf{x}_N(t), y_N) \} \stackrel{\text{i.i.d.}}{\sim} p_t(\mathbf{x}) p_t(\mathbf{y} \mid \mathbf{x})$$ for any $$t$$, and use them to a train a model for $$p_t(\mathbf{y} \mid \mathbf{x})$$. One application of this approach is class-conditional image generation, where $$\mathbf{y}$$ represents a target class label. We can train a noise-conditional classifier $$q_\phi(\mathbf{y}, \mathbf{x}, t) \approx p_t(\mathbf{y} \mid \mathbf{x})$$ based on pairs of noise-perturbed images and the corresponding class label, and combine it with an unconditionally trained time-dependent score-based model $$\mathbf{s}_\theta(\mathbf{x}, t)$$ for conditional sample generation.


* Specified with domain knowledge. When the forward process is a linear model, i.e., $$p(\mathbf{y} \mid \mathbf{x}) = \delta(\mathbf{y} = M \mathbf{x})$$, where $$M$$ is a linear operator and $$\delta(\cdot)$$ represents a point mass, it is often possible to directly specify an approximation to $$\nabla_\mathbf{x} \log p_t(\mathbf{y} \mid \mathbf{x})$$. There are many examples of linear forward problems, including image inpainting and colorization. Please find a general approach to specify $$\nabla_\mathbf{x} \log p_t(\mathbf{y} \mid \mathbf{x})$$ for linear inverse problems in Appendix I.4 of ref.<d-cite key="song2021scorebased"></d-cite>, or ref.<d-cite key="kadkhodaie2020solving"></d-cite>.

-->

A recent work from UT Austin <d-cite key="jalal2021robust"></d-cite> has demonstrated that score-based generative models can be applied to solving inverse problems in medical imaging, such as accelerating magnetic resonance imaging (MRI). Concurrently in <d-cite key="song2022solving"></d-cite>, we demonstrated superior performance of score-based generative models not only on accelerated MRI, but also sparse-view computed tomography (CT). We were able to achieve comparable or even better performance than supervised or unrolled deep learning approaches, while being more robust to different measurement processes at test time.

Below we show some examples on solving inverse problems for computer vision. 
<div class="l-body">
  <img class="img-fluid rounded z-depth-1 center" src="{{ site.baseurl }}/assets/img/score/class_cond.png" style="display: block; margin-left: auto; margin-right: auto;">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Class-conditional generation with an unconditional time-dependent score-based model, and a pre-trained noise-conditional image classifier on CIFAR-10. </figcaption>
</div>

<div class="l-body">
  <img class="img-fluid rounded z-depth-1 center" src="{{ site.baseurl }}/assets/img/score/inpainting.png" style="display: block; margin-left: auto; margin-right: auto;">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Image inpainting with a time-dependent score-based model trained on LSUN bedroom. The leftmost column is ground-truth. The second column shows masked images (y in our framework). The rest columns show different inpainted images, generated by solving the conditional reverse-time SDE. </figcaption>
</div>

<div class="l-body">
  <img class="img-fluid rounded z-depth-1 center" src="{{ site.baseurl }}/assets/img/score/colorization.png" style="display: block; margin-left: auto; margin-right: auto;">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> Image colorization with a time-dependent score-based model trained on LSUN church_outdoor and bedroom. The leftmost column is ground-truth. The second column shows gray-scale images (y in our framework). The rest columns show different colorizedimages, generated by solving the conditional reverse-time SDE. </figcaption>
</div>

<div class="l-body">
  <img class="img-fluid rounded z-depth-1 center" src="{{ site.baseurl }}/assets/img/score/lincoln.png" style="display: block; margin-left: auto; margin-right: auto;">
  <figcaption style="text-align: center; margin-top: 10px; margin-bottom: 10px;"> We can even colorize gray-scale portrays of famous people in history (Abraham Lincoln) with a time-dependent score-based model trained on FFHQ. The image resolution is 1024 x 1024.</figcaption>
</div>

## Connection to diffusion models and others

I started working on score-based generative modeling since 2019, when I was trying hard to make score matching scalable for training deep energy-based models on high-dimensional datasets. My first attempt at this led to the method sliced score matching<d-cite key="song2019sliced"></d-cite>. Despite the scalability of sliced score matching for training energy-based models, I found to my surprise that Langevin sampling from those models fails to produce reasonable samples even on the MNIST dataset. I started investigating this issue and discovered three crucial improvements that can lead to extremely good samples: (1) perturbing data with multiple scales of noise, and training score-based models for each noise scale; (2) using a U-Net architecture (we used RefineNet since it is a modern version of U-Nets) for the score-based model; (3) applying Langevin MCMC to each noise scale and chaining them together. With those methods, I was able to obtain the state-of-the-art Inception Score on CIFAR-10 in <d-cite key="song2019generative"></d-cite> (even better than the best GANs!), and generate high-fidelity image samples of resolution up to $256\times 256$ in <d-cite key="song2020improved"></d-cite>. 

The idea of perturbing data with multiple scales of noise is by no means unique to score-based generative models though. It has been previously used in, for example, [simulated annealing](https://en.wikipedia.org/wiki/Simulated_annealing), annealed importance sampling<d-cite key="neal2001annealed"></d-cite>, diffusion probabilistic models<d-cite key="sohl2015deep"></d-cite>, infusion training<d-cite key="bordes2017learning"></d-cite>, and variational walkback<d-cite key="goyal2017variational"></d-cite> for generative stochastic networks<d-cite key="alain2016gsns"></d-cite>. Out of all these works, diffusion probabilistic modeling is perhaps the closest to score-based generative modeling. Diffusion probabilistic models are hierachical latent variable models first proposed by [Jascha](http://www.sohldickstein.com/) and his colleagues <d-cite key="sohl2015deep"></d-cite> in 2015, which generate samples by learning a variational decoder to reverse a discrete diffusion process that perturbs data to noise. Without awareness of this work, score-based generative modeling was proposed and motivated independently from a very different perspective. Despite both perturbing data with multiple scales of noise, the connection between score-based generative modeling and diffusion probabilistic modeling seemed superficial at that time, since the former is trained by score matching and sampled by Langevin dynamics, while the latter is trained by the evidence lower bound (ELBO) and sampled with a learned decoder.

In 2020, [Jonathan Ho](http://www.jonathanho.me/) and colleagues <d-cite key="ho2020denoising"></d-cite> significantly improved the empirical performance of diffusion probabilistic models and first unveiled a deeper connection to score-based generative modeling. They showed that the ELBO used for training diffusion probabilistic models is essentially equivalent to the weighted combination of score matching objectives used in score-based generative modeling. Moreover, by parameterizing the decoder as a sequence of score-based models with a U-Net architecture, they demonstrated for the first time that diffusion probabilistic models can also generate high quality image samples comparable or superior to GANs.

Inspired by their work, we further investigated the relationship between diffusion models and score-based generative models in an ICLR 2021 paper <d-cite key="song2021scorebased"></d-cite>. We found that the sampling method of diffusion probabilistic models can be integrated with annealed Langevin dynamics of score-based models to create a unified and more powerful sampler (the Predictor-Corrector sampler). By generalizing the number of noise scales to infinity, we further proved that score-based generative models and diffusion probabilistic models can both be viewed as discretizations to stochastic differential equations determined by score functions. This work bridges both score-based generative modeling and diffusion probabilistic modeling into a unified framework.

Collectively, these latest developments seem to indicate that both score-based generative modeling with multiple noise perturbations and diffusion probabilistic models are different perspectives of the same model family, much like how [wave mechanics](https://en.wikipedia.org/wiki/Wave_mechanics) and [matrix mechanics](https://en.wikipedia.org/wiki/Matrix_mechanics) are equivalent formulations of quantum mechanics in the history of physics<d-footnote>Goes without saying that the significance of score-based generative models/diffusion probabilistic models is in no way comparable to quantum mechanics.</d-footnote>. The perspective of score matching and score-based models allows one to calculate log-likelihoods exactly, solve inverse problems naturally, and is directly connected to energy-based models, Schrödinger bridges and optimal transport<d-cite key="de2021diffusion"></d-cite>. The perspective of diffusion models is naturally connected to VAEs, lossy compression, and can be directly incorporated with variational probabilistic inference. This blog post focuses on the first perspective, but I highly recommend interested readers to learn about the alternative perspective of diffusion models as well (see [a great blog by Lilian Weng](https://lilianweng.github.io/lil-log/2021/07/11/diffusion-models.html)). 

Many recent works on score-based generative models or diffusion probabilistic models have been deeply influenced by knowledge from both sides of research (see a [website](https://scorebasedgenerativemodeling.github.io/) curated by researchers at the University of Oxford). Despite this deep connection between score-based generative models and diffusion models, it is hard to come up with an umbrella term for the model family that they both belong to. Some colleagues in DeepMind propose to call them "Generative Diffusion Processes". It remains to be seen if this will be adopted by the community in the future.

## Concluding remarks

This blog post gives a detailed introduction to score-based generative models. We demonstrate that this new paradigm of generative modeling is able to produce high quality samples, compute exact log-likelihoods, and perform controllable generation for  inverse problem solving. It is a compilation of several papers we published in the past few years. Please visit them if you are interested in more details:

* [Yang Song\*, Sahaj Garg\*, Jiaxin Shi, and Stefano Ermon. *Sliced Score Matching: A Scalable Approach to Density and Score Estimation*. UAI 2019 (Oral)](https://arxiv.org/abs/1905.07088)

* [Yang Song, and Stefano Ermon. *Generative Modeling by Estimating Gradients of the Data Distribution*. NeurIPS 2019 (Oral)](https://arxiv.org/abs/1907.05600)

* [Yang Song, and Stefano Ermon. *Improved Techniques for Training Score-Based Generative Models*. NeurIPS 2020](https://arxiv.org/abs/2006.09011)

* [Yang Song, Jascha Sohl-Dickstein, Diederik P. Kingma, Abhishek Kumar, Stefano Ermon, and Ben Poole. *Score-Based Generative Modeling through Stochastic Differential Equations*. ICLR 2021 (Outstanding Paper Award)](https://arxiv.org/abs/2011.13456)

* [Yang Song\*, Conor Durkan\*, Iain Murray, and Stefano Ermon. *Maximum Likelihood Training of Score-Based Diffusion Models*. NeurIPS 2021 (Spotlight)](https://arxiv.org/abs/2101.09258)

* [Yang Song\*, Liyue Shen\*, Lei Xing, and Stefano Ermon. *Solving Inverse Problems in Medical Imaging with Score-Based Generative Models*. ICLR 2022](https://arxiv.org/abs/2111.08005)

For a list of works that have been influenced by score-based generative modeling, researchers at the University of Oxford have built a very useful (but necessarily incomplete) website: [https://scorebasedgenerativemodeling.github.io/](https://scorebasedgenerativemodeling.github.io/).

There are two major challenges of score-based generative models. First, the sampling speed is slow since it involves a large number of Langevin-type iterations. Second, it is inconvenient to work with discrete data distributions since scores are only defined on continuous distributions. 

The first challenge can be partially solved by using numerical ODE solvers for the  probability flow ODE with lower precision (a similar method, denoising diffusion implicit modeling, has been proposed in <d-cite key="song2021denoising"></d-cite>). It is also possible to learn a direct mapping from the latent space of probability flow ODEs to the image space, as shown in <d-cite key="luhman2021knowledge"></d-cite>. However, all such methods to date result in worse sample quality.

The second challenge can be addressed by learning an autoencoder on discrete data and performing score-based generative modeling on its continuous latent space <d-cite key="mittal2021symbolic,vahdat2021score"></d-cite>. Jascha's original work on diffusion models <d-cite key="sohl2015deep"></d-cite> also provides a discrete diffusion process for discrete data distributions, but its potential for large scale applications remains yet to be proven.

It is my conviction that these challenges will soon be solved with the joint efforts of the research community, and score-based generative models/ diffusion-based models will become one of the most useful tools for data generation, density estimation, inverse problem solving, and many other downstream tasks in machine learning.
