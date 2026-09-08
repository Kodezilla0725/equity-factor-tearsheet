# Methodology

Why the notebook makes the choices it makes. Every number below comes from the
run in section 11 of `Equity_Factor_Tearsheet.ipynb`: AAPL against KO, ARKK and
BRK-B, 2015-01-05 to 2026-07-31, 2,910 trading days, benchmark SPY. Section
numbers in the text refer to that notebook.

## 1. What a factor model is, and why run one

AAPL returned 24.68% a year over this window. That number says nothing about
where the return came from. Some of it came from being a stock at all while the
stock market rose. Some came from being a very large company rather than a small
one, an expensive one rather than a cheap one, a profitable one rather than a
marginal one. Whatever is left after those are accounted for is the part that
required AAPL specifically. A factor model is the arithmetic that splits the
return into those pieces. It is worth doing because the pieces have prices.
Broad market exposure costs a few basis points a year in an index fund and the
common tilts cost a little more. Only the leftover can justify anything above
that.

The Fama-French three-factor model regresses an asset's excess return, meaning
return minus the risk-free rate, on three portfolio return series: the market
excess return Mkt-RF, a size spread SMB, and a value spread HML. The five-factor
version adds RMW for profitability and CMA for investment conservatism. Each
slope is that asset's exposure to a factor. The intercept is the average daily
return the factors leave unexplained. R² says how much of the day-to-day
variation the factors account for, and it varies a lot by name: KO 0.36, AAPL
0.59, BRK-B 0.68, ARKK 0.81. For AAPL the FF3 model gets 0.56 and FF5 gets
0.591, so profitability and investment add about three points.

Section 3 pulls the daily FF3 and FF5 tables from the Ken French library and
keeps them separate rather than merging them, because SMB is built from a
different sort in each model and the two series are not interchangeable. Section
4 joins prices to the factor calendar with an inner join and never forward
fills, so a missing day drops out instead of being invented. Section 6 fits both
models on the same dates. A high R² means different things for different assets,
which is why `interpret()` in section 8 reports it as context rather than as a
verdict: for ARKK, a fund charging active fees, 0.81 raises the closet-index
question, while AAPL's 0.59 says only that a large stock moves with the market.
Two details of the setup are easy to misread. The market in the regression is
the Ken French Mkt-RF series and not SPY; SPY appears on the tearsheet only as
the line the growth chart is drawn against, and changing `benchmark` in `Config`
moves no regression coefficient at all. And everything runs on daily rather than
monthly data, which buys 2,910 observations instead of roughly 139 and makes the
betas precise, at the cost of residuals badly enough behaved to need the
treatment in the next section.

## 2. Alpha and beta

Beta is rented. AAPL's market beta of 1.22 says that on a day when the market's
excess return is 1%, AAPL's expected excess return is 1.22%. Anyone can buy that
relationship cheaply by holding more of an index fund. Alpha is what is left
when every rented exposure has been credited with what it explains: the average
daily return that no factor accounts for. It is the only part of a return stream
that can be attributed to skill or to something specific about the asset, and it
is the part a fee is nominally charging for.

In the regression, the slopes are the exposures and the intercept is alpha in
daily units. AAPL's exposures are Mkt-RF 1.222, SMB -0.138, HML -0.507, RMW
0.536 and CMA 0.304, which reads as a high-beta, large-cap, growth, profitable
stock. The intercept is 0.000299 a day. The subtraction of the risk-free rate
before fitting is not cosmetic: if the left side is a raw return rather than an
excess return, the intercept absorbs the average risk-free rate over the window
and is not alpha at all.

Section 4 builds the excess return series as asset return minus the Ken French
RF on the shared date index. Section 6 stores that intercept as `alpha_daily`
and reports 0.000299 × 252 = 7.53% a year. The multiplication is linear rather
than compounded, because a regression intercept is an average daily increment
and not a return being reinvested; compounding the same daily figure would give
7.82%, and the choice is listed in the README limitations so the reader knows
which convention produced the number. The same code path gives the comparison
names their betas: KO 0.59, BRK-B 0.83, ARKK 1.28.

