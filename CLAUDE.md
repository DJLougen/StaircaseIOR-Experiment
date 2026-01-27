# Staircase IOR Experiments

This repository contains web-based psychophysics experiments built with **jsPsych 7.3.4** that investigate **Inhibition of Return (IOR)** using adaptive procedures (traditional staircases and Bayesian methods).

## Background: Inhibition of Return (IOR)

IOR is an attentional phenomenon where reaction times (RT) to targets are **slower** at previously cued locations compared to uncued locations. The effect is time-dependent:

- **Short SOAs** (~0-200ms): **Facilitation** - cued RT < uncued RT (attention helps)
- **Long SOAs** (~300ms+): **IOR** - cued RT > uncued RT (attention inhibits return)

**Key terminology:**
- **SOA** (Stimulus Onset Asynchrony): Time from cue onset to target onset
- **ISI** (Inter-Stimulus Interval): Gap between cue offset and target onset
- **Valid trial**: Target appears at cued location
- **Invalid trial**: Target appears at uncued location
- **IOR magnitude**: `valid RT - invalid RT` (positive = IOR present)

## Experiments

### 1. `index.html` - Crossover Experiment

**Goal:** Find the SOA where the effect transitions from facilitation to IOR (the "crossover point" where cued RT = uncued RT).

**Staircase logic:**
- Compares mean RT of last N valid vs invalid trials
- If `valid RT < invalid RT` → facilitation detected → step UP (increase ISI)
- If `valid RT >= invalid RT` → IOR detected → step DOWN (decrease ISI)
- Uses **symmetric 1-up 1-down** stepping (equal step sizes in both directions)
- **Delayed step shrinking:** Step size stays constant for first 5 reversals (exploration), then reduces by 25% on each subsequent reversal (fine-tuning)

**Parameters:**
| Parameter | Value |
|-----------|-------|
| ISI range | 50-600ms |
| Start ISI | 200ms |
| Initial step | 50ms |
| Min step | 10ms |
| RT window | 5 trials |
| Reversals to finish | 12 |
| Reversals before shrink | 5 |

### 2. `peak-ior.html` - Peak IOR Experiment

**Goal:** Find the SOA that produces the **maximum** IOR effect (strongest inhibition).

**Staircase logic:**
- Uses hill-climbing: compares current IOR magnitude to previous
- If IOR increased → keep going in same direction
- If IOR decreased → reverse direction (overshot the peak)
- Uses **asymmetric 1-up 3-down** stepping:
  - Step UP: `+1× stepSize`
  - Step DOWN: `-3× stepSize` (helps escape local minima)
- **Delayed step shrinking:** Step size stays constant for first 5 reversals (exploration), then reduces by 30% on each subsequent reversal (fine-tuning)
- Includes initial **sampling phase** (5 ISIs × 5 reps) to find good starting point

**Parameters:**
| Parameter | Value |
|-----------|-------|
| ISI range | 100-800ms |
| Start ISI | 300ms |
| Initial step | 75ms |
| Min step | 15ms |
| RT window | 6 trials |
| Reversals to finish | 14 |
| Reversals before shrink | 5 |
| Sampling ISIs | 100, 275, 450, 625, 800ms |

### 3. `psi-crossover.html` - Psi Method Crossover Experiment

**Goal:** Same as index.html (find crossover point), but using Bayesian adaptive method for faster convergence.

**Psi method logic:**
- Maintains a 2D posterior distribution over threshold and slope parameters
- Models P(IOR | ISI) using a logistic psychometric function
- After each observation (comparing valid vs invalid RTs), updates posterior via Bayes' theorem
- Selects next ISI to **maximize expected information gain** (minimize expected posterior entropy)
- Provides uncertainty estimates (posterior SD, confidence intervals)

**Advantages over traditional staircase:**
- Faster convergence: typically 30-50 trials vs 60-100
- Optimal stimulus placement at each trial
- Built-in uncertainty quantification
- Estimates both threshold AND slope of psychometric function

