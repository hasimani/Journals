The Original:

## Zown


I've worked in a real estate startup called [Zown](zown.ca), where I owned the entire ML pipelines. I started with maintaining the property evaluation pipeline which was to be honest written well. So I almost kept the structure and every once in a while added a new feature and retrained the model (TabNet). Then, I implemented a recommender system for real estate listings end-to-end.


### Image Tagging Model


before getting into the details of the recommender system I'm going to bring up fun fact. As a startup, we received a bunch of free credit packaged from every major cloud provider (AWS, GCP and Azure). One day our CTO realized that we had a tons of credits on Azure that he forgot about and they were expiring in like a week. So he appointed one of the engineers to make some use of that and that guy scrambled and thought of bunch of ideas, one of which was to use Azure foundry API to label all of the listing images that we had in our database. The labels included the location of the image (e.g. bedroom, bathroom, living room, exterior front) and a number of scores rating various aspects of the images (e.g. lighting, appliances, flooring). Once I became aware of this dataset (after laughing for some time), I thought of a side project which was fine tuning a vision model (a pretrained google ViT) to reproduce the labels. So it had a classification head and one head for scoring each of the attributes I listed above. This model is was added to the ingestion pipeline to produce the same output for new listing photos. This mattered because we'll see below how these visual attributes are used in the recommender system.


### Project Matching


The recommender system was named Project Matching and it was conceptualized along the lines of "Tinder but for homes". The UI literally had users left and right swipe the candidates.


The pipeline started from filtering the listings based on the user's preferences. The main attributes like property type, number of bedrooms and bathrooms are hard filtered by default to obtain candidates, but user could add other attributes like amenities and basement type to the hard filters as well (no ML until here). Then a group of diverse candidates are presented to user to cold start the recommendation (the Exploration Set). The rating of each of the candidates in the exploration set is heuristically determined and it's not determined solely on the left or right swipe. Factors like the photo on which the user swiped (the user can see multiple pictures of the listing before swiping which are ordered based on the labels: exterior front -> living room -> kitchen -> bedroom ...), how fast they swiped left, etc. impacted the rating. Then, a two tower approach is employed. On the item tower the property features (all the features that are not hard filtered like the location, exposure, ...) and pooling of the image attributes are input to learn the item embedding and on the user tower the embedding of each listing is concatenated to the embedding of the user rating to that listing. The user/item scores are calculated by the dot product of user embedding and item embedding. The towers are trained simultaneously together and continuously by gathering more reactions from the users

## Zown

I worked at a real estate startup called [Zown](zown.ca), where I owned the machine learning pipelines end-to-end. I began by managing the property evaluation pipeline. Because the existing architecture was fundamentally sound, I preserved the core structure, periodically engineering new features and retraining the TabNet model. Following this, I developed an end-to-end recommender system for real estate listings.

### Image Tagging Model

Before detailing the recommender system, our image tagging pipeline is worth noting. Leveraging expiring cloud credits, our team ran our entire historical database of listing images through Azure's Vision APIs. This generated labels identifying the room type (e.g., bedroom, bathroom, living room, exterior) along with quality scores for various attributes (e.g., lighting, appliances, flooring).

To operationalize this without relying on continuous API calls, I used this dataset to fine-tune a pre-trained Vision Transformer (Google ViT). The architecture utilized multi-task learning: one classification head for the room type and separate regression heads for scoring the visual attributes. This model was integrated directly into our ingestion pipeline to process new listing photos, serving as downstream features for the recommender system.

### Project Matching

The recommender system, named Project Matching, featured a swipe-based user interface designed for discovering homes.

The pipeline initiated by hard-filtering listings based on rigid user constraints (e.g., property type, bedroom/bathroom counts, and specific amenities) to generate a candidate pool. To address the cold-start problem, users were presented with an Exploration Set consisting of diverse candidates.

Instead of relying strictly on binary swipe data, user preference ratings for the Exploration Set were calculated using interaction heuristics—such as the specific photo displayed when the swipe occurred (photos were ordered logically: exterior -> living room -> kitchen, etc.) and the dwell time before a swipe.

