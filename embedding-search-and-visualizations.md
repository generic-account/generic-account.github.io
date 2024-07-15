# Embedding Search, Dimensionality Reduction, and Semantic Visualization

<style>
div {
  resize: both;
  overflow: auto;
}
</style>

## Visualizations of the Tobacco Products Liability Project Collections

Below are some interactive visualizations of the content and themes of our Tobacco Products Liability Project Collection. Each point is a document, and you can scroll around, click on points, and view similar points (determined by the embedding model). You can also experiment with alternative dimensionality reduction algorithms and visualizations by clicking on "UMAP" or "PCA" in the bottom left and tweaking its options. I would recommend against using t-SNE, as it is O(n^2) in time and space so it is very slow for large datasets. Sometimes it takes a few minutes to set up a visualization, so don't be surprised if it doesn't work immediately.

### Dimensionality reduced from 768 to 20 ahead of time (faster)

<div align="center"><iframe src="https://projector.tensorflow.org/?config=https://raw.githubusercontent.com/generic-account/visualizations/main/visualization-json" style="border:0px #ffffff none;" name="myiFrame" scrolling="no" frameborder="1" marginheight="0px" marginwidth="0px" height="800px" width="1900px" allowfullscreen></iframe></div>



### Raw data - visualization and dimensionality reductions are up to you (slower)

<div align="center"><iframe src="https://projector.tensorflow.org/?config=https://raw.githubusercontent.com/generic-account/visualizations/main/visualization-json-large" style="border:0px #ffffff none;" name="myiFrame" scrolling="no" frameborder="1" marginheight="0px" marginwidth="0px" height="800px" width="1900px" allowfullscreen></iframe></div>

