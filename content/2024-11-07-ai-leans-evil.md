+++
title = "Why Artificial Intelligence Leans Evil"
date = 2024-11-07
draft = false

[taxonomies]
tags = ["AI/ML", "Opinion"]

[extra]
author = "Baxter Eaves"
type = "post"
image = "/img/robodemon.webp"
image_align = "0 -10rem"
+++

We are worried about AI.
Apart from the evident environmental damage from expending [vast amounts of energy](https://www.technologyreview.com/2024/09/28/1104588/sorry-ai-wont-fix-climate-change/) training and querying models, which has reached such magnitude that big AI groups are now buying [*their own nuclear reactors*](https://www.bbc.com/news/articles/c748gn94k95o) to power them, or the evident financial damage from reallocating *billions* of dollars ([and *trillions* if Altman has his way](https://www.wsj.com/tech/ai/sam-altman-seeks-trillions-of-dollars-to-reshape-business-of-chips-and-ai-89ab3db0)) to technology of [questionable value](https://i.redd.it/865q5otx0d2d1.jpeg), people are worried about what AI is being used for.
We are worried about our children growing up in the age of the algorithm, in which every application on every device customizes itself to keep us engaged and generating revenue, often by degenerating our mental states; making us [angry](https://freemanmag.tulane.edu/2024/11/06/rage-clicks-study-shows-how-political-outrage-fuels-social-media-engagement/), [weak-willed narcissists](https://www.frontiersin.org/journals/psychiatry/articles/10.3389/fpsyt.2023.1083214/full).
We are worried about the eminent regression to sub-mediocrity in which AI-generated garbage overwhelms and drowns out legitimate and beautiful art created by human hearts, minds, and hands. We are worried about becoming beholden to AI; to becoming the man in the box of the [mechanical Turk](https://en.wikipedia.org/wiki/Mechanical_Turk); giving credit to machines for skills and tasks we've fought hard to master and being [thrown under the bus when they fail](https://gizmodo.com/ibm-watson-reportedly-recommended-cancer-treatments-tha-1827868882). We are worried about living in a post-truth world in which we are unable to believe anything we see or hear on a screen or through a speaker [[1](https://reutersinstitute.politics.ox.ac.uk/digital-news-report/2024/public-attitudes-towards-use-ai-and-journalism), [2](https://www.forbes.com/sites/jonathanbrill/2022/12/19/ai-is-creating-a-post-truth-world/), [3](https://www.wired.com/story/generative-ai-deepfakes-disinformation-psychology/), [4](https://www.economist.com/leaders/2024/01/18/ai-generated-content-is-raising-the-value-of-trust?utm_medium=cpc.adword.pd&utm_source=google&ppccampaignID=17210591673&ppcadID=&utm_campaign=a.22brand_pmax&utm_content=conversion.direct-response.anonymous&gad_source=1&gclsrc=ds), &c].

And while AI has given us a lot to worry about, it has done little for us. AI has largely been difficult to use for good. It is [failing healthcare](https://oatmealhealth.com/why-has-ai-failed-so-far-in-healthcare-despite-billions-of-investment/). It is [failing science](https://www.forbes.com/sites/jessedamiani/2024/05/31/will-ai-change-scientific-research-for-the-better-or-worse/). That is not to say that it has done no good. For one, AI has been used to good effect in [protein structure prediction](https://www.nobelprize.org/prizes/chemistry/2024/press-release/#:~:text=The%20Nobel%20Prize%20in%20Chemistry%202024%20is%20about%20proteins%2C%20life's,%3A%20predicting%20proteins'%20complex%20structures), though one could argue that it is also [failing in drug design](https://www.theglobeandmail.com/business/article-artificial-intelligence-drug-research-hype/) and [does little to progress our understanding of mechanisms and pathways](https://pmc.ncbi.nlm.nih.gov/articles/PMC9442638/). Regardless, I feel it is generally uncontroversial to suggest that AI leans evil.

Why? Because there are more constraints on being good. When we do good we must be careful. We must understand what is going to happen and why because what we do matters. We want to do no harm, so we must understand risk; we must understand uncertainty. To do this we must generate a transparent model of the thing we're trying to do and the environment in which we're trying to do it, and build a correctly calibrated uncertainty surrounding the outcomes of our actions. AI cannot do this. AI has scaled by taking a decidedly counter-human path: by way of mindless complexity[^1]. Whereas people seek to explain the world in the simplest terms, AI (today) seeks to overwhelm the complexity in the world with even greater parametric complexity. 

"How many parameters do we need?"

"Probably [1.76 Trillion](https://the-decoder.com/gpt-4-has-a-trillion-parameters/)[^2] should be enough?"

This complexity has made AI models opaque and brittle. And, not only brittle but unpredictably brittle, meaning that any uncertainty they do generate (yes, even with conformal prediction) is likely unrepresentative. When we do evil we do not care about the outcomes of our actions. In fact, the more chaos the better. And certainly, we have seen AI put to better use in domains where understanding is unnecessary and failure is inconsequential[^3].

AI does not lean evil because it has some sort of science fiction malevolence (or benevolent malevolence like reducing the threat humans pose to themselves), AI leans evil because of fundamental methodological flaws that make it incompatible with doing good. We are not going to fix this by continuing on our current endless cycle of trying to patch a deeply flawed approach, in turn creating a new deeply flawed approach that needs further patching. The only way we can fix this is by doing things entirely differently.

<br/>
<br/>

### Footnotes
<br/>

[^1]: If you do not agree with my use of "*mindless*", I'd like to remind you that [Google is building a freaking nuclear reactor](https://www.bbc.com/news/articles/c748gn94k95o).

[^2]: For reference, they say that there are around [86 billion neurons in the human brain](https://www.nature.com/scitable/blog/brain-metrics/are_there_really_as_many/).

[^3]: A quick trick to determine whether a task is inconsequential is to determine whether it can be completely automated away or if a human needs to remain in the loop to assume liability.

