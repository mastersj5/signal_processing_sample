# Time History Association: Web-Scale Signal Matching Pipeline

This repository contains a clean, optimized, and reproducible prototype for matching short, experimental time-history signal observations with a library of long-duration reference signals. 

## 1. Problem Overview
An experimental sensor dataset contains **40 short-duration observations** (signals of 50 samples). We need to associate each observation with its top 2 closest matches from a library of **4 distinct long-duration reference signals** (signals of 100 samples) and estimate their relative match probabilities. 

Because the observations only cover a sub-portion of the reference signal's duration, the algorithm must slide the short signal across the long signal to find the optimal alignment before scoring.

---

## 2. Design Methodology & Algorithmic Rationale
During the exploration phase, we prototyped and compared two fundamental signal processing techniques:

### Technique A: Sliding-Window Mean Squared Error (MSE)
* **How it works**: This brute-force approach slides the experimental signal index-by-index across the reference signal. At each offset, it computes the mean squared difference between the normalized signals:
  $$\text{MSE} = \frac{1}{N}\sum_{i=1}^{N} (x_i - y_i)^2$$
* **Intuition**: This is highly intuitive because it directly measures physical distance (L2 norm) between signal values at matching timestamps.
* **Limitation**: Pure Python implementations require nested loops over the arrays, making execution time slow and inefficient for real-time scale.

### Technique B: Normalized Cross-Correlation (NCC)
* **How it works**: Normalized Cross-Correlation measures the similarity of two waveforms as a function of the displacement of one relative to the other. By standardizing both signals to have a mean of 0 and standard deviation of 1, the cross-correlation peak represents the optimal alignment:
  <img width="246" height="51" alt="image" src="https://github.com/user-attachments/assets/21028efb-a6d6-4583-b6db-e63c83ffe6a7" />
* **Intuition**: Rather than measuring absolute distance, NCC measures shape alignment and covariance. 
* **Advantage**: It leverages highly optimized compiled backends (like SciPy's `scipy.signal.correlate`), which use FFT convolutions under the hood to perform the sliding comparisons virtually instantaneously.

### Algorithmic Comparison
* **Accuracy**: Both algorithms identify **identical** optimal alignment indices.
* **Performance**: NCC is approximately **15 to 20 times faster** than sliding-window MSE in standard test loops, which makes it the chosen foundation for our production-ready pipeline.

---

## 3. Implementation & Optimization Strategy
To maximize execution efficiency for large-scale production, the following pipeline optimizations were designed:
1. **Pre-loading Data**: Rather than performing repetitive and slow Disk I/O inside the matching loops, reference files are loaded into memory once as NumPy arrays.
2. **Pre-normalization**: The reference signals are pre-standardized (mean subtracted and divided by standard deviation) during initialization. This eliminates redundant arithmetic operations during runtime.
3. **Scaled Softmax Probabilities**: To convert similarity scores into robust probabilities that isolate confident matches, we apply a scaled Softmax function:
   $$P(x_i) = \frac{e^{s \cdot x_i}}{\sum e^{s \cdot x_j}}$$
   A scaling factor ($s = 10$) is used to widen confidence margins, clearly highlighting strong matches while grouping weaker ones.

---

## 4. Performance Results & Key Findings
* **Total Execution Time**: **~13.59 ms** to process all 40 experimental signals.
* **Average Match Speed**: **~0.34 ms per signal**.
* **Example Association Outcomes**:
  * `exp_signal0.csv` matches `ref_signal2.csv` (71.38% probability) and `ref_signal0.csv` (10.13% probability).
  * `exp_signal11.csv` matches `ref_signal2.csv` (77.55% probability) and `ref_signal0.csv` (10.68% probability).

---

## 5. Deployment Recommendations for Real-Time Scaling
If deploying this system at a massive scale (e.g., millions of reference curves or high-frequency stream processing), we recommend the following evolutionary steps:
1. **GPU Acceleration**: Implement matching with PyTorch or CuPy to execute massive batch 1D convolutions on GPUs, dropping individual match speeds to microseconds.
2. **Pruning with Vector Databases**: Instead of brute-force matching against all references, index reference curves in a Vector Database (e.g., Milvus, Pinecone, or FAISS) using Locality-Sensitive Hashing (LSH) to query only the closest candidate matches.
3. **Distributed Batching**: Use a framework like Ray or Apache Spark to distribute processing of incoming sensor batches over multi-core computing nodes.

---

## 6. Project References
* [SciPy signal.correlate Documentation](https://docs.scipy.org/doc/scipy/reference/generated/scipy.signal.correlate.html)
* [Wikipedia: Cross-Correlation](https://en.wikipedia.org/wiki/Cross-correlation)
* [Wikipedia: Template Matching (SSD & NCC)](https://en.wikipedia.org/wiki/Template_matching)
