# Locked In: Mortgage Prepayment Behavior Across Rate Regimes

An individual capstone research paper modeling voluntary mortgage prepayment on Freddie Mac loan-level data for two origination vintages that lived through opposite refinance incentives: 2018 loans, which became deeply refinanceable when rates fell below 3% in 2020-21, and 2020 loans, which have been locked in far below market rates since 2022. The project builds a discrete-time hazard panel of about 4.1 million loan-months, fits a logistic baseline and a gradient-boosted tree model to each cohort, and runs a cross-regime transfer test (train on one vintage, test on the other). Headline results: 79.3% of modeled 2018 loans prepaid against 37.0% of 2020 loans (a 42-point gap), and a model carried across regimes keeps its ranking power (every cross-regime AUC within 0.02-0.04 of its native benchmark) but loses its calibration, with predicted monthly prepayment levels off by as much as a factor of three.

**Course:** BA889 Analytics Practicum (MSBA, Summer 2026)
**Instructor:** Professor Ritwick Roy
**Institution:** Boston University, Questrom School of Business

## Author

This is individual work, not a team project. The study design, data pipeline, models, analysis, and paper are by Roberto Albornoz Martinez.

## Project Overview

Prepayment models sit underneath the valuation of mortgage-backed securities and the management of servicing portfolios, and they are calibrated on historical data, which means on a particular interest-rate regime. This project asks what happens when the regime changes.

Two cohorts of Freddie Mac borrowers, separated by only two years of origination, experienced opposite incentive environments against the same calendar history. The 2018 vintage (average note rate 4.74%) became deeply in-the-money during the 2020-21 rate trough, and 78.9% of its loans had prepaid by September 2025. The 2020 vintage (average note rate 3.19%) was created at the money and has sat far below a market rate above 6% since 2022; only 36.9% of its loans have prepaid, and the rest are locked in. "Regime" here means the path of a cohort's locked-in rate against market rates, not a calendar period: both cohorts share the same macroeconomic shocks, and what differs is the incentive position.

The analysis proceeds in three stages. First, a discrete-time hazard panel is built from the monthly performance records of each vintage, with a refinance incentive computed from the Freddie Mac PMMS 30-year rate series. Second, two models per cohort (logistic regression and gradient boosting) are fitted on an 80% loan-level training split and evaluated on the held-out 20%. Third, every model scores the other cohort's held-out loans, so that transfer across regimes is measured on two dimensions that can fail independently: ranking (AUC) and calibration (predicted-to-actual monthly hazard).

## Research Questions

1. How does the rate incentive drive monthly voluntary prepayment in each regime, and do the two regimes respond to the same incentive in the same way?
2. Does a prepayment model calibrated on one regime transfer to the other, both in its ability to rank loans by risk and in its ability to state how much prepayment there will be?

## Data Sources

