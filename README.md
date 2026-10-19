# GW150914-chirp-mass-with-a-hand-written-MCMC

Estimating the chirp mass of the first detected gravitational-wave event, GW150914, from LIGO Hanford (H1) open data, using a Newtonian chirp template, a frequency-domain Gaussian likelihood and a Metropolis sampler written from scratch. The analysis then tests its own model with an injection of a realistic waveform of known mass into real detector noise.

![Real event vs known-truth injection](figures/injection_vs_real.png)

## Result

| | Chirp mass, detector frame (68% interval) | Reference |
|---|---|---|
| **GW150914, real H1 data** | **36.9 +2.1 / −0.5 M☉** | published ≈ 30 M☉ → offset **+23%** |
| **Injection of a realistic waveform into real H1 noise** | 38.3 +1.3 / −2.1 M☉ | known truth 30.72 M☉ → bias **+25%** |

**The main finding is the bias, not the number.** A leading-order (Newtonian) chirp template overestimates the chirp mass of a GW150914-like system by about 25%. The injection, where the true answer is known, is biased by the same amount as the real event's offset from the published value. This points to the model's physics as the cause of the gap, rather than the code or the data.

The statistical uncertainty (about ±0.5 M☉) is much smaller than this model bias (about +7 M☉): **for this analysis, the error budget is dominated by the model, not the noise.**

As a rough consistency check only, dividing the real-data result by the injection's bias factor (36.9 / 1.25) gives ≈ 29.5 M☉, close to the published value. A single injection is not a calibration, so this is not presented as a measurement.

## Method

