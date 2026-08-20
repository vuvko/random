---
tags: [ml]
---
# The Seam Is the Label

*Published: 2026-08-09*

[Last time](./2026-07-31-not-all-synthetic-data.md) I wrote about the "reverse" trick: if you grow a dataset by asking a generator to add glasses to a face, the generator's fingerprint ends up in exactly the images that have glasses, and the model learns "generated means glasses" instead of learning what glasses look like.
The fix was to immediately ask the generator to take the glasses off again, so that both classes carry the same fingerprint and it stops predicting anything.

That trick has a requirement I only mentioned briefly: the edit has to be invertible.
You need to be able to undo the thing you just did and keep an image you would still put in the dataset.

Object detection breaks that requirement.
You can ask a generator to un-add an object, but that pass leaves artifacts of its own.
And you rarely regenerate the whole frame anyway — you patch the region where the object goes, because that is the cheap way to do it and because copy-paste of objects into scenes is a genuinely good augmentation on its own[^copypaste].
So the fingerprint is not spread over the image.
It sits in a region.
The same region you are asking the model to draw a box around.

## Patching one region

Here is the setup, with the cat[^2] standing in for the object.
I take an ellipse around it, pass just that region through a generator, and paste it back.
I do not actually need a sophisticated generator to illustrate the point: a round trip through half resolution followed by a sharpening pass is what an upsampling layer does inside most generators, and it leaves the same kind of trace — the fine grain in the patch is not the grain the camera put there.

I will show pieces of Python along the way, so that you can reproduce any of this.
Rather than cluttering every snippet with imports, here is everything they use:

```python
import io

import numpy as np
from PIL import Image, ImageFilter
from scipy import ndimage
```

That is the whole toolbox.
The only external libraries are for matrix and simple image manipulation: `numpy`, `scipy`, and `pillow` (imported as `PIL` for historical reasons).

```python
# You can think of this function as any image manipulation/generator
def generator(img: Image.Image) -> Image.Image:
    w, h = img.size
    small = img.resize((w // 2, h // 2), Image.BICUBIC)
    back = small.resize((w, h), Image.BICUBIC)
    return back.filter(ImageFilter.UnsharpMask(radius=2, percent=110))

photo: Image.Image  # the original photo
mask: np.ndarray    # the region we are manipulating

edited = np.asarray(generator(photo), float)
original = np.asarray(photo, float)
patched = original * (1 - mask[..., None]) + edited * mask[..., None]
```

The sharpening matters. Without it the patch is simply blurry and anyone can see it; with it, the photo looks fine and only the very finest grain is missing.
And what is missing can always be written off as something the lens did.