- **Freddie Mac Single-Family Loan-Level Dataset (academic sample)**: 50,000 loans from each of the 2018 and 2020 origination vintages, with monthly performance records through September 30, 2025 (1,998,728 and 2,340,586 loan-months respectively). Each vintage archive (`sample_2018.zip`, `sample_2020.zip`) holds an origination file (one row per loan: rate, balance, credit score, LTV, DTI, purpose, occupancy, term) and a servicing file (one row per loan-month: balance, rate, delinquency status, estimated current LTV, borrower-assistance status, termination code). Every loan in both samples is fixed-rate and carries no prepayment penalty. Column names come from Freddie Mac's official file layout workbook (July 2026 release). Dataset page: https://www.freddiemac.com/research/datasets/sf-loanlevel-dataset
- **Freddie Mac Primary Mortgage Market Survey (PMMS)**: weekly 30-year fixed rates, averaged to monthly and merged onto every loan-month with no unmatched rows. Used to build the refinance incentive (the loan's current rate minus the prevailing market rate) and the spread at origination. Series page: https://www.freddiemac.com/pmms

**The Freddie Mac data files are not included in this repository**, in line with the dataset's terms of use. To reproduce the analysis:

1. Register on the Freddie Mac dataset page and download the sample files for the 2018 and 2020 vintages (`sample_2018.zip` and `sample_2020.zip`) and the file layout workbook (`.xlsx`).
2. Download the PMMS history as a CSV with at least the columns `date` (M/D/YYYY) and `pmms30`, saved as `PMMS_history.csv`.
3. Place the five files in one data folder and set `PROJECT_ROOT` in Section 0 of the notebook to the folder that contains it (the notebook expects `<PROJECT_ROOT>/Data` for inputs and writes checkpoints, figures, tables, and fitted models to `<PROJECT_ROOT>/Output`).
4. Run all cells in order. The notebook was developed in Google Colab Pro with Google Drive mounted; the full servicing files take roughly 0.5 GB of memory per vintage, and the fitted-model checkpoints are about 1 GB each.

## Methodology

**Target definition and exclusions**
- Voluntary prepayment is identified by Freddie Mac termination code 01 (early payoff in full: refinancing, home sale, or any other voluntary payoff).
- Loans that ended in default-related dispositions or administrative removals (230 in 2018, 143 in 2020, under 0.5% of each cohort) are excluded rather than counted as non-prepayments, because a loan that defaulted never made the payoff decision under study. The modeled populations are 49,770 and 49,857 loans.

**Hazard panel construction**
- Every month of every modeled loan's life becomes one row carrying an indicator equal to one only in the month the loan prepaid: a discrete-time hazard structure of 1,778,314 loan-months (2018) and 2,336,348 (2020).
- The observation window is capped at 66 months of loan age, the longest window both cohorts can supply. The cap censors 1,342 late prepayments in the 2018 cohort (3.4% of its prepayments) and 4 in 2020; censored loans stay in the panel as loans that had not prepaid within the window.
- Balance-dependent variables (unpaid balance and estimated current LTV) enter as start-of-month values, lagged one month within each loan, so the model sees the borrower's situation before the payoff rather than after it.
- Freddie Mac's missing-value codes (999 for estimated LTV and DTI, 9999 for credit score) are converted to missing throughout.

**Features** (all knowable at the start of the month)
- Rate incentive: the loan's current rate minus the PMMS 30-year rate.
- Incentive path: burnout (cumulative positive incentive already lived through), peak incentive to date, and the length of the current in-the-money streak above a +0.5 percentage-point threshold (the point where the raw response curve turns sharply upward).
- Equity and eligibility: start-of-month estimated LTV (with a missingness indicator), delinquency status, and assistance-plan status.
- Origination characteristics: credit score, DTI, LTV, log original balance, 30-year term flag, loan purpose, occupancy, and the spread at origination (note rate minus the market rate at the first-payment month).
- Loan age and calendar month.
- Missing values are filled with medians computed from training loans only, and each fitted model keeps its own medians for use wherever it later predicts.

**Models and validation**
- 20% of loans in each cohort are held out, split by loan (`GroupShuffleSplit`, fixed seed) so that no loan's months appear on both sides.
- Logistic regression (`statsmodels` Logit) with standard errors clustered at the loan level, so that coefficients, standard errors, and p-values make the economics inspectable.
- Gradient boosting (`scikit-learn` `HistGradientBoostingClassifier`, default settings, fixed seed), able to express the nonlinear floor-and-elbow response visible in the raw data.
- Event rates are left at their natural levels (2.13% and 0.79% of training months), because the calibrated monthly probability is itself the quantity of interest.
- The delinquency and assistance indicators perfectly predict non-prepayment (no prepayment occurs in any assistance-plan month of either cohort, nor in any delinquent 2020 month), so no finite logistic coefficient exists for them; they are retained in the tree model only.

**The cross-regime transfer test**
- Each fitted model scores two test sets: its own cohort's held-out loans (which must reproduce the within-regime results exactly) and the other cohort's held-out loans, prepared with the scoring model's own training medians, exactly as a frozen production model deployed into a new rate environment would.
- Transfer is measured on ranking (AUC) and on calibration (the ratio of mean predicted to actual monthly hazard, plus decile calibration curves).

**Interpretation**
- Permutation importance on each boosting model over its own cohort's test set (5 repeats).
- Partial dependence of the monthly hazard on the rate incentive for both boosting models, swept over the same pooled 50,000-row sample, with each model's training range (99.5th percentile of its training incentive) marked.

## Key Findings

**The raw regime contrast**
- Among the modeled loans, 79.3% of the 2018 cohort prepaid within the data against 37.0% of the 2020 cohort, a 42-point gap. In hazard form, a 2018 loan prepaid in 2.14% of the months it was at risk; a 2020 loan in 0.79%.
- Both cohorts' prepayment waves crest in the same calendar month, March 2021, at 5.42% and 1.99% of surviving loans, at loan ages of roughly 30 and 11 months: timing followed market conditions, not the loan clock. At the trough, 99.9% of surviving 2018 loans stood above the +0.5 point threshold against 44.1% of 2020 loans. By month 66, 23.6% of the 2018 cohort was still active, compared with 61.6% of the 2020 cohort.
- The response curve has a floor, an elbow, and a crossing. Deep out of the money, both cohorts converge on roughly half a percent per month (home sales unrelated to refinancing). The response begins near +0.5 percentage points and rises steeply: the 2018 monthly rate nearly quadruples between +0.5 and +1.5, from 1.09% to 3.90%. The two curves do not coincide (the 2020 cohort prepays more at +0.5 and substantially less at +1.5), an early sign that the incentive response is entangled with borrower composition.
- The 2020 cohort is not the 2018 cohort two years later: higher median credit score (768 against 754), lower original LTV (74 against 80) and DTI (34 against 37), a larger median balance ($263k against $207k), and 70% refinances against two-thirds purchase loans in 2018.

**Within-regime model performance (held-out loans)**
- Logistic baseline AUC: 0.740 (2018) and 0.706 (2020). Gradient boosting AUC: 0.767 and 0.755, with average precision of 0.062 and 0.031 against event-rate baselines of 0.022 and 0.008.
- The logistic coefficients show the two regimes priced the same variables differently: the incentive coefficient is +0.48 per percentage point in 2018 against +0.35 in 2020; burnout is nearly identical (-0.023 and -0.025); the spread at origination is negative in both and much stronger in 2020 (-0.59 against -1.39); credit score is highly significant in 2018 and indistinguishable from zero in 2020, consistent with a cohort of pre-selected prime refinancers.

**The transfer test** (every model evaluated on every regime)

| Family | Model trained on | Test set | Direction | AUC | Predicted / actual hazard |
|---|---|---|---|---|---|
| Logistic | 2018 | 2018 | Native | 0.740 | 0.96x |
| Logistic | 2018 | 2020 | Cross | 0.689 | 0.62x |
| Logistic | 2020 | 2018 | Cross | 0.711 | 3.18x |
| Logistic | 2020 | 2020 | Native | 0.706 | 1.03x |
| Gradient boosting | 2018 | 2018 | Native | 0.767 | 0.98x |
| Gradient boosting | 2018 | 2020 | Cross | 0.719 | 1.03x |
| Gradient boosting | 2020 | 2018 | Cross | 0.745 | 1.29x |
| Gradient boosting | 2020 | 2020 | Native | 0.755 | 1.08x |

- **Ranking survives.** Every cross-regime AUC lies within 0.02-0.04 of its native benchmark (worst case 0.689 against a native 0.706).
- **Calibration does not, and the failure is asymmetric.** Native models are honest (predicted-to-actual ratios of 0.96 to 1.08). The 2018 logistic model predicts 0.62 times the true monthly hazard on 2020 data, while the 2020 logistic model predicts 3.18 times the observed hazard on 2018 data. The tree model is more robust but not immune: 1.03x in one direction, 1.29x in the other, and its highest-risk decile of the 2018 test set is predicted at 12.7% against 5.8% observed. The dangerous direction is from the low-activity regime into the high-activity one.
- **The gap is invisible on the metric that validation reporting leads with.** A model review that checked a transferred model's AUC would conclude it still works; a portfolio priced on its predicted levels would be systematically mispriced.

**The mechanism**
- Permutation importance shows two different decision structures: the 2020 model depends overwhelmingly on the incentive (shuffling it costs 0.162 of AUC, more than every other feature combined), while the 2018 model spreads its reliance across loan size (0.049), the in-the-money streak (0.048), loan age (0.036), and the incentive (0.021). Where refinancing was briefly economical, identifying the window is the whole task; where it was economical for years, the task is identifying which borrowers act, and when.
- Partial dependence shows each model's learned hazard-versus-incentive curve ending where its training data ends. The 2020 model's curve flattens at 3.57% before its own training limit of +1.79 and answers every question beyond it with that plateau; the 2018 model's curve is still rising at +2.5. Above +1, the 2020 model's implied hazard exceeds the 2018 model's by roughly 1.5 percentage points, the direct mechanism of the 1.29x overprediction and, in more extreme form, of the logistic model's 3.18x.

**Recommendations for practitioners**
1. Validate transferred models on regime coverage and calibration, not discrimination alone: a predicted-to-actual hazard ratio is a one-line computation, and it is the number that detects the failure AUC conceals.
2. Monitor a deployed model's inputs against its training range: the damage concentrates where the current incentive distribution exceeds the regime the model was fitted on, so a simple range check on the incentive is a direct early-warning signal.
3. Treat a regime change as a recalibration trigger rather than trusting flexible models to bridge it: functional form helps, but no functional form substitutes for data from the regime being priced.

## My Contribution

This project is individual work. I chose the research question and the cross-regime design, built the data pipeline from the raw Freddie Mac files (schema assignment from the official layout, cleaning, target definition, hazard-panel construction, and feature engineering), fitted and validated the models, designed and ran the transfer test and the interpretation diagnostics, and wrote the paper.

## Tools and Technologies

- **Python** (pandas, NumPy)
- **Statistical modeling**: statsmodels (Logit with loan-clustered standard errors)
- **Machine learning**: scikit-learn (HistGradientBoostingClassifier, GroupShuffleSplit, roc_auc_score, average_precision_score, permutation_importance, partial_dependence)
- **Visualization**: matplotlib, seaborn
- **Data handling**: pyarrow (Parquet checkpoints), joblib (model checkpoints), openpyxl (file layout workbook)
- **Environment**: Google Colab Pro with Google Drive mounted; a single seed (`RANDOM_STATE = 42`) governs sampling, splits, and model initialization

## Repository Contents

- `Locked_In_Prepayment_Across_Rate_Regimes.ipynb`: the full notebook, with outputs, from raw-file ingestion to the transfer test and interpretation.
- `Locked_In_Mortgage_Prepayment_Across_Rate_Regimes_Paper.pdf`: the final research paper (August 2026).
- `figures/`: the seven figures produced by the notebook. Five of them appear in the paper: `fig1_hazard_vs_incentive.png` (paper Figure 1), `fig4_prepayment_calendar.png` (Figure 2), `fig5_calibration.png` (Figure 3), `fig6_importance.png` (Figure 4), and `fig7_pdp_incentive.png` (Figure 5). `fig2_survival_by_age.png` and `fig3_incentive_paths.png` are supporting exhibits from the exploratory analysis.
- `requirements.txt`: Python dependencies.
- `.gitignore`: excludes the data folder, checkpoints, and fitted models.

## Course Context

**BA889 Analytics Practicum** is a course in the Boston University MSBA program at the Questrom School of Business, taken after the core analytics sequence (BA775, BA780, BA810, BA820, BA830). Students apply their business-analytics skills (data analysis, data visualization, machine learning) to answer a meaningful question for a business setting and document the work in a paper, either a reflection paper on an analytics internship or a research paper on an analytics project carried out outside the MSBA coursework. This repository is the research-paper option. The practicum allows a paper to build on an earlier course project but not to repeat one; this study is new work, distinct from the loan-default projects listed below in both the borrower decision it models (prepayment rather than default) and its design (a panel hazard with a cross-regime test rather than a cross-sectional classifier).

## Other Projects

- [Loan Default Risk Prediction](https://github.com/rfalbornoz/loan-default-risk-prediction): supervised ML pipeline (logistic regression, decision tree, random forest, XGBoost) on LendingClub data for New England and New York (team project).
- [European Pharmaceutical R&D Analysis](https://github.com/rfalbornoz/european-pharmaceutical-rd-analysis): unsupervised learning and statistical testing on EMA drug-authorization data (team project).
- [Massachusetts Loan Default Risk](https://github.com/rfalbornoz/analytics-massachusetts-loan-risk): descriptive and geospatial analysis integrating LendingClub loans with ACS Census estimates (team project).
