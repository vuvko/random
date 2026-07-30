# Not All Synthetic Data Is Equally Useful

*Published: 2026-07-31*

It's well known that when it comes to model quality, data is what matters most. So whenever the question comes up of where to get more data — and labeled data at that — generating synthetic data starts to look very attractive.

But if you've ever trained computer vision models purely on generated data, you've probably seen their quality drop when tested on a real domain. And that isn't an accident: the problem isn't the quality of the labels or the variety of classes — it's simply that the pixels of a real image differ slightly from a generated one. The neural networks used in computer vision latch onto a strong texture signal very quickly[^geirhos_texture_iclr2019] and, in particular, easily grab onto a generator's "fingerprint"[^cnnspot_cvpr2020].

## Domain Adaptation

The broader problem — test data differing from training data — is known as domain adaptation[^da_theory]. There are many ways to use the data you already have to make a model more robust to a domain shift. The most trivial is to mix real data into the synthetic data during training. Or to train the model so that it can't tell where an image came from[^ganin_adaptation] (somewhat similar to GAN training).

But if we don't synthesize images from scratch and instead only alter part of them through a generator (it doesn't much matter whether it's a GAN or diffusion), there's a simpler approach.

## Undo It, or the "Reverse" Trick

Editing images is a convenient way to grow a dataset — especially its rare classes. To make this concrete, imagine we want to train a model that classifies facial attributes: whether there are glasses, a hat, a mask or scarf, whether the mouth is open. Real datasets with these attributes do exist, but images covering every combination at once are scarce. This is where generators help: you can take some of the images without glasses and ask the generator to add glasses.

As we already noted, neural nets are excellent at soaking up the generator's texture signal. And in our case, if we generate a great many images with glasses, the model will learn a simple rule: "a generated image means glasses." It won't bother learning anything more meaningful (the shapes of the glasses, their position, their edges) — why would it, when there's an easier cue available.

This is exactly where the "reverse" trick helps. Having generated an image with glasses, we immediately ask the generator to "take the glasses off." Now the generator's texture signal in our data is no longer rigidly bound to a specific class.

What's happening underneath: previously "generated" meant "has glasses" — and that was the shortest possible cue for the model. After the reverse, the images *without* glasses also pass through the generator, so its fingerprint now shows up in both classes at once. A single fingerprint no longer lets you guess the class, and the model is forced to look at the glasses themselves.

The idea isn't new — people have long fought the same disease in deepfake detection. If the "real" frames were never touched by a generator while the "fake" ones were, the detector learns the generator's artifact instead of the forgery itself and fails to transfer to new generators. One fix is re-synthesis: pass the inputs through a generative transform to strip out the dependence on any specific artifact[^resynthesis_ijcai2021]. In essence, that's the same reverse.

You can also look at it from a theoretical angle. We want the prediction to be independent of a spurious feature (the generator's fingerprint) while keeping its dependence on the meaningful one (the glasses). This property is called counterfactual invariance, and it's known that this is exactly what buys robustness under a domain shift[^cf_invariance_neurips2021]. The reverse achieves it in the cheapest way possible — directly in the data, without touching the training process. Notice that the goal here is exactly the same as in the domain-adversarial approach[^ganin_adaptation] from the previous section: to keep the model from relying on where the image came from. Only instead of a separate loss, we get there by the construction of the dataset.

## Takeaways

Synthetic data is useful exactly to the extent that it doesn't hand the model an easy shortcut in place of the real signal. If you don't generate a whole image but only edit part of it, the generator's fingerprint almost always ends up bound to whichever class you were adding. The reverse breaks that binding: once you've generated an attribute, immediately take it back out again, so the generator's fingerprint lands in both classes. It's cheap, it requires no change to training, and the model starts learning what it actually should.

[^geirhos_texture_iclr2019]: Geirhos R. et al. [ImageNet-trained CNNs are biased towards texture; increasing shape bias improves accuracy and robustness](https://arxiv.org/abs/1811.12231) // International Conference on Learning Representations. — 2019.

[^cnnspot_cvpr2020]: Wang S.-Y. et al. [CNN-generated images are surprisingly easy to spot... for now](https://openaccess.thecvf.com/content_CVPR_2020/html/Wang_CNN-Generated_Images_Are_Surprisingly_Easy_to_Spot..._for_Now_CVPR_2020_paper.html) // Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition. — 2020. — pp. 8695–8704.

[^da_theory]: Ben-David S. et al. [A theory of learning from different domains](https://link.springer.com/content/pdf/10.1007/s10994-009-5152-4.pdf) // Machine Learning. — 2010. — Vol. 79, no. 1. — pp. 151–175.

[^ganin_adaptation]: Ganin Y., Lempitsky V. [Unsupervised domain adaptation by backpropagation](https://proceedings.mlr.press/v37/ganin15.html) // International Conference on Machine Learning. — PMLR, 2015. — pp. 1180–1189.

[^resynthesis_ijcai2021]: He Y. et al. [Beyond the Spectrum: Detecting Deepfakes via Re-Synthesis](https://arxiv.org/abs/2105.14376) // Proceedings of the 30th International Joint Conference on Artificial Intelligence (IJCAI). — 2021.

[^cf_invariance_neurips2021]: Veitch V. et al. [Counterfactual Invariance to Spurious Correlations in Text Classification](https://arxiv.org/abs/2106.00545) // Advances in Neural Information Processing Systems. — 2021. — Vol. 34.