For the recommendation engine, I implemented a two-tower architecture. The item tower processed property metadata (excluding hard-filtered features) and pooled image attributes to generate item embeddings. The user tower processed the embeddings of previously evaluated listings concatenated with the user's explicit/implicit ratings for those listings. The final user-item affinity score was computed via the dot product of the user and item embeddings. The system utilized continuous online training to update the towers simultaneously as new user reactions were captured.

---

## Architectural & Process Critiques

Building ML in a startup often requires scrappy decision-making, but several practices described in the original text introduce significant technical debt and model risk. Here are the problematic practices and how to correct them.

### 1. Ad-Hoc Model Retraining

**The Practice:** *"every once in a while added a new feature and retrained the model"*
Adding features and retraining models on an arbitrary schedule leads to inconsistent performance, makes debugging regressions difficult, and ignores underlying data drift.
**The Recommendation:** Implement a Continuous Training (CT) pipeline. Retraining should be triggered programmatically by one of three things:

* **Time:** A strict cron schedule (e.g., weekly).
* **Performance:** A drop in online metrics (e.g., CTR or prediction accuracy falling below a threshold).
* **Data Drift:** Statistical shifts in the underlying feature distributions.

### 2. Blind Knowledge Distillation

**The Practice:** *"use Azure... API to label all of the listing images... fine tuning a vision model... to reproduce the labels"*
Using a third-party API to generate ground truth labels without human validation means your ViT model is learning the API's mistakes, biases, and edge-case failures. You are treating noisy pseudo-labels as gold standards.
**The Recommendation:** Always keep a "human in the loop" for evaluation. Before fine-tuning, sample a few hundred API-labeled images and have a human verify them to calculate the API's error rate. Create a purely human-annotated "Golden Test Set" to evaluate your fine-tuned ViT. This ensures you are measuring actual real-world accuracy, not just how well your model mimics Azure.

### 3. Engineering Heuristics into the Target Variable

**The Practice:** *"The rating... is heuristically determined... Factors like the photo on which the user swiped... how fast they swiped left, etc. impacted the rating."*
Manually weighting implicit signals (like dwell time or photo index) to create a target "rating" score introduces massive human bias. If your heuristic is flawed, the model will optimize perfectly for a flawed metric, leading to poor actual recommendations.
**The Recommendation:** Let the model learn the weights. Keep the target variable clean (e.g., a binary 1 for a right swipe, 0 for a left swipe). Pass the contextual data (dwell time, photo index at time of swipe) as **features** into the user tower or a secondary ranking model. If you must use heuristics to blend implicit/explicit feedback, rigorously A/B test the heuristic formulas against user retention metrics.

### 4. Unsafeguarded Continuous Training

**The Practice:** *"The towers are trained simultaneously together and continuously by gathering more reactions from the users."*
True online learning (updating weights continuously on live data streams) is highly susceptible to data poisoning, feedback loops, and catastrophic forgetting. A bot or a few erratic users can instantly skew the model's embeddings for everyone.
**The Recommendation:** Move to **micro-batch training**. Gather user reactions into batches (e.g., hourly or daily) and train offline. Before promoting the newly updated weights to production, pass the model through an automated validation gate to ensure metrics on a holdout set haven't degraded.

### 5. Infrastructure Driven by Expiring Credits

**The Practice:** *"scrambled and thought of bunch of ideas... to make some use of [expiring credits]"*
Building permanent infrastructure (the ingestion pipeline, the ViT model) because of a temporary financial anomaly violates lean engineering principles. It forces the team to maintain technical debt indefinitely for a feature that wasn't prioritized by product needs.
**The Recommendation:** Only promote hackathon or "sunk-cost" projects to production if they pass a strict ROI evaluation. If the visual attributes don't demonstrably improve the downstream recommendation metrics during offline testing, the ViT should be discarded rather than permanently added to the ingestion pipeline.