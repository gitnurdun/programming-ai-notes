# Classic Computer Vision Papers Collection (2000-2025): Part 10

## 91. "CLIP-Adapter: Better Vision-Language Models with Feature Adapters" by Gao et al. (2022)

### Introduction
CLIP-Adapter proposed a lightweight adaptation approach that adds feature adapters to pre-trained CLIP models for improved transferability to downstream tasks.

### Key Contributions
1. **Feature adapters**: Lightweight modules for adaptation
2. **Parameter efficiency**: Minimal additional parameters
3. **Improved transfer**: Better performance on downstream tasks
4. **Simple integration**: Easy to add to existing models

### Architecture
- Frozen CLIP model
- Feature adapter layers
- Task-specific adaptation
- Minimal parameter increase

### Impact
CLIP-Adapter provided an efficient way to adapt large pre-trained models to specific tasks.

---

## 92. "CoOp: Learning to Prompt for Vision-Language Models" by Zhou et al. (2022)

### Introduction
CoOp introduced a method for learning to prompt vision-language models, automatically learning continuous prompts that adapt pre-trained models to downstream tasks.

### Key Contributions
1. **Continuous prompting**: Learnable continuous prompts
2. **Prompt optimization**: Direct optimization of prompts
3. **Zero-shot adaptation**: Adapt without labeled data
4. **Better generalization**: Improved transfer performance

### Architecture
- Pre-trained vision-language model
- Learnable prompt tokens
- Prompt optimization
- Task-specific prompting

### Impact
CoOp advanced the field of prompt learning for vision-language models.

---

## 93. "PromptDet: Towards Open-Vocabulary Detection Using Uncurated Images" by Wang et al. (2022)

### Introduction
PromptDet introduced an open-vocabulary detection framework that can detect objects from arbitrary categories using text prompts without requiring category-specific annotations.

### Key Contributions
1. **Open-vocabulary detection**: Detect unseen categories
2. **Text prompting**: Use text descriptions for detection
3. **Weak supervision**: Learn from uncurated images
4. **Unified framework**: Single model for multiple categories

### Architecture
- Vision-language alignment
- Text-guided detection
- Open-vocabulary classification
- Unified detection framework

### Impact
PromptDet advanced open-vocabulary object detection significantly.

---

## 94. "DetCLIP: Dictionary-Enriched Visual-Concept Paralleled Learning for Open-Vocabulary Object Detection" by Zhong et al. (2022)

### Introduction
DetCLIP developed a framework for open-vocabulary object detection by learning visual-concept correspondences with enriched dictionaries.

### Key Contributions
1. **Dictionary enrichment**: Expand concept vocabulary
2. **Visual-concept alignment**: Better concept representation
3. **Open-vocabulary detection**: Detect unseen categories
4. **Strong performance**: State-of-the-art open-vocabulary results

### Architecture
- Dictionary enrichment module
- Visual-concept alignment
- Open-vocabulary detection head
- Multi-modal learning

### Impact
DetCLIP advanced the state-of-the-art in open-vocabulary detection.

---

## 95. "Open-Vocabulary Semantic Segmentation with Mask-adapted CLIP" by Tam et al. (2022)

### Introduction
This work adapted CLIP for open-vocabulary semantic segmentation by learning to generate masks for arbitrary categories using text descriptions.

### Key Contributions
1. **Open-vocabulary segmentation**: Segment unseen categories
2. **Mask adaptation**: Adapt CLIP for segmentation
3. **Text-guided segmentation**: Use text descriptions for segmentation
4. **Generalizability**: Work on diverse categories

### Architecture
- CLIP-based encoder
- Mask generation module
- Text-guided segmentation
- Open-vocabulary adaptation

### Impact
This work extended open-vocabulary capabilities to semantic segmentation.

---

## 96. "Group-Free 3D Object Detection via Transformers" by Cheng et al. (2021)

### Introduction
Group-Free 3D introduced a transformer-based approach to 3D object detection that directly predicts object bounding boxes without grouping point proposals.

### Key Contributions
1. **Group-free detection**: No need for point grouping
2. **Transformer architecture**: Use transformers for 3D detection
3. **Direct prediction**: Predict boxes directly from points
4. **Strong performance**: Competitive with grouping-based methods

### Architecture
- Point feature extraction
- Transformer encoder-decoder
- Direct box prediction
- Query-based detection