**Parameters:**
| Parameter | Value |
|-----------|-------|
| ISI range | 50-600ms |
| ISI grid resolution | 23 levels |
| Threshold grid | 30 levels |
| Slope grid | 15 levels (20-150ms) |
| RT window | 3 trials per observation |
| Min trials | 30 |
| Max trials | 80 |
| Stop criterion | Posterior SD < 20ms |

## Trial Structure (All Experiments)

1. **Fixation** (750ms): Central fixation cross with peripheral boxes
2. **Cue** (50ms): White flash in one box
3. **ISI** (variable): Gap period (staircase-controlled)
4. **Target** (1000ms max): White square in one box, respond with Z (left) or M (right)
5. **Feedback**: "Too slow!" if no response

## Key Files

| File | Description |
|------|-------------|
| `index.html` | Crossover experiment (symmetric staircase) |
| `peak-ior.html` | Peak IOR experiment (asymmetric staircase with sampling) |
| `psi-crossover.html` | Crossover experiment (Bayesian Psi method) |

## Key Functions

### Staircase Functions (index.html, peak-ior.html)
- `updateStaircase()` - Core logic for step direction and size
- `shouldUpdateStaircase()` - Check if enough trials collected
- `isStaircaseFinished()` - Check reversal count
- `getRecentRTs(n)` - Get last n valid/invalid RTs for comparison
- `getThreshold()` / `getPeakISI()` - Calculate final estimate from reversals

### Psi Method Functions (psi-crossover.html)
- `initializePsi()` - Set up uniform prior over parameter grid
- `psychometricFunction(isi, threshold, slope)` - Logistic function P(IOR | ISI)
- `expectedEntropy(isi)` - Calculate expected posterior entropy for an ISI
- `selectOptimalISI()` - Find ISI that minimizes expected entropy
- `updatePosterior(isi, observedIOR)` - Bayesian update after observation
- `getThresholdEstimate()` - Posterior mean for threshold
- `getThresholdSD()` - Posterior standard deviation
- `isPsiFinished()` - Check stopping criteria (min trials, SD threshold)

### Trial Functions (all files)
- `generateTrialHTML(phase)` - Create stimulus display
- `setupNewTrial(isPractice)` - Configure trial parameters

## Staircase Design Rationale

**Why symmetric for crossover?**
The crossover point is a balance point where the effect equals zero. Overshooting from either direction is equally problematic, so symmetric stepping is appropriate.

**Why 1-up 3-down for peak-finding?**
When hill-climbing for a maximum, larger downward steps help escape local maxima. The asymmetry biases the search to more aggressively explore when the effect decreases, preventing the staircase from getting stuck.

**Why delay step shrinking until after 5 reversals?**
Early reversals are often noise, not signal - the staircase hasn't found the target region yet. Shrinking too early slows convergence and can trap the staircase far from the true target. The first 5 reversals serve as a coarse exploration phase with large steps, then shrinking begins for fine-tuning.

**Why use the Psi method?**
The Psi method (Kontsevich & Tyler, 1999) is optimal in an information-theoretic sense - each trial placement maximizes the expected reduction in uncertainty about the threshold. Unlike staircases which follow fixed rules, Psi adapts based on the full posterior distribution. This typically reduces the number of trials needed by 30-50% while also providing confidence intervals on the estimate.

**When to use staircase vs Psi?**
- **Staircase:** Simpler to implement, well-understood properties, robust to model misspecification
- **Psi:** Faster convergence, provides uncertainty estimates, but assumes the psychometric function shape is correct

## Data Output

Both experiments output CSV with columns including:
- `trial_number`, `phase` (practice/staircase)
- `isi`, `validity`, `cue_side`, `target_side`
- `rt`, `response`, `correct`
- `staircase_reversal`, `step_size`
