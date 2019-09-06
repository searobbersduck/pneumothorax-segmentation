# SIIM-ACR Pneumothorax Segmentation

# First place solution 

## Model Zoo
- AlbuNet (resnet34) from [\[ternausnets\]](https://github.com/ternaus/TernausNet)
- Resnet50 from [\[selim_sef SpaceNet 4\]](https://github.com/SpaceNetChallenge/SpaceNet_Off_Nadir_Solutions/tree/master/selim_sef/zoo)
- SCSEUnet (seresnext50) from \[[selim_sef SpaceNet 4\]](https://github.com/SpaceNetChallenge/SpaceNet_Off_Nadir_Solutions/tree/master/selim_sef/zoo)

## Main Features
### Triplet scheme of inference
Let our sementation model output some mask with probabilities of pneumothorax pixels. Let's name this mask as source sigmoid mask. 
I used triplet of different thresholds: *(top_score_threshold, min_contour_area, bottome_score_threshold)* 

Decision rule based on doublet *(top_score_threshold, min_contour_area)* used instead of pneumathorax/non-pneumathorax classification models:
- *top_score_threshold* is simple binarization threshold and transform source sigmoid mask into a discrete mask of zeros and ones.
- *min_contour_area* is maximum allowed number of pixels with value greater than *top_score_threshold*

Those images that didn't pass this doublet of thresholds were counted non-pneumathorax images. 

For the remaining pneumathorax images we binarize source sigmoid mask using *bottome_score_threshold* - yet another binarizetion threshold. 

Simplified version of this scheme:
```python
classification_mask = predicted > top_score_threshold
mask = predicted.copy()
mask[classification_mask.sum(axis=(1,2,3)) < min_contour_area, :,:,:] = np.zeros_like(predicted[0])
mask = mask > bot_score_threshold
return mask
```

### Search best triplet thresholds during validation 
- Best triplet on validation: (0.75, 2000, 0.3).
- Best triplet on Public Leaderboard: (0.7, 600, 0.3)

### Combo loss
Used \[[combo loss\]](https://github.com/SpaceNetChallenge/SpaceNet_Off_Nadir_Solutions/blob/master/selim_sef/training/losses.py) combinations of BCE, dice and focal. In the best experiments the weights of (BCE, dice, focal), that I used were:
- (3,1,4);
- (1,1,1);
- (2,1,2) respectively.
 
### Sliding sample rate
Let's name portion of pneumathorax images as sample rate.

**Main idea:** control this portion using sampler of torch dataset. 

On each epoch my sampler get all images from dataset with pneumathorax and sample some from non-pneumathorax according to this sample rate. During train process we reduce this parameter from 0.8 on start to 0.4 in the end.

Large sample rate at the beginning provides a quick start of learning process, whereas a small sample rate provides better convergence of neural network weights to the initial distribution of pneumathorax/non-pneumathorax images.

### Augmentations
Used following transforms from \[[albumentations\]](https://github.com/albu/albumentations)
```python
albu.Compose([
    albu.HorizontalFlip(),
    albu.OneOf([
        albu.RandomContrast(),
        albu.RandomGamma(),
        albu.RandomBrightness(),
        ], p=0.3),
    albu.OneOf([
        albu.ElasticTransform(alpha=120, sigma=120 * 0.05, alpha_affine=120 * 0.03),
        albu.GridDistortion(),
        albu.OpticalDistortion(distort_limit=2, shift_limit=0.5),
        ], p=0.3),
    albu.ShiftScaleRotate(),
    albu.Resize(img_size,img_size,always_apply=True),
])
```

### Checkpoints averaging
top3 checkpoints averaging from each pipeline on inference

### Horizontal flip TTA

## Data Preparation

## Install
```bash
pip install -r requirements.txt
```

## Pipeline launch example
Training:
```bash
python Train.py experiments/albunet_valid/train_config_part0.yaml
```
Inference:
```bash
python Inference.py experiments/albunet_valid/2nd_stage_inference.yaml
```
Submit:
```bash
python TripletSubmit.py experiments/albunet_valid/2nd_stage_submit.yaml
```

## Best experiments:
- AlbunetPublic - best model for Public Leaderboard
- AlbunetValid - best resnet34 model on validation
- seunet - best seresnext50 model on validation
- resnet50 - best resnet50 model on validation


## Final Submission
My best model for Public Leaderboard was AlbunetPublic (PL: 0.8871), and score of all ensembling models was worse.
But I suspected overfitting for this model therefore both final submissions were ensembles.

- First ensemble believed in Public Leaderboard scores more and used more "weak" triplet thresholds.
- Second ensemble believed in the validation scores more, but used more "strict" triplet thresholds.

Private Leaderboard:
- 0.8679
- 0.8641


