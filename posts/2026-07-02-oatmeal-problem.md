# The Oatmeal Problem

*Published: 2026-07-03*

In the previous [post](./2026-06-30-procgen-stories.md) I've briefly touched the problem of generating a lot of simple templates.
And making an interesting variety of results out of them is a fool's errand.

Later I found a good interpretation and much more research that tries to answer the question I brushed under in my post.
Why does generating more content give you the same feeling of boredom.

## 10,000 Bowls of Oatmeal

In their post titled ["So you want to build a generator..."](https://galaxykate0.tumblr.com/post/139774965871/so-you-want-to-build-a-generator), Kate Compton gives a nice example.
Suppose you are creating a lot of oatmeal bowls.
You can guarantee to create as many distinct bowls as you like.
But from the user's perspective it would all look "just a lot of oatmeal".

Rabii and Cook took this metaphor and formalized it as the information theory problem in ["Why Oatmeal is Cheap"](https://dl.acm.org/doi/10.1145/3582437.3582484) (FDG 2023).
There is a fixed relationship between the knowledge encoded in a generator, the size of its possibility space, and the complexity of what it can produce.
Scaling the possibility space without adding encoded knowledge is cheap — it exponentially expands the number of outputs while the complexity ceiling barely moves.
The result is a large quantity of noisy, perceptually similar content.
Oatmeal is cheap because we don't add information into it.

## The Dearth of the Author

Max Kreminski connects this to large generative models in ["Endless Forms Most Similar"](https://link.springer.com/article/10.1007/s00146-025-02326-6).
When an AI tool expands a small prompt into a large output, the model makes most of the creative decisions and not the user.
The author is not absent, but their intent is stretched so thinly over the output that they are barely present.
He calls this the "dearth of the author".

The result is similar to Rabii and Cook's.
The more output scales relative to the prompt, the more it reflects the model's defaults rather than the author's choices.

The oatmeal problem is not about quantity.
It shows what constraints were put into the system, and which biases are present in the final generator.
Without adding more restrictions, we would only get more random noise.

## What to Do?

In their post, Kate Compton gave a good advice on what to do before the generator, and it is something I skipped over several times in my games: writing good and bad examples of potential generator.
It is funny to have the data-first approach be shoved into my face after my several failed attempts to circumvent it.

---

> This post is part of my [#100DaysToOffload](https://100daystooffload.com/) challenge.
