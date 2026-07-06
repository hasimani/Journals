## Zown

I worked at a real estate startup called [Zown](zown.ca), where I owned the machine learning pipelines end-to-end. I began by managing the property evaluation pipeline. Because the existing architecture was fundamentally sound, I preserved the core structure, only adding a Continuous Training (CT) pipeline. Following this, I developed an end-to-end recommender system for real estate listings.

### Image Tagging Model

Before detailing the recommender system, our image tagging pipeline is worth noting. As a startup, we received a bunch of free credits from every cloud provider (AWS, GCP and Azure). One day our CTO realized that we had a tons of credits on Azure that he was unaware of and they were expiring shortly.

He appointed one of the engineers to make some use of them. One of his ideas was to ran our entire historical database of listing images through Azure AI Foundry APIs. This generated labels identifying the room type (e.g., bedroom, bathroom, living room, exterior) along with quality scores for various attributes (e.g., lighting, appliances, flooring).

A few hundred sample size of the API-labeled images were manually verified by domain experts (the sales team) to establish a baseline agreement rate.

Once I was made aware of this dataset, I took the initiative to operationalize these capabilities in-house. I used this dataset to fine-tune a pre-trained Vision Transformer (Google ViT). The architecture utilized multi-task learning: one classification head for the room type and separate regression heads for scoring the visual attributes. This model was integrated directly into our ingestion pipeline to process new listing photos, serving as downstream features for the recommender system.

### Project Matching

The recommender system was named Project Matching and was conceptualized as a "Tinder for homes". It featured an intuitive left-and-right swipe user interface to evaluate property candidates.

The pipeline initiated by hard-filtering listings based on rigid user constraints (e.g., property type, bedroom/bathroom counts, and specific amenities) to generate a candidate pool. To address the cold-start problem, users were presented with an Exploration Set consisting of diverse candidates.

Along with the binary swipe data, user behaviours for the Exploration Set were recorded. That includes the specific photo displayed when the swipe occurred (photos were ordered logically: exterior -> living room -> kitchen, etc.) and the dwell time before a swipe.

For the recommendation engine, I implemented a two-tower architecture. The item tower processed property metadata (excluding hard-filtered features) and pooled image attributes to generate item embeddings. The user tower processed the embeddings of previously evaluated listings concatenated with the user's explicit/implicit ratings for those listings. The final user-item affinity score was computed via the dot product of the user and item embeddings. Moreover, for obtaining a richer embedding and a more expressive model, a two-head auxiliary MLP was added to predict the recorded dwell time and engagement depth (how many photos they clicked through).

 The system utilized micro-batch offline training to update the towers. Before promoting the newly updated weights to production, the model is passed through an automated validation gate to ensure metrics on a holdout set haven't degraded.