The first thing to try is [error level analysis](https://en.wikipedia.org/wiki/Error_level_analysis): save the image again as a JPEG and look at what moved. Pixels that have already been through the compressor sit close to where it wants them and barely shift; freshly drawn ones have further to fall.
The technique is a quick way to see which parts of an image might have been manipulated.

```python
def ela(img: Image.Image, quality: int = 90) -> np.ndarray:
    buf = io.BytesIO()
    img.save(buf, "JPEG", quality=quality)
    again = np.asarray(Image.open(buf), float)
    return np.abs(np.asarray(img, float) - again).max(axis=2)
```

![Top row shows original, bottom row shows patched. The photos are the same to the eye. Error level analysis is where the oval shows up. Note that with a more sophisticated generator the boundary will not be this clean, but most of the time it is still visible.](./images/patched-objects/by-eye.webp)

## One statistic finds it better

ELA is doing well here, but it is answering a question about compression while being dominated by content, which is why it needs the control row to stay readable in a more real-world scenario.
So let's ask something closer to how capturing an image physically differs from manipulating one digitally.
Cut the image into 48-pixel windows and, in each one, measure how much of the finest detail is left — the grain right at the scale of individual pixels — compared with detail one step coarser.
A region that came back from a generator is short of the finest grain no matter what it depicts, because the generator drew it rather than the sensor recording it.

Getting the two scales cleanly apart is a small Fourier transform per window (I will write about the Fourier transform for images some other time).
You can treat the next snippet as magic.
The point is to illustrate what is possible with some clever math.

```python
gray: np.ndarray  # our photo in grayscale
WIN, STRIDE = 48, 24
H, W = gray.shape

yy, xx = np.mgrid[0:WIN, 0:WIN]
rad = np.hypot(xx - WIN // 2, yy - WIN // 2) / (WIN / 2)
finest, coarser = rad > 0.75, (rad > 0.35) & (rad < 0.6)
taper = np.outer(np.hanning(WIN), np.hanning(WIN))
grain = gray - ndimage.gaussian_filter(gray, 1.0)

def score(window: np.ndarray) -> float:
    # fft here is fast Fourier transform
    power = np.abs(np.fft.fftshift(np.fft.fft2(window * taper))) ** 2
    return np.log10(power[finest].mean()) - np.log10(power[coarser].mean())

smap = np.array([[score(grain[y:y + WIN, x:x + WIN])
                  for x in range(0, W - WIN + 1, STRIDE)]
                 for y in range(0, H - WIN + 1, STRIDE)])
```

No training[^1], no neural network, several lines of Python.
Turning that map into a box takes seven more: keep the windows that fall furthest below the median, and take the bounding box of the largest connected blob.

```python
suspicious = np.median(smap) - smap
hit = suspicious > suspicious.max() * 0.55
blobs, n = ndimage.label(hit)
biggest = blobs == 1 + np.argmax(ndimage.sum(hit, blobs, range(1, n + 1)))
ys, xs = np.nonzero(biggest)
box = (xs.min() * STRIDE, ys.min() * STRIDE,
       xs.max() * STRIDE + WIN, ys.max() * STRIDE + WIN)
```

![The maps on the right are that statistic on the same two photos as before, on the same scale, dark where the fine grain has gone missing. The untouched map has plenty of structure, but no clear blob, and the dashed box that falls out of it lands on nothing in particular, with IoU 0.10. In the patched map the ellipse is simply there. Take the largest connected group of suspicious windows, draw a box around it, and it lands on the object with IoU 0.86: an annotation recovered from a photograph by a hand-written statistic that has never seen a cat.](./images/patched-objects/localised.webp)

This is worse than the classification case, and it is worth being precise about why.
There, the fingerprint leaked the label: one bit per image.
And that can be countered either by passing every image through the same generator, or by the "reverse" trick.
Here it leaks the coordinates.
Your detector can get a good score on your synthetic set by learning "put a box around the region whose texture statistics are wrong", and it will look like it works right up until it meets a photograph in testing where nobody pasted anything.

## Three ways to hide it

But synthetic data is also how you attack the long tail.
Especially in object detection, where you want to cover as many situations as you can.
Who knows, maybe your elephant detector will have to find elephants in living rooms[^elephant].
There are strategies to compensate for generator noise.

The obvious first move is to soften the seam.
We simply blur the edge of the mask so that the boundary is not as crisp.
The next is to add decoys — paste equally-processed empty patches into places with no object, so that "patched" stops meaning "object".
The third is to process the whole frame after compositing, so the trace is everywhere and stops being a location.
But that requires a generator that can go over the whole image without destroying anything of value in it.

```python
#1 blur the seam
img = paste(photo, ellipse(box, feather=8))
#2 paste meaningless objects around
for _ in range(3):
    img = paste(img, ellipse(somewhere_random()))
#3 re-process the whole image (same generator)
img = generator(img)
```

![Feathering does not help at all: the boundary softens, the interior goes on glowing, and the box is if anything slightly better than before. Re-processing the whole frame flattens the map. Decoys leave the patches perfectly visible — there are just four of them now, and only one has a cat in it, so the box lands somewhere between them.](./images/patched-objects/mitigations.webp)

Across two dozen photos from this blog with randomly placed patches, the picture holds up.

![Feathering buys you nothing at all: 0.85 either way. Decoys leave the patch just as detectable, but they break the thing that mattered, dropping the box onto the object from 0.70 to 0.27. Re-processing the whole frame takes detectability itself down to a coin flip — and it charges you real detail in every image you own, including the ones nobody ever touched.](./images/patched-objects/hiding-the-patch.webp)

If you are building a detection dataset, decoys are the cheap answer and they are the direct analogue of the reverse trick: you are not removing the fingerprint, you are removing the correlation between the fingerprint and the label.
It needs nothing invertible from the generator.
But it requires your generator to know how to paste "nothing" onto your image.

And of course you can build more sophisticated processing pipelines, and even randomise which one is used.
That spreads more variety across your samples, and makes it harder for the network to settle on the fingerprint of any single generator.

## The same measurement, read backwards

Everything above has a mirror image in face forgery detection, where people arrived at it from the other side.

Almost every face swap ends with a blending step: the altered face has to go back into an existing photograph.
Face X-ray[^facexray] builds a detector around exactly that.
It predicts a greyscale image of the blending boundary, and because it assumes nothing except that blending happened, it generalises across manipulation methods it has never seen.
The part that should make you sit up: it works when trained purely on synthetically blended real images, having never been shown a single fake.
And never with the whole image re-processed afterwards.

Self-blended images[^sbi] push it further.
Blend a real face with a slightly perturbed copy of itself, train on that, and you get 93.18% AUC on Celeb-DF[^celebdf] without a deepfake anywhere in training.
The detector is not learning what forgeries look like.
It is learning what compositing looks like, and forgeries happen to composite.

Which means my list of mitigations is also a threat model, and the entries line up.
Feathering is the obvious attack on a boundary detector, and both my measurement and the existence of interior-statistics detectors say it does not do much.
Re-processing the whole frame is the one that actually works — against my fourteen lines of numpy and, for the same reason, against a detector trained on seams.
And some popular face-swapping pipelines already have a post-processing step specifically because of that[^dfl].

So there is no neutral ground here.
The fingerprint you cannot scrub out of your training data is the same one that makes forgery detection possible in the first place, and the work of hiding it is the same work as the work of finding it, run in the opposite direction.
It is not only a cat-and-mouse game of detectors chasing generators.

But here is the hopeful part: for training on synthetic data we do not need an undetectable seam.
We only need to make the generator's noise hard enough to use that the network goes looking for something more meaningful instead[^texture].

---

> This post is part of my [#100DaysToOffload](https://100daystooffload.com/) challenge.

[^1]: You can argue that the magic constants and the other arguments are training. And you would be right. But it is still a handful of parameters, ones you could tune by hand.

[^2]: The cat comes from my photos of the Solovki islands. There are more in the [series](./2026-07-18-solovki-part1.md) about that trip.

[^dfl]: Perov I. et al. [DeepFaceLab: Integrated, flexible and extensible face-swapping framework](https://arxiv.org/abs/2005.05535) // arXiv preprint arXiv:2005.05535. — 2020. Its [merger](https://github.com/iperov/DeepFaceLab/blob/master/merger/MergeMasked.py) degrades the destination frame down to meet the generated face rather than the other way around: `image_denoise_power` median-blurs the whole frame and `bicubic_degrade_power` downscales and re-upscales it — the very round trip I use as a stand-in generator above — both before the face is pasted in, and `color_degrade_power` then reduces the colours of the finished image.

[^elephant]: Rosenfeld A., Zemel R., Tsotsos J. K. [The Elephant in the Room](https://arxiv.org/abs/1808.03305) // arXiv preprint arXiv:1808.03305. — 2018.

[^texture]: Geirhos R. et al. [ImageNet-trained CNNs are biased towards texture; increasing shape bias improves accuracy and robustness](https://arxiv.org/abs/1811.12231) // International Conference on Learning Representations. — 2019.

[^copypaste]: Ghiasi G. et al. [Simple Copy-Paste Is a Strong Data Augmentation Method for Instance Segmentation](https://openaccess.thecvf.com/content/CVPR2021/html/Ghiasi_Simple_Copy-Paste_Is_a_Strong_Data_Augmentation_Method_for_Instance_CVPR_2021_paper.html) // Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition. — 2021. — pp. 2918–2928.

[^facexray]: Li L. et al. [Face X-ray for More General Face Forgery Detection](https://openaccess.thecvf.com/content_CVPR_2020/html/Li_Face_X-Ray_for_More_General_Face_Forgery_Detection_CVPR_2020_paper.html) // Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition. — 2020. — pp. 5001–5010.

[^sbi]: Shiohara K., Yamasaki T. [Detecting Deepfakes With Self-Blended Images](https://openaccess.thecvf.com/content/CVPR2022/html/Shiohara_Detecting_Deepfakes_With_Self-Blended_Images_CVPR_2022_paper.html) // Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition. — 2022. — pp. 18720–18729.

[^celebdf]: Li Y. et al. [Celeb-DF: A Large-Scale Challenging Dataset for DeepFake Forensics](https://openaccess.thecvf.com/content_CVPR_2020/html/Li_Celeb-DF_A_Large-Scale_Challenging_Dataset_for_DeepFake_Forensics_CVPR_2020_paper.html) // Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition. — 2020. — pp. 3207–3216.
