# Embedding Search, Dimensionality Reduction, and Semantic Visualization

<style>
center {
  resize: both;
  overflow: auto;
}
</style>

## Introduction

**Context**:

Currently, the UCSF Industry Documents Library uses Apache Solr's MoreLikeThis (MLT) feature to allow researchers or anybody browsing the web to discover more documents similar to the one they are currently viewing. This is important for those exploring the dataset or looking for specific information to navigate more easily. Anybody can click the "More Like This" button in the top right corner when viewing a document, and get a list of about 10 similar documents.

However, sometimes this doesn't work how we want. Apache Solr is currently configured to produce this list based on a weighted sum of fields like the industry, author, and title similarity. This doesn't take into consideration document contents or themes of documents, and while often the titles of documents are correlated with their contents, a lot of the time they are not. There are cases where the current MLT search returns no results as there aren't any documents with similar authors and names, and poor quality MLT results are much more common.

**Solution:**

While there might be many possible solutions to this problem, I believe the most elegant is to generate document embeddings, a way of taking a document and converting it to a vector that (hopefully) represents all the key information of that document. Other algorithms like TF-IDF compare word frequencies and counts, but an embedding-based search could go much deeper.

Document embeddings capture semantic content by encoding the context and meaning of words in a dense vector space. This method uses natural language processing (NLP) to (hopefully) provide more nuanced and relevant search results. Unlike TF-IDF or similar algorithms like MLT, which rely on the frequency of terms and often miss context, embeddings allow for more sophisticated comparisons that consider the semantic and thematic elements of documents, improving the accuracy and relevance of search results.

We can also generate interesting visualizations, necessitating dimensionality reductions. As the embeddings are just vectors (lists of numbers), we could imagine each one as representing a point in a high-dimensional space. Documents close to eachother in this space would also hopefully be close to eachother in meaning or theme. However, visualizing high-dimensional data requires reducing it to 2 or 3 dimensions, which makes it comprehensible to our 2 and 3-D adjusted minds. Dimensionality reduction techniques such as PCA, t-SNE, and UMAP can transform embeddings into lower-dimensional representations, enabling the creation of interactive and intuitive visualizations which help users explore the dataset more effectively, revealing clusters and patterns that might not be apparent through text alone. Additionally, the visualizations give the user an ability to grasp entire collections or databases at once, which can often consist of thousands or millions of documents in the case of UCSF.



## Embedding Search

To create an embedding search algorithm, we need two main parts: an embedding algorithm and a similarity metric. There are also nearest neighbor algorithms that are necessary to the search, but Apache Solr provides its own kNN (k-nearest neighbor) algorithm that gets approximate closest documents based on the embeddings and distance function quickly.

### Embedding Algorithms:

The first algorithm I tested was **Doc2Vec**, an extension of the Word2Vec model designed to generate vector representations of entire documents, rather than single words. It creates these embeddings by training on a corpus of provided documents and learning to predict words based on both their surrounding words and a unique document identifier. This lets Doc2Vec to produce vectors that encode the document content and structure, making it useful for various natural language processing tasks such as document classification, clustering, visualization, and embedding search.

Doc2Vec generates embeddings that are tailored to the specific corpus it is trained on, ensuring that the vectors represent the unique characteristics of the documents. We can also choose the exact size of the vector (a variable I will illustrate in the visualization section), resulting in quick and efficient similarity searches and training for documents within the training dataset. It also means that we don't have to send data to a third party or run extremely resource-intensive machine learning models, so we can save a lot of money.

However, its performance might degrade when applied to out-of-sample data, as the model might not generalize well to documents with different styles or content not seen during training. For UCSF specifically, this means that an embedding model trained on only JUUL documents wouldn't embed documents from the opioids industry very well. Additionally, in my testing it seemed that running Doc2Vec on a random sample of the entire IDL database, a lot of contextual information was lost. This is shown in the visualizations below, where Doc2Vec doesn't group 

The solution to these problems is likely training and running different Doc2Vec models for each industry or even for each collection. Then when we execute a similarity search, we first filter for only documents in the same industry, then sort by closeness to the original document according to our distance function. This would mean giving up seamless visualizations of the entire database at once, but would be significantly faster and more accurate than training and running one Doc2Vec model on the entire database.

