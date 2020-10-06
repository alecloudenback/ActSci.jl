# LifeContingencies.jl

[![Stable](https://img.shields.io/badge/docs-stable-blue.svg)](https://JuliaActuary.github.io/LifeContingencies.jl/stable/) 
[![Dev](https://img.shields.io/badge/docs-dev-blue.svg)](https://JuliaActuary.github.io/LifeContingencies.jl/dev/)
![](https://github.com/JuliaActuary/LifeContingencies.jl/workflows/CI/badge.svg)
[![codecov](https://codecov.io/gh/JuliaActuary/LifeContingencies.jl/branch/master/graph/badge.svg)](https://codecov.io/gh/JuliaActuary/LifeContingencies.jl)
[![lifecycle](https://img.shields.io/badge/LifeCycle-Developing-yellow)](https://www.tidyverse.org/lifecycle/)

LifeContingencies is a package enabling actuarial life contingent calculations.

## Features:

- Integration with other JuliaActuary packages such as [MortalityTables.jl](https://github.com/JuliaActuary/MortalityTables.jl)
- Fast calculations, with some parts utilizing parallel processing power automatically
- Use functions that look more like the math you are used to (e.g. `A`, `ä`)
with [Unicode support](https://docs.julialang.org/en/v1/manual/unicode-input/index.html)
- All of the power, speed, convenience, tooling, and ecosystem of Julia
- Flexible and modular modeling approach

## Package Overview

- Leverages [MortalityTables.jl](https://github.com/JuliaActuary/MortalityTables.jl) for
the mortality calculations
- Contains common insurance calculations such as:
    - `insurance(life)`: Whole life
    - `insurance(life,n)`: Term life for `n` years
    - `ä(life)`: Life contingent annuity due
    - `ä(life,n)`: Life contingent annuity due for `n` years
- Contains various commutation functions such as `D(x)`,`M(x)`,`C(x)`, etc.
- `SingleLife` and `JointLife` capable
- Interest rate mechanics via [`Yields.jl`](https://github.com/JuliaActuary/Yields.jl)
- More documentation available by clicking the DOCS badges at the top of this README

## Examples

###  Basic Functions
Calculate various items for a 30-year-old male nonsmoker using 2015 VBT base table and a 5% interest rate

```julia

using LifeContingencies
using MortalityTables
using Yields

tbls = MortalityTables.tables()
vbt2001 = tbls["2001 VBT Residual Standard Select and Ultimate - Male Nonsmoker, ANB"]
age = 30
life = SingleLife(
    mort = vbt2001.select[age],
    issue_age = age
)

lc = LifeContingency(
    life,
    Yields.Constant(0.05)
)


insurance(lc)          # Whole Life insurance
insurance(lc,10)       # 10 year term insurance
premium_net(lc)        # Net whole life premium 
V(lc,5)                # Net premium reserve for whole life insurance at time 5
ä(lc)                  # Whole life annuity due
ä(lc, 5)               # 5 year annuity due
...                    # and more!
```

### Net Premium for Term Policy with Stochastic rates
Use a stochastic interest rate calculation to price a term policy:

```julia
using LifeContingencies, MortalityTables
using Distributions

tbls = MortalityTables.tables()
vbt2001 = tbls["2001 VBT Residual Standard Select and Ultimate - Male Nonsmoker, ANB"]

# use an interest rate that's normally distirbuted
μ = 0.05
σ = 0.01

years = 100
int =   Yields.Forward(rand(Normal(μ,σ), years))

life = SingleLife(
    mort = vbt2001.select[30],
    issue_age = 30
)

lc = LifeContingency(
    life,
    int
)

term = 10
insurance(lc,term) # around 0.055
```
#### Extending example to use autocorrelated interest rates
You can use autocorrelated interest rates - substitute the following in the prior example
using the ability to self reference:

```julia
σ = 0.01
initial_rate = 0.05
vec = fill(initial_rate,years)

for i in 2:length(vec)
    vec[i] = rand(Normal(vec[i-1],σ))
end

int = Yields.Forward(vec)
```

### Premium comparison across Mortality Tables

Compare the cost of annual premium, whole life insurance between multiple tables visually:

```julia
using LifeContingencies, MortalityTables, Plots

tbls = MortalityTables.tables()
tables = [
    tbls["1980 CET - Male Nonsmoker, ANB"],
    tbls["2001 VBT Residual Standard Select and Ultimate - Male Nonsmoker, ANB"],
    tbls["2015 VBT Male Non-Smoker RR100 ANB"],
    ]

issue_ages = 30:90
int = Yields.Constant(0.05)

whole_life_costs = map(tables) do t
    map(issue_ages) do ia
        lc = LifeContingency(
                SingleLife(
                    mort=t.ultimate,
                    issue_age=ia
                ),
                int
            )

        premium_net(lc)

    end
end

plt = plot(ylabel="Annual Premium per unit", xlabel="Issue Age",
            legend=:topleft, legendfontsize=8,size=(800,600))
for (i,t) in enumerate(tables)
    plot!(plt,issue_ages,whole_life_costs[i], label="$(t.d.name)")
end
display(plt)
```
![Comparison of three different mortality tables' effect on insurance cost](https://user-images.githubusercontent.com/711879/85190836-cb539800-b281-11ea-96b0-e3f3eab59449.png)


### Joint Life

```julia
m1 = tbls["1986-92 CIA – Male Smoker, ANB"]
m2 = tbls["1986-92 CIA – Female Nonsmoker, ANB"]
l1 = SingleLife(mort = m1.ultimate, issue_age = 40)
l2 = SingleLife(mort = m2.ultimate, issue_age = 37)

jl = JointLife(lives=(l1, l2), contingency=LastSurvivor(), joint_assumption=Frasier())


insurance(jl)   # whole life insurance
...     # similar functions as shown in the first example above
```
## Commutation and Unexported Function shorthand

Because it's so common to use certain variables in your own code, LifeContingencies avoids exporting certain variables/functions so that it doesn't collide with your own usage. For example, you may find yourself doing something like:

```julia
a = ...
b = ...
resutl = b - a
```

If you imported `using LifeContingencies` and the package exported `a` (`annuity_immediate`) then you could have problems if you tried to do the above. To avoid this, we only export long-form functions like `annuity_immediate`. To utilize the shorthand, you can include them into your code's scope like so:

```julia
using LifeContingencies # brings all the default functions into your scope
using LifeContingencies: a, ä # also brings the short-form annuity functions into scope
```

**Or** you can do the following:

```julia
using LifeContingencies # brings all the default functions into your scope
... # later on in the code
LifeContingencies.ä(...) # utilize the unexported function with the module name
```
For more on module scoping, see the [Julia Manual section](https://docs.julialang.org/en/latest/manual/modules/#Summary-of-module-usage-1).

### Actuarial notation shorthand

```julia
V => reserve_premium_net
v => disc
A => insurance
ä => annuity_due
a => annuity_immediate
P => premium_net
ω => omega
```
### Commutation functions

```julia
l,
D,
M,
N,
C,
```

## References

- Life Insurance Mathematics, Gerber
- [Actuarial Mathematics and Life-Table Statistics, Slud](http://www2.math.umd.edu/~slud/s470/BookChaps/Chp6.pdf)
- [Commutation Functions, MacDonald](http://www.macs.hw.ac.uk/~angus/papers/eas_offprints/commfunc.pdf)
