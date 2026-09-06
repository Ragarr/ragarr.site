---
title: "Forecasting Soil Moisture with LSTMs and Transformers: What the Data Actually Needed"
date: 2026-09-06 08:00:00 +0200
categories: [Research, Machine Learning]
tags: [time-series, lstm, transformer, deep-learning, sensor-networks, publication, preprocessing]
pin: true
toc: true
---

<!-- TODO: cover image → media/2026-09-06-soil-moisture-forecasting/portada.png,
     then add the `image:` front-matter block. -->

> **Status: draft.** The pipeline details, metrics and ablation results below are being written up
> from the two papers. Every quantitative claim is deliberately absent until it comes from the
> published work.
{: .prompt-warning }

This is the line of work I did in the **GIAA group** at Universidad Carlos III de Madrid, in
collaboration with the Universidad Politécnica de Madrid, on forecasting soil moisture from ground
sensor networks in the **Duero basin**. It produced two first-author publications:

- **Deep Learning for Robust Soil Moisture Forecasting Using LSTM and Transformer** — SOCO 2025,
  Springer.
- **Data-driven forecasting of soil moisture based on enhanced pre-processing of data from on ground
  sensor networks** — R. Aguilar, M. A. Patricio, A. Berlanga, J. M. Molina, S. Zubelzu, *Physics and
  Chemistry of the Earth, Parts A/B/C*, vol. 145, art. 104766, 2026.
  [DOI](https://doi.org/10.1016/j.pce.2026.104766)

The obvious post to write about this would be about the models. This is not that post, because the
models were not where the difficulty was. The title of the journal article says so out loud —
*enhanced pre-processing* — and that is the honest description of where the contribution lives.

## 1. Field sensor data is a mess, and that is the actual problem

A soil moisture time series from a research paper is a clean, evenly sampled, gap-free vector. A soil
moisture time series from a sensor buried in a field in the Duero basin is not that. It is an
instrument exposed to weather, wildlife, power interruptions and time, transmitting over a link that
does not always work, and it produces data with all of the corresponding pathologies:

- **Gaps**, from a few missing samples to extended outages, and not randomly distributed — sensors
  tend to fail under exactly the conditions that make the readings most interesting.
- **Drift**, where a sensor's calibration moves slowly enough that no single reading looks wrong.
- **Hard faults**, producing stuck values, flatlines, out-of-range spikes, or plausible-looking
  garbage that is harder to catch than obvious garbage.
- **Irregular sampling**, because acquisition schedules and reality do not always agree.

None of this is incidental. It is a permanent property of the measurement setting, and any method
that is going to be used on real data has to survive it.

## 2. What most papers do about it, and why that is not enough

The standard treatment is short: drop the incomplete records, interpolate the small gaps linearly,
and proceed to the model. Sometimes it is not mentioned at all, which amounts to the same thing.

There are two problems with that. The first is that dropping incomplete records is not a neutral
operation — it removes precisely the periods when the instrumentation was under stress, which
biases the dataset toward benign conditions and quietly inflates the reported performance. The second
is that it makes results non-transferable: a model validated on the surviving clean subset has not
been shown to work on the data a practitioner will actually have.

So the interesting question is not *which architecture forecasts best on clean data*. It is *what has
to happen to the raw data before any architecture has a fair chance*, and how much of the final
performance that step is responsible for.

## 3. The pre-processing pipeline

<!-- TODO: this is the core section and it comes straight from the PCE paper. Needed:
       - the gap-handling strategy, and the threshold at which the treatment changes
         (short gaps vs. extended outages are presumably not handled the same way)
       - the quality-control rules: how stuck values, flatlines, spikes and
         out-of-range readings are detected, and what the criteria actually are
       - the drift correction / calibration approach
       - resampling and alignment: target frequency and how irregular timestamps
         are reconciled
       - normalisation, and whether it is fitted per sensor or globally
       - the order the stages run in, and which are conditional
       - a pipeline diagram → media/2026-09-06-soil-moisture-forecasting/pipeline.png
     State the rules concretely enough that someone could reimplement them. -->

## 4. Models: LSTM, Transformer, and honest baselines

Both architectures were evaluated on the same pre-processed data, against baselines — the point of
which is not to be beaten but to establish how much of the problem is genuinely hard. In forecasting,
persistence and seasonal-naive baselines are unreasonably strong on the exact horizons where deep
models are usually reported as winning, and a paper that omits them is not reporting a result.

<!-- TODO: from both papers —
       - the exact architectures and hyperparameters used for the LSTM and the Transformer
       - the baselines the paper actually compares against
       - forecast horizon(s) and input window length
       - train/validation/test split, and whether it is time-based (it must be) and
         whether sensors are held out as well as time periods
       - metrics and results tables
       - which model wins, on which horizon, and by how much -->

## 5. What each part contributed: pre-processing vs. architecture

This is the ablation that makes the case, and it is the reason the journal paper is framed the way it
is: separating how much of the final accuracy comes from the pre-processing and how much from the
choice of architecture.

<!-- TODO: the ablation numbers from the PCE paper — each model with and without
     the enhanced pre-processing, and ideally with individual stages disabled, so
     the relative contribution is visible rather than asserted. -->

If the pre-processing accounts for a large share of the improvement, that is a more useful finding
for a practitioner than any ranking of architectures, because it is the part that transfers to their
sensors, their basin and their failure modes.

## 6. From SOCO 2025 to the journal article

<!-- TODO: what was extended between the two papers. Candidates to confirm:
       - a larger or longer dataset, more sensors, more stations
       - the pre-processing contribution itself, if that is what the journal paper adds
       - the ablation study
       - additional baselines or architectures
       - broader evaluation (more horizons, more sites, cross-sensor generalisation)
     A clear statement of the delta is worth having — it is the question a reader
     who has seen the conference paper will arrive with. -->

## 7. Reproducing it

The code is at
[github.com/Ragarr/soil-moisture-forecasting-lstm-transformer](https://github.com/Ragarr/soil-moisture-forecasting-lstm-transformer).

<!-- TODO:
       - confirm the repository is public
       - add setup/run instructions, or point at the repository README
       - say what the data availability situation is: are the Duero basin series
         redistributable, and if not, what a reader can run instead -->

---

*Both papers are listed on the [Publications](/publications/) page.*
