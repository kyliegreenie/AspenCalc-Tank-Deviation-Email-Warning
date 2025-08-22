# Tank Level Sensor Validation & Flow‑Rate Warning System

A fault‑tolerant approach to tank level measurement using two redundant level sensors. The script:

* Detects sensor disagreement, noise, drift, and stuck behavior
* Chooses the more reliable sensor automatically
* Flags high fill‑rate conditions for email alerting (integration notes below)
* Demonstrates a real incident via a sanitized screenshot (tag names blurred)

> **Why this exists:**
> Even sensors that were previously accurate can **drift, break, or get noisy**. This logic continuously checks plausibility and reliability so downstream control/alarms use the **best available signal**.
> With so many tanks it can be easy to overlook a tank exceeding it's max flow rate. This code along with Aspen SQL++ will automatically send an email when the max flow rate is exceeded.

---

> 🛡️ **Security**: All tag names, plant identifiers, and proprietary details are **blurred or removed**. Keep production IDs private.

---

## Screenshot: What You’re Looking At (Real‑World Example)

The screenshot captures two independent level sensors on an isobutylene storage tank:

* **Sensor A**: **stable** and consistent with expected process behavior
* **Sensor B**: reading \~**20% high** with visible **noise**

Both sensors had previously been considered "good" at commissioning; over time, **Sensor B degraded**. The algorithm:

1. Detected the **mismatch** between Sensor A and B
2. Quantified **noise** (standard deviation) over a recent window
3. Verified whether readings were in a realistic **range** (3–98%)
4. Tested for **stuck** behavior using short‑term deltas
5. **Automatically selected** the reliable sensor for use by downstream logic
6. Evaluated recent fill‑rate trends and set a **warning flag** if the rate exceeded `Max_Fill_Rate` (hook for email alerts)

> **Purpose of including this image:** It demonstrates the exact situation the code is built to handle: **previously good sensors can drift**. The tags are blurred to avoid disclosing plant internals, naming conventions, or safety‑critical identifiers.

---

## How the Algorithm Works (Step‑by‑Step Walkthrough)

Below is a guided tour of the provided script. Function names like `TagAverage` and `TagStatistics` refer to AspenCalc/IP21‑style history queries and stats helpers.

### 1) Compute short‑window averages (current value smoothing)

```vb
Level_1_NowAvg = (TagAverage(Level_1, Now()-0:06, Now(),  0, 0))
Level_2_NowAvg = (TagAverage(Level_2, Now()-0:06, Now(), 0, 0))
```

* **What it does:** Averages each sensor’s level over the last **6 minutes** to reduce transient noise and get a stable present value.

### 2) Baseline disagreement test

```vb
Level_difference = abs(Level_1_NowAvg - Level_2_NowAvg)
if(Level_difference > 5) then
    unknown_faulty_sensor = 1
else
    unknown_faulty_sensor = 0
endif
```

* **Idea:** If the two sensors disagree by more than 5 %, at least one is suspect.
* **Output:** `unknown_faulty_sensor` is a simple flag: *there’s a discrepancy to investigate*.

### 3) Noise quantification via rolling standard deviation

```vb
TagStatistics(Level_1, Now()-0:06, Now(), 0, 0, a, b, c, d, e, f, std1, h)
TagStatistics(Level_2, Now()-0:06, Now(), 0, 0, a1, b1, c1, d1, e1, f1, std2, h1)

if(std1 > (0.12 * Max_Fill_Rate)) then
    std_potential_sensor_1_faulty = 10
else
    std_potential_sensor_1_faulty = 0
endif

if(std2 > (0.12 * Max_Fill_Rate)) then
    std_potential_sensor_2_faulty = 10
else
    std_potential_sensor_2_faulty = 0
endif
```

* **What it does:** Computes standard deviation over the same 6‑minute window. Excess noise relative to process dynamics is a red flag.
* **Thresholding:** If `std` is above `0.12 * Max_Fill_Rate`, mark that sensor as potentially faulty due to noise.

When **both** look noisy, the code clears the weaker case so you don’t double‑count faults:

```vb
if((std_potential_sensor_2_faulty > 1) and (std_potential_sensor_1_faulty > 1) and (std2 > std1)) then
    std_potential_sensor_1_faulty = 0
endif

if((std_potential_sensor_2_faulty > 1) and (std_potential_sensor_1_faulty > 1) and (std1 > std2)) then
    std_potential_sensor_2_faulty = 0
endif
```

* **Intent:** If both are noisy, prefer the one with **less** noise.

### 4) Plausibility bounds (range checks)

```vb
if((unknown_faulty_sensor = 1) and ((Level_1_NowAvg > 98) or (Level_1_NowAvg < 3))) then
    potential_sensor_1_faulty = 15
elseif((unknown_faulty_sensor = 1) and ((Level_2_NowAvg > 98) or (Level_2_NowAvg < 3))) then
    potential_sensor_2_faulty = 14
endif
```

* **What it does:** If the sensors disagree and **one** is outside a realistic band (here **3–98%**), that one is likely wrong.

### 5) Stuck‑value detection (low motion vs. active fill)

```vb
if(( std1 > (0.05 * Max_Fill_Rate)) and (std2 < (0.05 * Max_Fill_Rate)) and (unknown_faulty_sensor = 1)) then
    potential_sensor_2_faulty = 13
endif
if(( std2 > (0.05 * Max_Fill_Rate)) and (std1 < (0.05 * Max_Fill_Rate)) and (unknown_faulty_sensor = 1)) then
    potential_sensor_1_faulty = 12
endif
```

