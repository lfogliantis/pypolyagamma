# PyPólyaGamma

This is a Cython port of Jesse Windle's code at
https://github.com/jwindle/BayesLogit. It provides a
Python interface for efficiently sampling Pólya-gamma
random variates.

# Background
Pólya-gamma augmentation is a method of performing
fast and simple Bayesian inference in models with
Gaussian latent variables and count observations.
While such models are non-conjugate, if it has the
right form (specifically, if it is a Bernoulli, binomial,
or negative binomial with a logistic link function),
we can use introduce a set of Pólya-gamma
auxiliary variables that render it conditionally conjugate.
This facilitates fast Gibbs sampling algorithms on an
extended space of Gaussian latent variables
and Pólya-gamma auxiliary variables, where integrating out the
auxiliary variables leaves the original model intact.

Given the auxiliary variables, the latent Gaussian variables
have a Gaussian conditional distribution. Likewise, given
the Gaussian latent variables and the observed count data,
the auxiliary variables have a Pólya-gamma conditional distribution.
Thus, to implement the Gibbs sampling algorithm, we must be
able to efficiently sample Pólya-gamma random variates. This
library provides code to do exactly that.

This is illustrated in the figure below, which shows the posterior
under a simple Binomial model. Next, we'll show how to perform
Gibbs sampling for such a model.

![Marginals](https://raw.githubusercontent.com/slinderman/pypolyagamma/simplegsl/aux/marginals.png)

# Demo
Here's a simple example of inference in a binomial model
with `N=10` counts and probability `p=logistic(x)`, with
a standard normal prior on `x`.

First, sample a count from the model:
```python
from pypolyagamma import logistic, PyPolyaGamma

# Consider a simple binomial model with unknown probability
# Model the probability as the logistic of a scalar Gaussian.
N = 10
mu = 0.0
sigmasq = 1.0
x_true = npr.normal(mu, np.sqrt(sigmasq))
p_true = logistic(x_true)
y = npr.binomial(N, p_true)
```

Now we can run a Gibbs sampler to estimate the posterior
distribution of `x` given `y`.

```python
# Gibbs sample the posterior distribution p(x | y)
# Introduce PG(N,0) auxiliary variables to render
# the model conjugate.  First, initialize the PG
# sampler and the model parameters.
N_samples = 10000
pg = PyPolyaGamma(seed=0)
xs = np.zeros(N_samples)
omegas = np.ones(N_samples)

# Now run the Gibbs sampler
for i in range(1, N_samples):
    # Sample omega given x, y from its PG conditional
    omegas[i] = pg.pgdraw(N, xs[i-1])

    # Sample x given omega, y from its Gaussian conditional
    sigmasq_hat = 1./(1. / sigmasq + omegas[i])
    mu_hat = sigmasq_hat * (mu / sigmasq + (y - N / 2.))
    xs[i] = npr.normal(mu_hat, np.sqrt(sigmasq_hat))		
```

For this simple example, we can compute the true posterior
and compare the samples to the target density. 
![Binomial](https://raw.githubusercontent.com/slinderman/pypolyagamma/simplegsl/aux/binomial.png)

# Installation

This is the simplest way to install:

    git clone git@github.com:slinderman/pypolyagamma.git
    cd pypolyagamma
    pip install -e .

To check if it worked, run:

    python test/basic.py

If it prints `Tests passed!` then you're good to go!

Under the hood, the installer will download
[GSL](https://www.gnu.org/software/gsl/),
untar it, and place it in `deps/gsl`. It will then configure GSL and
compile the Pólya-gamma code along with the required GSL source files.
This way, you don't need GSL to be installed and available on your
library path. 

## Parallel sampling with OpenMP
By default, the simple installation above will not support
parallel sampling. If you are compiling with GNU `gcc` and `g++`,
you can enable OpenMP support with the flag:

    USE_OPENMP=True pip install -e .

Mac users: you can install `gcc` and `g++` with Homebrew. Just
make sure that they are your default compilers, e.g. by setting
the environment variables `CC` and `CXX` to point to the GNU versions
of `gcc` and `g++`, respectively. With Homebrew, these versions
will be in `/usr/local/bin` by default.

To sample in parallel, call the `pgdrawvpar` method:

    n = 10             # Number of variates to sample
    b = np.ones(n)     # Vector of shape parameters
    c = np.zeros(n)    # Vector of tilting parameters
    out = np.empty(n)  # Outputs

    # Construct a set of PolyaGamma objects for sampling
    nthreads = 8
    seeds = np.random.randint(2**16, size=nthreads)
    ppgs = [pypolyagamma.PyPolyaGamma(seed) for seed in seeds]

    # Sample in parallel
    pypolyagamma.pgdrawvpar(ppgs, b, c, out)

If you haven't installed with OpenMP, this function will
revert to the serial sampler. 