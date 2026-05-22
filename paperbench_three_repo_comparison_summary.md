<!--
What this file does:
  Summarizes three PaperBench repository comparisons for inclusion in the
  DK-Class-Report writeup.

Sample usage:
  Include this Markdown section as report prose, or translate the tables and
  paragraphs into the ACL LaTeX manuscript under latex/acl_latex.tex.

Inputs:
  /mnt/blk1/Paper2Code/outputs/zzcomparison/bam/comparison.json
  /mnt/blk1/Paper2Code/outputs/zzcomparison/adaptive-pruning/comparison.json
  /mnt/blk1/Paper2Code/outputs/zzcomparison/bbox/comparison.json
  /mnt/blk1/Paper2Code/data/paperbench_jsons/<paper>/paper_cleaned.json
-->

# Comparison Summary for Three PaperBench Repositories

We compared three generated repositories against their corresponding papers,
rubrics, and official reference repositories: BaM (`bam`), APT
(`adaptive-pruning`), and BBOX-ADAPTER (`bbox`). The comparison used the
reference repositories as behavioral calibration, not as strict file-template
targets. Scores are rubric-weighted implementation-faithfulness estimates, so
they reflect both code coverage and evidence that the paper's experiments were
actually reproduced.

| Paper | Weighted score | Rubric leaves | Main implementation profile | Paper faithfulness |
|---|---:|---:|---|---|
| BaM | 0.308 | 789 | Strongest on core variational-inference algorithms; weak on executed experiments and real PosteriorDB/deep-generative reproduction. | Partially faithful: the main algorithmic logic is present, but paper-scale evidence is mostly missing. |
| APT | 0.246 | 123 | Coherent adaptive-pruning framework with masks, salience scoring, dynamic ranks, and training utilities; empirical evidence is minimal. | Weakly faithful: captures the broad APT idea, but diverges on several paper-critical details and lacks reported paper results. |
| BBOX-ADAPTER | 0.290 | 279 | Modular black-box adapter pipeline with pair scoring, online stores, NCE-style training, reranking, and beam search; configs/results are incomplete. | Partially faithful: architecture is recognizable, but training/evaluation details and reproduced tables are largely absent. |

## BaM: Batch and Match

The BaM repository is the strongest of the three in terms of core mathematical
logic. It implements BaM, ADVI, Score-VI, Fisher-VI, and GSM-style algorithms
with full-covariance Gaussian variational families, synthetic target
distributions, per-iteration metrics, and shared experiment infrastructure.
This matches the paper's central claim that BaM is a score-divergence-based
black-box variational inference method with closed-form Gaussian-family
updates.

However, most paper-obedience gaps are in experimental detail rather than the
algorithm skeleton. The comparison found no committed evidence of the required
long seeded runs, grid-search sweeps, final figures, or reproduced quantitative
results. PosteriorDB support is mostly generic scaffolding: real target-specific
log posterior and score functions for the paper's models are absent or replaced
by toy/generic mechanisms. The deep generative experiment is also partial:
CIFAR loading, VAE-style code, posterior scoring, and reconstruction metrics
exist, but trained decoder artifacts and completed reconstruction outputs were
not found.

Overall, BaM obeys the paper at the level of method structure, but not at the
level of full experimental reproduction.

## APT: Adaptive Pruning and Tuning

The APT repository implements a plausible adaptive-pruning system. It contains
an APT-like adapter layer with frozen base weights, input/output masks, dynamic
rank expansion, salience-density pruning, rank allocation, two-stage training,
distillation scaffolding, evaluation utilities, and RoBERTa/T5 configurations
that encode many paper hyperparameters. This makes the repository recognizable
as an implementation of the paper's high-level idea: adaptively tune salient
parameters while pruning unimportant structure to improve training and
inference efficiency.

The main weaknesses are paper-critical details. Baselines are local proxies
rather than the specified Mask Tuning and CoFiPruning implementations. Several
method details diverge from the paper/reference behavior, including effective
adapter scaling, multi-head attention mask indexing, outlier salience,
Equation-9-style adapter salience, layer mapping for distillation, and KL
prediction distillation. The empirical side is especially weak: the only
observed run is a toy tiny-random RoBERTa SST-2 run with no pruning, while the
paper requires 60% sparsity experiments, relative accuracy analysis, efficiency
claims, and ablations.

Overall, APT captures the broad engineering direction of the paper, but it does
not yet obey the paper closely enough in detailed mechanics or experimental
evidence.

## BBOX-ADAPTER

The BBOX-ADAPTER repository contains a substantial adaptation pipeline. It has
code for scalar question-answer pair scoring, ranking/NCE-style adapter
training, online positive/negative example storage, single-step reranking,
sentence-level full-step beam search, dataset preparation, and metric
utilities. It also recognizes several important paper settings from Appendix
H.2, including maximum generation length 512, temperature 1.0, learning rate
5e-6, batch size 64, 6000 training steps, AdamW, and beam size 3. Dataset
manifests show materialized GSM8K, StrategyQA, TruthfulQA, and text-only
ScienceQA splits.

The repository still falls short of paper-faithful reproduction. There are no
committed adapter checkpoints, training logs, evaluation summaries, or
reproduced numbers for the paper's main tables. The default configuration is
not runnable as-is because config keys expected by the loader do not match the
provided YAML. The ranking loss path has an indentation bug affecting
`RankingNCELoss.forward`, and the Equation 3 score regularizer is effectively
disabled by default because the alpha value is not resolved from the expected
configuration location. Several paper components, including MLM ablation,
fine-tuning baselines, plug-and-play results, cost trends, and the Mixtral
white-box extension, are stubs or planned aggregations rather than completed
experiments.

Overall, BBOX-ADAPTER has a recognizable architecture and many useful modules,
but its details and evidence are not sufficient to claim close obedience to the
paper.

## Cross-Paper Interpretation

Across all three repositories, the dominant pattern is the same: generated code
often captures the high-level method and implements plausible scaffolding, but
falls short on paper-specific detail and experimental proof. BaM is most
convincing on core algorithmic logic. APT is most coherent as an engineering
framework for pruning and tuning, but has the largest gap between method
scaffold and paper-scale results. BBOX-ADAPTER has the broadest pipeline for a
black-box LLM adaptation workflow, but suffers from configuration and loss-path
issues that make reproducibility fragile.

The comparisons suggest that these generated repositories should be treated as
starting points for reproduction, not as faithful reproductions of the papers.
They are useful for studying how much of the method an automated coding system
can reconstruct, but they require substantial human correction before their
outputs can support paper-level empirical claims.