The second major category of embedding algorithm I tested were **TogetherAI embedding models**, pretrained models used by large language models (LLMs) to generate vector representations of text, so that text can be understood and utilized by LLMs. They perform well across diverse and variable text inputs due to their pretraining on extensive and varied datasets (mostly text data from the internet). They can likely capture broader contextual relationships and nuance, making them suitable for a wider range of documents. However, they're often resource-intensive and slow, requiring significant computational power for processing and dimensionality reduction. Usually they would be used via a third party, sending data to them and receiving embeddings, but this incurs a fee and may restrict our ability to work with sensitive or unredacted data.

Additionally, there are a few other key practical barriers to using TogetherAI models. For one, they have maximum context sizes in the range of 512 tokens to 32768 tokens. However, tokens don't correlate very well with words or number of pages, and the number of tokens a document uses up can be quite variable depending on the text it is used on. For example, low quality OCR looks like gibberish and would likely take up more tokens per page than perfectly coherent OCR. As it is hard to predict the number of tokens a document will take up, I essentially used a trial and error approach, where I first tried to feed in the entire document, and if that errored because of too many input tokens, I cut the document down by 2/3 and retried it recursively. Thus, eventually the document would fit in the context size and it would return an embedding. This is not elegant at all and there are far better ways of embedding large documents (such as taking many embeddings and averaging them), but these are all complicated when submitting bulk API requests to TogetherAI.

Another practical difficulty with the TogetherAI embedding models are their dimensionality. Embedding models often produce vectors with a dimensionality of 768 or more, which would take up too much storage space and would be too slow to search through. Thus, dimensionality reduction algorithms must be applied to reduce the number of dimensions, but this warps the data.

I tested all 8 of TogetherAI's embedding models, using the $5 of free credits anybody can get by signing up for an account. It is difficult to compare their performance as we don't actually know what documents are "closer" to others, because this is a subjective judgment. The best way to get a sense of performance is to look at the visualizations, but even these are poor metrics for how the embeddings actually look and act in the high-dimensional space. However, I am interesting in developing a small dataset to test this based on human understanding of document similarity.

### Similarity Metrics:

Apache Solr includes a dense vector search, so these embeddings can be plugged right into our current database. It's important that this dense vector kNN search is approximate, as this balances speed of retrieval with accuracy of results. There are multiple similarity metrics:

**Euclidean Distance** measures the straight-line distance between two points in a multi-dimensional space, similar to the Pythagorean theorem (it is actually the Pythagorean theorem, but with more numbers squared under the square root). It is useful when the magnitude of the vectors is important, but it is the slowest of the algorithms.

**Cosine Similarity** calculates the cosine of the angle between two vectors, focusing on their direction rather than their magnitude. This metric is useful when the orientation of the vectors is more relevant than their length, such as in text similarity tasks where document size varies. Something interesting that I observed when testing is that in extremely high dimensional data (e.g. the 768 dimensions generated by TogetherAI embedding models), the 10 closest documents were the exact same when measuring distance using cosine and Euclidean distance. I believe this is related to the curse of dimensionality, as in high dimensions distances become more uniform. Overall, I believe this is the strongest of all the algorithms, and it is generally standard for these types of applications.

**Dot Product (Inner Product)** measures the length of the projection of one vector onto another, combining aspects of both magnitude and direction. It's computationally efficient and often used in scenarios where both the size and alignment of vectors matter, or when speed is necessary.

## Dimensionality Reductions

As previously mentioned, to efficiently utilize the embeddings from the TogetherAI models, we must reduce their number of dimensions from ~768 to somewhere between 10 and 100. I often defaulted to using 20 dimensions, but this was somewhat arbitrarily chosen based on what seemed to be the standard in previous work. Another application of dimensionality reduction algorithms is in visualization of data. Instead of converting to somewhere around 20 dimensions, we can convert the data down to 2 or 3 dimensions, which we can visualize with standard tools. This can also be applied to the Doc2Vec embeddings to visualize them as well.