## 3. Why the standard error is the whole story

The +7.53% is a measurement, and measurements have error bars. Here the error
bar swamps the thing being measured. The honest version of AAPL's result is that
its annual alpha is somewhere between losing 3.09% and making 18.96%, an
interval twenty-two percentage points wide with zero comfortably inside it.
Quoting +7.53% on its own is not false, but it is a statement about the middle
of a distribution most of which lies somewhere else, and it invites a reader to
treat a coin flip as a finding.

Ordinary least squares standard errors assume the residuals are independent
across days and have constant variance. Daily equity residuals are neither.
Volatility arrives in clusters, and residuals carry short-run serial
correlation, so consecutive days partly repeat each other's information. When
that happens, 2,910 days contain less independent evidence than 2,910
independent draws would, but OLS computes the standard error as though every day
were fresh. The result is a standard error that is too small, a t-statistic that
is correspondingly too large, and significance that the data does not support.
Newey-West replaces the covariance estimate with one that adds the correlations
between residuals at nearby dates, weighted down as the gap grows, out to a
chosen lag. It leaves the coefficient itself untouched. All that changes is the
uncertainty attached to it.

`newey_west_lags()` in section 6 implements floor(4 × (n/100)^(2/9)), the usual
rule of thumb, which gives 8 lags at n = 2,910 and 7 at the 1,512 observations
of the synthetic test in section 10. `run_factor_regression()` passes that lag
count to statsmodels as `cov_type="HAC"`, so the betas printed on the tearsheet
are plain OLS betas and only the t column reflects the correction. What survives
the correction is the point. Mkt-RF comes out at t = 44.97 and is not in question
under any covariance assumption anyone would defend. Alpha comes out at t = 1.34
and does not clear the 1.96 that `interpret()` in section 8 requires. The width
follows from how much of AAPL is left unexplained: the bootstrap interval implies
a standard error of about 5.6 percentage points a year on a 7.53% estimate, so
eleven and a half years of daily data are not enough to establish the sign.

## 4. Why a bootstrap on top of HAC

The HAC interval is a formula's answer to the question of how much this number
would move if history could be run again. The bootstrap answers the same
question by rerunning something close to history a thousand times and watching
where the estimate lands. Running both is how you find out whether the answer is
a property of the data or a property of one assumption. If the two disagree, at
least one of them is being carried by its assumptions, and that is worth knowing
before either gets quoted.

HAC produces an interval that is symmetric around the estimate and leans on the
sampling distribution of alpha being roughly normal. The bootstrap assumes
nothing about its shape. It resamples the observations, refits the regression on
each resample, and reads the 2.5th and 97.5th percentiles off the thousand
alphas that come back. Resampling individual days would break the serial
dependence that HAC exists to correct for and would hand back an interval
narrower than the data deserves, so the resampling is done in contiguous blocks
of five trading days, which keeps short runs of dependence intact.

`bootstrap_alpha_ci()` in section 6 runs 1,000 resamples at block length 5 with
seed 42, all set in `Config` in section 2, refitting by least squares each time
and keeping the intercept. Only alpha is resampled. The betas do not need it:
Mkt-RF at t = 44.97 and HML at t = -10.31 sit nowhere near a threshold where the
choice of interval method could change the reading, while alpha at 1.34 sits
exactly where it could. A thousand refits spent on the one coefficient in doubt
is the trade the code makes. For AAPL the resulting distribution has mean +8.41%
annualized against the OLS point estimate of +7.53%, with a 95% interval of
-3.09% to +18.96%. The 0.88 point gap between the two centers is small beside a
22 point width, and the two procedures agree on the verdict. That agreement is
what `interpret()` insists on: alpha is called significant only when the HAC
t-statistic clears 1.96 and the bootstrap interval excludes zero, so a result
resting on one assumption alone never gets announced.

## 5. Why the engine is validated on synthetic data