### Data
- 32 s of H1 strain at 4096 Hz centred on GW150914, from the [Gravitational Wave Open Science Center](https://gwosc.org) (GWOSC).
- GWOSC no longer distributes 32 s files for this event, so the loader downloads the 4096 s release once and caches a 32 s segment centred on the event. Downloads go to a temporary file and are renamed only when complete, so an interrupted download can't leave a corrupt cache.

### Preprocessing
1. **High-pass at 15 Hz** before estimating the noise spectrum. The raw strain is dominated by low-frequency ground motion about 10 billion times more powerful (in power) than the noise in the analysis band. Without the high-pass, spectral leakage inflates the noise estimate in-band, and the whitened noise comes out too quiet.
2. **Noise spectrum (PSD)** by Welch's method, 4 s segments.
3. **Whitening:** divide the spectrum by the noise level, scaled so that whitened noise has unit variance.
4. **Band-pass 35–350 Hz** and notches at 60/120/180 Hz (US mains), for visualisation.

**Checks:** the whitened noise has a standard deviation close to 1. After band-passing it matches the value obtained by running the identical pipeline on simulated white noise (≈ 0.35), where the right answer is known.

### Template
Leading-order (0PN) inspiral, depending on a single mass parameter, the chirp mass Mc. With τ = t_c − t and T = G·Mc/c³:

- frequency: f(τ) = (1/π) (5 / 256τ)^(3/8) T^(−5/8)
- phase: Φ(τ) = −2 (τ / 5T)^(5/8)
- amplitude ∝ f^(2/3)

The template is cut off at **F_MAX = 150 Hz**, since the formula breaks down near merger. The template is built as a cosine and a sine version, so the unknown amplitude and phase enter linearly and are solved for exactly by least squares.

### Likelihood
Gaussian likelihood in the **frequency domain**, using only 35–150 Hz (the range the template models). In the frequency domain, Gaussian noise is independent between bins, so the likelihood is exact rather than approximate, and noisy spectral lines are automatically down-weighted by their large PSD. Amplitude and phase are maximised analytically (a *profile likelihood*).

**Normalisation check:** with no signal model, χ² per degree of freedom is 0.957 (expected ≈ 1). The best-fit Newtonian template has SNR 12.4.

### Prior and sampler
- Flat prior on Mc in [20, 50] M☉ and on t_c within ±20 ms of the best fit.
- Sampling is done in **ln Mc**, so the log-prior includes the Jacobian term ln Mc.
- **Random-walk Metropolis**, written from scratch. Three short pilot runs (discarded as burn-in) learn the posterior covariance; the main run proposes steps along it, scaled by 2.38²/d. This matters because Mc and t_c are strongly correlated.
- **Two chains** started far apart (Mc = 32 and 42). Convergence is checked with the Gelman–Rubin statistic: **R̂ = 1.003 (Mc) and 1.004 (t_c)**.

The posterior has small secondary modes (around 35, 40 and 42 M☉) from **cycle slips**: a slightly different mass with a shifted t_c can line the template up one cycle earlier or later. Both chains visit these modes, which is why the interval is asymmetric and percentiles are used rather than mean ± standard deviation.

## Validation

### Injection test in real noise
A realistic inspiral–merger–ringdown waveform (IMRPhenomD; GW150914-like detector-frame masses 39 + 32 M☉, **true Mc = 30.72 M☉**) is added to 32 s of real H1 noise taken 300 s before the event, scaled to **SNR 20**, the value LIGO's search measured for GW150914 in Hanford ([Abbott et al. 2016, arXiv:1602.03839](https://arxiv.org/abs/1602.03839)). The identical pipeline and sampler recover **38.3 +1.3 / −2.1 M☉, a bias of +25%** (R̂ = 1.005).

The waveform file (`gw150914_like_imrphenomd.npz`) is included in the repo. `generate_injection_waveform.py` shows exactly how it was made; it needs LALSuite, which runs on Linux and macOS but not natively on Windows.

### Sensitivity to the template cut-off

| F_MAX | Chirp mass |
|---|---|
| 120 Hz | 37.7 +0.5 / −0.6 M☉ |
| 150 Hz | 36.9 +2.1 / −0.5 M☉ |
| 180 Hz | 37.1 +0.3 / −0.3 M☉ |

The result moves by less than 1 M☉ across a 50% change in the cut-off, so it is not an artefact of choosing 150 Hz. (The 150 Hz run has a wider interval because its chains visited the secondary mode near 40 M☉; the 120 and 180 Hz runs were shorter and stayed in the main mode.)

## Figures

| | |
|---|---|
| `figures/whitened_chirp.png` | The whitened, band-passed chirp in H1 |
| `figures/template_vs_data.png` | A 30 M☉ Newtonian template against the data |
| `figures/profile_likelihood.png` | Profile likelihood vs chirp mass |
| `figures/posterior_mc.png` | MCMC trace plot and posterior |
| `figures/injection_vs_real.png` | Real event vs known-truth injection |

## Limitations

- **Newtonian (0PN) template.** No post-Newtonian corrections, mass ratio or spins. This is the source of the ~25% bias.
- **One detector.** Only H1 is used; L1 would add information and a consistency check.
- **Amplitude and phase are maximised, not marginalised.** For two linear parameters this changes the posterior only slightly, but a fully Bayesian treatment would integrate over them.
- **The bias comes from a single injection** (one waveform, one noise segment). A proper calibration would use many noise segments and a range of waveforms.
- **Detector-frame mass.** No cosmological redshift correction to the source frame.
- **Noise assumed Gaussian and stationary** over the 32 s segment.

## Possible next steps

- Add post-Newtonian phase terms (a TaylorF2 template) and show the bias shrinking.
- Repeat the injection over many noise segments to measure the bias and its spread.
- Add a Newtonian-template injection to confirm the pipeline is unbiased when the model is exactly right.
- Combine H1 and L1.
- Compute the effective sample size of the chains.

## Reproducing

```bash
pip install -r requirements.txt
jupyter notebook gw150914_chirp_mass.ipynb
```

The first run downloads about 130 MB from GWOSC; later runs use the cached 32 s segment. On Windows networks with HTTPS inspection (some antivirus or university networks), `truststore` is used so certificate checks still pass.

Environment used: Windows, Python 3.11, numpy, scipy, matplotlib, h5py, gwosc, truststore.

## References

- B. P. Abbott et al. (LIGO Scientific and Virgo Collaborations), *The basic physics of the binary black hole merger GW150914*, [arXiv:1608.01940](https://arxiv.org/abs/1608.01940). Source of the ≈ 30 M☉ comparison value.
- B. P. Abbott et al., *GW150914: First results from the search for binary black hole coalescence with Advanced LIGO*, [arXiv:1602.03839](https://arxiv.org/abs/1602.03839). Source of the H1 SNR of 20.

## Acknowledgement

<!-- TODO: paste GWOSC's required acknowledgement text here, from the "Acknowledge" page on gwosc.org -->
This research has made use of data obtained from the Gravitational Wave Open Science Center (gwosc.org).

---
*Author: Stylianos Spyrou*
