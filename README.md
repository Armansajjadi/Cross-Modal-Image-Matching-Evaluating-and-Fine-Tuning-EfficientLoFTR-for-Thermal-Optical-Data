# EfficientLoFTR for Thermal-Optical Image Matching

Evaluation and fine-tuning of **EfficientLoFTR (ELoFTR)** on multimodal thermal-optical image pairs, with a quantitative and qualitative comparison against a classical feature matching pipeline (ORB + RANSAC).

This project was completed as a technical assignment for Prof. Hasan F. Ates, Özyeğin University.

## Overview

Image matching between thermal (infrared) and optical (visible) images is challenging because the two modalities capture very different signals (heat vs. reflected light), so classical handcrafted descriptors often fail to find reliable correspondences. This project investigates whether a modern deep learning matcher, **EfficientLoFTR**, can close this gap, both out of the box (zero-shot) and after fine-tuning on a small thermal-optical dataset.

**Pipeline:**
1. Run a classical baseline: ORB keypoint matching + RANSAC geometric verification.
2. Run EfficientLoFTR zero-shot (pretrained on the optical-only MegaDepth dataset).
3. Fine-tune EfficientLoFTR on a thermal-optical dataset, in two stages (backbone-only, then end-to-end).
4. Compare all three methods on inlier ratio, reprojection accuracy, and runtime.

## Method

### 1. Classical Baseline (ORB + RANSAC)
- ORB keypoint detection and description.
- Brute-force Hamming matching.
- Homography estimation with RANSAC for geometric (outlier) verification.

### 2. EfficientLoFTR (Zero-Shot)
- Pretrained weights from the [official EfficientLoFTR repository](https://github.com/zju3dv/efficientloftr), trained on the optical-only MegaDepth dataset.
- Applied directly to thermal-optical pairs, with no further training, to measure out-of-the-box cross-modal generalization.

### 3. EfficientLoFTR (Fine-Tuned)
Fine-tuning is performed in two stages on the thermal-optical dataset:
- **Stage 1:** the Transformer/attention layers are frozen; only the CNN backbone is trained. This adapts the low-level feature extractor to the thermal domain without disturbing the pretrained matching layers.
- **Stage 2:** the entire network is unfrozen and trained end-to-end, starting from the best Stage 1 checkpoint.

Supervision uses synthetic, known ground-truth homographies: the optical image is randomly warped, so the exact transformation between the pair is always available for both training supervision and evaluation.

## Dataset

The assignment suggested the **LasHeR** thermal-visible tracking dataset. Given the dataset size and the available compute, this project instead uses **RoadScene**, a smaller paired infrared/visible dataset, as recommended as an alternative in the assignment brief.

- Repository: https://github.com/jiayi-ma/RoadScene
- Each sample pair consists of an infrared image and a corresponding low-resolution visible image, split 80% / 10% / 10% into train / validation / test sets.

## Evaluation Metrics

| Metric | Description |
|---|---|
| Total matches | Raw number of correspondences found before geometric filtering |
| RANSAC inliers | Number of correspondences consistent with the estimated homography |
| Inlier ratio | RANSAC inliers / total matches |
| MMA @ 3px | Mean Matching Accuracy: fraction of matches whose reprojection error (vs. the ground-truth homography) is below 3 pixels — used as a proxy for pose/reprojection accuracy |
| Speed (FPS) | Frames processed per second, measured per image pair |

## Repository Structure

```
.
├── ELoFTR_Thermal_Optical_Matching.ipynb   # Main notebook: setup, baseline, zero-shot, fine-tuning, evaluation
├── efficientloftr/                         # Cloned EfficientLoFTR repository (model code + pretrained weights)
├── RoadScene/                              # Dataset (infrared / visible image pairs + ground truth)
├── eloftr_roadscene_stage1_best.pt         # Best Stage 1 checkpoint (backbone fine-tuning)
├── eloftr_roadscene_stage2_best.pt         # Best Stage 2 checkpoint (full fine-tuning) — final model
└── README.md
```

## Results

Evaluation on the RoadScene test set:

| Method | Avg. Total Matches | Avg. RANSAC Inliers | Inlier Ratio (%) | MMA @ 3px (%) | Speed (FPS) |
|---|---|---|---|---|---|
| Classical (ORB) | 411.4 | 7.7 | 2.2% | 5.59% | 11.4 |
| Pretrained ELoFTR | 847.5 | 323.6 | 32.0% | 71.76% | 6.2 |
| Fine-Tuned ELoFTR | 1600.6 | 701.2 | 44.8% | 76.35% | 9.1 |

## Discussion

- **Classical (ORB + RANSAC):** ORB finds the most candidate keypoints relative to its inlier count (411 matches) but almost all of them are wrong: only 2.2% survive RANSAC, and matching accuracy (MMA @ 3px) is just 5.6%. This confirms that a handcrafted binary descriptor, designed for same-modality matching, cannot bridge the thermal-optical appearance gap. It is the fastest method (11.4 FPS), but the matches it produces are not reliable enough for downstream tasks such as registration or pose estimation.

- **Pretrained ELoFTR (zero-shot):** even though it was trained only on optical images (MegaDepth) and never saw a thermal image during training, it already reaches a 32.0% inlier ratio and 71.76% MMA, far above ORB. This shows that a learned, dense matcher generalizes much better across modalities than handcrafted features, likely because it matches on learned semantic/structural patterns rather than raw pixel-level descriptors. The cost is speed: 6.2 FPS, since the model is much heavier than ORB.

- **Fine-tuned ELoFTR:** after the two-stage fine-tuning (frozen-backbone stage, then full end-to-end stage) on thermal-optical pairs, both reliability and quantity of matches improve further: inlier ratio rises to 44.8% and MMA @ 3px to 76.35%, with almost twice as many total matches as the pretrained model (1600.6 vs. 847.5). Interestingly, fine-tuning also recovers some speed (9.1 FPS vs. 6.2 FPS pretrained); this is likely because the fine-tuned model produces more confident, well-separated matches, which can reduce the relative overhead of post-processing/RANSAC compared to the noisier pretrained outputs. Fine-tuning required additional dataset preparation (synthetic homography supervision) and training time, but the accuracy gain over the zero-shot model is clear and consistent across all three quality metrics.

**Overall:** the results follow the expected order — Classical ORB < Pretrained ELoFTR < Fine-Tuned ELoFTR — on every accuracy metric (inlier ratio, MMA), confirming both that learned dense matching is far better suited to cross-modal thermal-optical matching than handcrafted features, and that domain-specific fine-tuning provides a further, measurable improvement on top of zero-shot transfer.

## References

- Wang, Y. et al. *Efficient LoFTR: Semi-Dense Local Feature Matching with Sparse-Like Speed*. [Project page](https://zju3dv.github.io/efficientloftr/) | [Code](https://github.com/zju3dv/efficientloftr)
- RoadScene dataset: https://github.com/jiayi-ma/RoadScene

## Author

Arman — B.Sc. Computer Engineering student. Prepared for Prof. Hasan F. Ates, Özyeğin University.