Before asking code a question you cannot check, ask it one you can. Section 10
builds an asset whose alpha and betas are known because they were chosen, runs
it through the same regression and the same bootstrap the live path uses, and
asserts that the numbers come back. If the code cannot recover an answer that
was put there deliberately, nothing it reports about AAPL is worth reading. The
test runs before section 11 fetches a single price, so a broken engine fails
before it can produce a plausible-looking tearsheet.

The synthetic asset has alpha 0.0005 a day, about 13% a year, and betas Mkt-RF
1.10, SMB -0.30, HML 0.25, RMW 0.15 and CMA -0.10, with normal noise, over 1,512
business days. The engine returns alpha 0.00051 at t = 6.39, market beta 1.096,
SMB -0.290, HML 0.232, RMW 0.157, CMA -0.066 and R² 0.931, and the bootstrap
interval on daily alpha, 0.00036 to 0.00066, contains the true 0.0005. CMA comes
back worst, -0.066 against a true -0.10, which is what the smallest coefficient
on the lowest-variance factor should do in a sample this size. The assertions are
therefore written on the market beta and on alpha, where the test has power to
detect a real error.

The cell asserts the market beta to within 0.06, alpha to within 0.00015, the
bootstrap interval bracketing the true alpha, and R² above 0.85, and raises if
any of them fails. It uses block length 1 rather than 5 because the synthetic
residuals are independent by construction, so there is no dependence to
preserve. The same cell hand-checks Sortino on four returns, +2%, -1%, +2%, -1%,
at a zero risk-free rate: squared shortfalls averaged over all four days give a
downside deviation of sqrt(5e-5) and a ratio of 11.2250, while averaging over
only the two losing days gives sqrt(1e-4) and 7.9373. That assertion is there
because the first implementation did the second thing, which deflated the ratio
by a factor of sqrt(2) on this example. The check stays in the file so the error
cannot return quietly. The cell also prints a table of true against estimated
values, but the printing is not the test. A table a reader has to eyeball passes
silently on the day someone breaks the alignment code in section 4, which is why
the tolerances are assertions.

## 6. What a null result means

Four names, eleven and a half years of daily data, and no detectable alpha in
any of them. That is the expected outcome. Large liquid US stocks are the most
scrutinized assets in the world, and a five-factor regression on daily closes
finding persistent unexplained return in them would be surprising. A tool that
reported significant alpha on whatever you pointed it at would be evidence of a
bug, not a discovery, and the most likely bugs are dull ones: misaligned factor
dates, forward-filled prices, a raw return where an excess return belongs.

The FF5 alpha t-statistics are AAPL +1.34, KO +0.15, ARKK +1.02 and BRK-B -0.07.
The 1.96 threshold corresponds to a 5% false positive rate on a single test, and
across four tests with no correction the chance of at least one crossing by luck
alone would be about 18% if the tests were independent, which four US equities
are not. None crossed. The point estimates on their own would read as a ranking:
+7.53%, +4.54%, +0.66%, -0.22%, ordered from best to worst manager. They are four
draws from distributions that all straddle zero, and the ordering carries no
information. Nor is the finding that alpha is zero. AAPL's interval reaches
+18.96% at the top, so a true alpha of ten points a year is equally consistent
with what was observed. The test establishes only that eleven and a half years of
daily returns cannot separate AAPL's 24.68% from a basket of factor exposures,
which is a claim about the strength of the evidence and not about the world.

`interpret()` in section 8 prints that alpha is not distinguishable from zero and
that the return stream is replicable with cheap factor exposure whenever either
test fails, so the null is stated in words rather than left for the reader to
infer from a t-statistic. `compare()` in section 9 applies its colour gradient to
the Sharpe column only and deliberately not to the alpha column, because shading
four estimates that are statistically indistinguishable would rank them on the
page and invite exactly the reading the notebook exists to prevent. The missing
multiple testing correction is listed as a limitation in section 12 rather than
patched, because with four names fixed in advance it would change nothing here,
while for anyone screening hundreds it would change everything.