There are a few key embedding algorithms I tested, which I'll describe below.

**PCA**

Principal Component Analysis (PCA) transforms the data into a set of orthogonal components that capture the maximum variance, reducing dimensionality while preserving as much variance as possible using eigenvalue decomposition. It's computationally efficient, very fast, and easy to apply to new data, making it a practical choice for large datasets or initial discovery work. However, PCA's linear nature means it can struggle with capturing complex, non-linear relationships like those present in document embeddings. Additionally, PCA is sensitive to outliers which can skew results significantly.

**t-SNE**

t-distributed Stochastic Neighbor Embedding (t-SNE) works by converting high-dimensional Euclidean distances into conditional probabilities that represent similarities, and then optimizing the low-dimensional representation to reflect these probabilities.

It's very good at preserving local structures in the data and can understand non-linearity, making it useful for visualizing clusters, but it has a high computational cost, scaling quadratically in both time and space with the number of data points, making it unsuitable for very large datasets. The stochastic nature of t-SNE also means that different runs can produce different results, and it commonly has a number of hyperparameters to tune, described below. Note that different implementations use different hyperparameters or names.

- **Perplexity** determines the size of the neighborhood for attracting points, with higher values causing points that are further away to be considered neighbors, typically ranging from 5 to 50.

- **Exaggeration** controls the magnitude of attraction between points in the early stages of optimization, enhancing the formation of clusters.

- **Learning Rate** influences the step size during optimization, affecting how quickly or slowly the algorithm converges to a solution.

**UMAP**

Uniform Manifold Approximation and Projection (UMAP) fundamentally works similarly to t-SNE, computing hihg-dimensional similarities and tuning a low-dimensional graph to to match it as much as possible, but it also uses sophisticated techniques from algebraic topology and cross-entropy similarity functions to speed up generation.

Like t-SNE it works well for non-linear data and is stochastic, but it is considered better at preserving the overall shape of the data and is much faster than t-SNE. However, it's effectiveness depends significantly on the tuning of its hyperparameters, and it's much more sensitive to small changes in them than t-SNE. Thus, in exchange for running each reduction faster, we must run a greater quantity of reductions testing different hyperparameters which are described below.

- **Number of Nearest Neighbors** balances the emphasis between local and global data structure, where a lower value focuses on local structure and a higher value captures more global patterns. However, if this metric is too high,  UMAP breaks down and fails to cluster data well.

- **Minimum Distance** controls how tightly points are packed together in the low-dimensional space, with higher values preventing points from clustering too closely and promoting a more even spread.

Because of its speed and the size of my datasets, I largely stuck to using PCA for initial visual investigation, and UMAP for more in-depth searching.

**Autoencoders**

The final option I considered were using autoencoders, which are neural network models designed to learn efficient codings of input data. They work by compressing the data into a lower-dimensional representation and then reconstructing it back to the original input (training to match the identity function). This process can capture complex, non-linear relationships within the data and contextual understandings. While autoencoders may provide the better dimensionality reduction, they are computationally expensive to train and run, especially on large amounts of data.

Due to the difficulty of pursuing this option, I decided to leave it to future experiments. Dimensionality reduction isn't the core of this project, and wouldn't be relevant if we used Doc2Vec (the overall best algorithm).

## Visualizations

Below are some interactive visualizations of the content and themes of our Tobacco Products Liability Project Collection. Each point is a document, and you can scroll around, click on points, and view similar points (determined by the embedding model). You can also experiment with alternative dimensionality reduction algorithms and visualizations by clicking on "UMAP" or "PCA" in the bottom left and tweaking its options. I would recommend against using t-SNE, as it is O(n^2) in time and space so it is very slow for large datasets. Sometimes it takes a few minutes to set up a visualization, so don't be surprised if it doesn't work immediately.

### Tobacco Products, BAAI-bge-base, with dimensionality reduced from 768 to 20 ahead of time (faster)

<center>
  <iframe src="https://projector.tensorflow.org/?config=https://raw.githubusercontent.com/generic-account/visualizations/main/visualization-json" style="border:5px #ffffff solid;" name="myiFrame" scrolling="no" frameborder="1" marginheight="0px" marginwidth="0px" height="800px" width="1900px" allowfullscreen></iframe>