### Impact
Group-Free 3D showed that transformer-based approaches can be effective for 3D detection.

---

## 97. "3DETR: An End-to-End Transformer for 3D Object Detection" by Misra et al. (2021)

### Introduction
3DETR applied the transformer architecture to 3D object detection, providing an end-to-end solution without hand-designed components.

### Key Contributions
1. **End-to-end 3D detection**: No NMS or anchor generation
2. **Transformer architecture**: Attention mechanisms for 3D detection
3. **Point cloud processing**: Direct processing of point clouds
4. **Set prediction**: Bipartite matching for assignment

### Architecture
- Point cloud encoder
- Transformer encoder-decoder
- Object queries
- Bipartite matching
- Detection heads

### Impact
3DETR demonstrated transformer effectiveness for 3D object detection.

---

## 98. "NeRF-W: Neural Radiance Fields for Unconstrained Photo Collections" by Martin-Brualla et al. (2021)

### Introduction
NeRF-W extended NeRF to handle unconstrained photo collections with potential inconsistencies and transient objects.

### Key Contributions
1. **Weighted loss**: Handle inconsistent training views
2. **Temporal modeling**: Model dynamic scenes
3. **Robust optimization**: Handle unconstrained collections
4. **Dynamic scene handling**: Deal with transient objects

### Architecture
- Per-view appearance codes
- Per-point transient/static decomposition
- Weighted volume rendering
- Dynamic scene modeling

### Impact
NeRF-W made NeRF applicable to more realistic photo collections.

---

## 99. "K-Planes: Explicit Radiance Fields in Space, Time, and Appearance" by Tancik et al. (2023)

### Introduction
K-Planes represented radiance fields using explicit plane-based decompositions, providing an efficient and interpretable alternative to MLP-based NeRFs.

### Key Contributions
1. **Plane-based representation**: Decompose into explicit planes
2. **Efficient computation**: Faster training and rendering
3. **Interpretable representation**: Explicit geometric decomposition
4. **Strong performance**: Competitive with NeRF variants

### Architecture
- Canonical planes in 3D space
- Factorized plane representations
- Efficient feature lookup
- Plane fusion for output

### Impact
K-Planes provided an efficient alternative to neural radiance fields.

---

## 100. "Gaussian Splatting from Motion Blur" by Liao et al. (2024)

### Introduction
This recent work showed how to leverage motion blur information for more accurate 3D Gaussian splatting, improving novel view synthesis quality.

### Key Contributions
1. **Motion blur utilization**: Use blur for better 3D reconstruction
2. **Blur-aware optimization**: Account for motion during training
3. **Enhanced accuracy**: Better 3D scene understanding
4. **Temporal consistency**: Improved multi-view consistency

### Architecture
- Motion blur modeling
- Gaussian splatting optimization
- Temporal consistency
- Blur-aware rendering

### Impact
This work demonstrates how classical computer vision cues can enhance modern neural representations.

---

## Conclusion

This collection of 100 classic computer vision papers spans from 2000 to 2025, covering major developments across various computer vision domains:

- **Classical methods** (SIFT, SURF, HOG, RANSAC)
- **Deep learning foundations** (AlexNet, VGG, ResNet, Inception)
- **Detection and segmentation** (R-CNN family, YOLO, Mask R-CNN)
- **Transformers in vision** (ViT, Swin, DETR)
- **Vision-language models** (CLIP, BLIP, DALL-E)
- **Diffusion models** (DDPM, Stable Diffusion, Imagen)
- **3D vision** (NeRF, PointNet, 3D Gaussian Splatting)
- **Video understanding** (TimeSformer, VideoMAE)
- **Pose estimation** (PoseFormer, RAFT)
- **Open-vocabulary learning** (Segment Anything, Open-Vocabulary Detection)

These papers represent the evolution of computer vision from classical feature engineering to modern deep learning and foundation models. They demonstrate key concepts and techniques that have shaped the field and continue to influence current research directions.

The progression shows how the field has moved from:
- Hand-crafted features → Learned representations
- Task-specific models → General foundation models
- Single modality → Multi-modal understanding
- Supervised learning → Self-supervised and foundation modeling
- Pixel-level tasks → Semantic understanding

This systematic collection provides a comprehensive overview of the major developments in computer vision over the past 25 years.