* **Meaning:** If one sensor shows process motion (higher variance) and the other barely changes during a period **when they disagree**, the low‑motion sensor may be **stuck**.
* **Note in code:** This can trigger during no‑fill conditions—but the downstream logic only **cares near max fill rate**.

### 6) Second stuck test using short‑term deltas (1‑hour lookback)

```vb
if((unknown_faulty_sensor = 1) and ((abs(Level_1_NowAvg- (TagAverage(Level_1,  Now()-1:06, Now()-1:00,  0, 0)))) < (0.1 * Max_Fill_Rate)) and (abs(Level_2_NowAvg- (TagAverage(Level_2,  Now()-1:06, Now()-1:00,  0, 0))) > (0.1 * Max_Fill_Rate))) then
    potential_sensor_1_faulty = 11
endif

if((unknown_faulty_sensor = 1) and ((abs(Level_2_NowAvg- (TagAverage(Level_2,  Now()-1:06, Now()-1:00,  0, 0)))) < (0.1 * Max_Fill_Rate)) and (abs(Level_1_NowAvg- (TagAverage(Level_1,  Now()-1:06, Now()-1:00,  0, 0))) > (0.1 * Max_Fill_Rate))) then
    potential_sensor_2_faulty = 10
endif
```

* **What it does:** Compares current 6‑min averages to values **60–66 minutes ago**. If one barely changed (< 10% of max rate) while the other changed significantly (> 10% of max rate), the low‑change sensor may be stuck.

### 7) Choosing which sensor to use

```vb
if((potential_sensor_1_faulty > 0) and (potential_sensor_2_faulty = 0)) then
    use_sensor_2 = 10
endif
if((potential_sensor_2_faulty > 0) and (potential_sensor_1_faulty = 0)) then
    use_sensor_1 = 10
endif
if((std_potential_sensor_1_faulty > 0) and (potential_sensor_2_faulty = 0)) then
    use_sensor_2 = 11
endif
if((std_potential_sensor_2_faulty > 0) and (potential_sensor_1_faulty = 0)) then
    use_sensor_1 = 11
endif
if((potential_sensor_1_faulty = 0) and (potential_sensor_2_faulty = 0) and (std_potential_sensor_1_faulty = 0) and (std_potential_sensor_2_faulty = 0)) then
    use_sensor_1 = 12
endif
if ((potential_sensor_2_faulty > 0) and (potential_sensor_1_faulty > 0)) then
    use_sensor_1 = 13
endif
```

* **Interpretation:**

  * If **only** Sensor 1 looks bad → **use Sensor 2** (and vice‑versa)
  * If only a **noise** rule trips for one sensor → prefer the other
  * If **neither** looks bad → default to Sensor 1 (`use_sensor_1 = 12`)
  * If **both** look questionable → conservative default to Sensor 1 (`use_sensor_1 = 13`) *(tunable policy)*

### 8) One‑sensor availability edge cases

```vb
if(Level_1 = 0) then
    use_sensor_1 = 0
endif
if(Level_1 = 0) then
    use_sensor_2 = 16
endif
if(Level_2 = 0) then
    use_sensor_1 = 17
endif
if(Level_2 = 0) then
    use_sensor_2 = 0
endif

if((use_sensor_1 > 0) and (use_sensor_2 > 0)) then
    use_sensor_2 = 0
endif
if((use_sensor_1 = 0) and (use_sensor_2 = 0)) then
    use_sensor_1 = 18
endif
```

* **What it does:** Handles conditions where **one tag may be absent/invalid** (equals zero here). Ensures you never end with *both* selected or *none* selected.

### 9) Fill‑rate warning (email alert hook)

```vb
if(use_sensor_1 > 1) then
    if ((TagAverage(Level_1, Now()-0:06, Now(), 0, 0) -
         TagAverage(Level_1, Now()-1:06, Now()-1:00, 0, 0)) >= Max_Fill_Rate) then
        Tank_Fill_Rate_Warning_60m = 1
    else
        Tank_Fill_Rate_Warning_60m = 0
    endif
endif

if(use_sensor_2 > 1) then
    if ((TagAverage(Level_2, Now()-0:06, Now(), 0, 0) -
         TagAverage(Level_2, Now()-1:06, Now()-1:00, 0, 0)) >= Max_Fill_Rate) then
        Tank_Fill_Rate_Warning_60m = 1
    else
        Tank_Fill_Rate_Warning_60m = 0
    endif
endif
```

* **What it does:** Using the **selected** sensor, compares present 6‑min average to the value **60–66 minutes prior**. If the increase ≥ `Max_Fill_Rate`, assert `Tank_Fill_Rate_Warning_60m`.
* **How to use:** Wire this flag to your alarm/email mechanism.

---

## Signals/Flags Summary

* `unknown_faulty_sensor` — simple disagreement detector (> 5 units)
* `std_potential_sensor_1_faulty`, `std_potential_sensor_2_faulty` — noise‑based suspicion
* `potential_sensor_1_faulty`, `potential_sensor_2_faulty` — out‑of‑range / stuck / delta‑based suspicion
* `use_sensor_1`, `use_sensor_2` — *selection* indicators (nonzero means selected; specific values indicate which rule fired)
* `Tank_Fill_Rate_Warning_60m` — **alert**: fill exceeds `Max_Fill_Rate` over \~1 hour