</center>



### Tobacco products, BAAI-bge-base raw data - visualization and dimensionality reductions are up to you (slower)

<center>
  <iframe src="https://projector.tensorflow.org/?config=https://raw.githubusercontent.com/generic-account/visualizations/main/visualization-json-large" style="border:5px #ffffff solid;" name="myiFrame" scrolling="no" frameborder="1" marginheight="0px" marginwidth="0px" height="800px" width="1900px" allowfullscreen></iframe>
</center>

### All IDL sample, Doc2Vec 5 dimensions

<center>
  <iframe src="https://projector.tensorflow.org/?config=https://raw.githubusercontent.com/generic-account/visualizations/main/vismany_ids_5-data" style="border:5px #ffffff solid;" name="myiFrame" scrolling="no" frameborder="1" marginheight="0px" marginwidth="0px" height="800px" width="1900px" allowfullscreen></iframe>
</center>

### All IDL sample, Doc2Vec 100 dimensions

<center>
  <iframe src="https://projector.tensorflow.org/?config=https://raw.githubusercontent.com/generic-account/visualizations/main/vismany_ids_100-data" style="border:5px #ffffff solid;" name="myiFrame" scrolling="no" frameborder="1" marginheight="0px" marginwidth="0px" height="800px" width="1900px" allowfullscreen></iframe>
</center>

### All IDL sample, Doc2Vec 250 dimensions

<center>
  <iframe src="https://projector.tensorflow.org/?config=https://raw.githubusercontent.com/generic-account/visualizations/main/vismany_ids_250-data" style="border:5px #ffffff solid;" name="myiFrame" scrolling="no" frameborder="1" marginheight="0px" marginwidth="0px" height="800px" width="1900px" allowfullscreen></iframe>
</center>

## Conclusion

The initial implementation has demonstrated promising results, showing that embedding-based search can provide relevant document recommendations. The performance of the embedding models, particularly Doc2Vec, was quite good, often yielding recommendations similar to the original MLT algorithm, which suggests that the previous method had some merit. However, it was also able to return appropriate results when MLT did not and actually appeared to have contextual understanding. TogetherAI models, while providing better contextual understanding, often struggled with long documents, necessitating truncation, which might lead to loss of important information. Doc2Vec's efficiency and cost-effectiveness make it a strong candidate for continued use, especially when combined with pre-filtering for industry.

## Limitations

One significant challenge is scaling the system to handle larger datasets effectively. I tested on three datasets: one of 2.8k documents (San Francisco Walgreen litigation documents), 17k (Tobacco Products Liability Project collection), and one of 30k (my random sample of all the IDL documents). None of these compare to the roughly 20 million documents in the IDL currently, so there would likely be unpredictable complications when scaling.

## Next Steps and Future Investigations

Below are a few related areas I would like to research.

- **Autoencoders** - It would be interesting to study the potential of autoencoder models for generating higher-quality, context-aware embeddings specifically for our datasets and compare them to more general-purpose dimensionality reduction algorithms like UMAP and PCA. Performance would be an extremely important factor due to the size of our datasets, so I could specifically research lightweight autoencoder models.

- **Matryoshka Embedding Models** - These are machine-learning based models trained to generate valid embeddings at a range of dimension sized, capturing the most important information in the first few dimensions. These perform better than standard embedding models overall, and could eliminate the need for extra dimensionality reduction algorithms.

- **Dataset for Similarity Quality** - Compile or curate a benchmark for accuracy of document similarity algorithms. This would likely be subjective, but by averaging many opinions I could potentially generate a useful testing dataset that would allow me to systematically test and measure the performance of various embedding models, better quantify the information lost by dimensionality reductions, and formalize some hyperparameter optimization.

- **testing Scalability** - I think this would be the most useful research area, as it is incredibly relevant for UCSF. I could perform tests on embedding time, search time, model training time and space, database space, additional optimization algorithms such as prefiltering, and other relevant factors when scaling up to large datasets.

