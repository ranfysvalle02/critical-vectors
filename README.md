# bvr-py

---

# Extracting Relevant Text Chunks Using Embeddings and Clustering

---

## Introduction

When dealing with large volumes of text, it's often necessary to extract the most relevant or representative parts for tasks like summarization, topic modeling, or information retrieval. Manually sifting through extensive text can be time-consuming and impractical. To address this challenge, we introduce **CriticalVectors**, a Python class that automates the selection of significant text chunks using embeddings and clustering algorithms.

This tool leverages the power of natural language processing (NLP) and machine learning to:

- Split text into manageable chunks.
- Compute embeddings for each chunk to capture semantic meaning.
- Cluster the embeddings to identify groups of similar chunks.
- Select representative chunks from each cluster.

In this blog post, we'll delve into how CriticalVectors works, its features, and how you can use it in your projects.

## Key Features

- **Flexible Text Splitting**: Split text into sentences or paragraphs based on your preference.
- **Embeddings with Ollama**: Utilize the `OllamaEmbeddings` model with `'nomic-embed-text'` to compute embeddings.
- **Clustering Strategies**: Choose between KMeans or Agglomerative clustering to group similar text chunks.
- **Automatic Cluster Determination**: Automatically determine the optimal number of clusters based on the data.
- **FAISS Integration**: Optionally use Facebook's FAISS library for efficient clustering on large datasets.

## How It Works

### 1. Text Splitting

The input text is split into smaller chunks to make processing manageable. You can choose to split the text by sentences or paragraphs. Additionally, you can specify the maximum number of tokens per chunk to control the size.

```python
chunks = self.split_text(
    text,
    method=self.split_method,
    max_tokens_per_chunk=self.max_tokens_per_chunk
)
```

### 2. Computing Embeddings

Each text chunk is transformed into an embedding vector using the `OllamaEmbeddings` model. This vector representation captures the semantic meaning of the text.

```python
embeddings = self.compute_embeddings(chunks)
```

### 3. Clustering Embeddings

The embeddings are clustered using either KMeans or Agglomerative clustering. Clustering helps group similar chunks together.

```python
if self.strategy == 'kmeans':
    selected_chunks = self._select_chunks_kmeans(chunks, embeddings, num_clusters)
elif self.strategy == 'agglomerative':
    selected_chunks = self._select_chunks_agglomerative(chunks, embeddings, num_clusters)
```

### 4. Selecting Representative Chunks

From each cluster, the chunk closest to the centroid (central point) is selected as the representative. This ensures that the selected chunks are the most representative of their respective clusters.

```python
selected_chunks = [chunks[idx] for idx in closest_indices]
```

## Installation and Requirements

Ensure you have the following installed:

- Python 3.x
- Required Python packages:
  - `numpy`
  - `faiss-cpu` (for FAISS integration)
  - `nltk`
  - `scikit-learn`
  - `urllib3`
- NLTK data (`punkt` tokenizer)

Install the Python packages using pip:

```bash
pip3 install numpy faiss-cpu nltk scikit-learn urllib3
```

Download NLTK data:

```python
import nltk
nltk.download('punkt')
```

## Usage

### Full Code

```python
import numpy as np
import faiss
from langchain_ollama import OllamaEmbeddings
import urllib.request
import urllib.error
import nltk
from nltk.tokenize import sent_tokenize, word_tokenize


class CriticalVectors:
    """
    A robust class to select the most relevant chunks from a text using various strategies,
    """

    def __init__(
        self,
        chunk_size=500,
        strategy='kmeans',
        num_clusters='auto',
        embeddings_model=None,
        split_method='sentences',
        max_tokens_per_chunk=512,
        use_faiss=False
    ):
        """
        Initializes CriticalVectors.

        Parameters:
        - chunk_size (int): Size of each text chunk in characters.
        - strategy (str): Strategy to use for selecting chunks ('kmeans', 'agglomerative').
        - num_clusters (int or 'auto'): Number of clusters (used in clustering strategies). If 'auto', automatically determine the number of clusters.
        - embeddings_model: Embedding model to use. If None, uses OllamaEmbeddings with 'nomic-embed-text' model.
        - split_method (str): Method to split text ('sentences', 'paragraphs').
        - max_tokens_per_chunk (int): Maximum number of tokens per chunk when splitting.
        - use_faiss (bool): Whether to use FAISS for clustering.
        """
        # Validate chunk_size
        if not isinstance(chunk_size, int) or chunk_size <= 0:
            raise ValueError("chunk_size must be a positive integer.")
        self.chunk_size = chunk_size

        # Validate strategy
        valid_strategies = ['kmeans', 'agglomerative']
        if strategy not in valid_strategies:
            raise ValueError(f"strategy must be one of {valid_strategies}.")
        self.strategy = strategy

        # Validate num_clusters
        if num_clusters != 'auto' and (not isinstance(num_clusters, int) or num_clusters <= 0):
            raise ValueError("num_clusters must be a positive integer or 'auto'.")
        self.num_clusters = num_clusters

        # Set embeddings_model
        if embeddings_model is None:
            self.embeddings_model = OllamaEmbeddings(model="nomic-embed-text")
        else:
            self.embeddings_model = embeddings_model

        # Set splitting method and max tokens per chunk
        self.split_method = split_method
        self.max_tokens_per_chunk = max_tokens_per_chunk

        # Set FAISS usage
        self.use_faiss = use_faiss

    def download_raw_content(self, url=''):
        if url == '':
            return "unavailable"
        try:
            with urllib.request.urlopen(url) as response:
                data = response.read().decode()
            return data
        except urllib.error.HTTPError:
            return "unavailable"
        except Exception as e:
            return f"An error occurred: {str(e)}"

    def split_text(self, text, method='sentences', max_tokens_per_chunk=512):
        """
        Splits the text into chunks based on the specified method.

        Parameters:
        - text (str): The input text to split.
        - method (str): Method to split text ('sentences', 'paragraphs').
        - max_tokens_per_chunk (int): Maximum number of tokens per chunk.

        Returns:
        - List[str]: A list of text chunks.
        """
        # Validate text
        if not isinstance(text, str) or len(text.strip()) == 0:
            raise ValueError("text must be a non-empty string.")

        if method == 'sentences':
            nltk.download('punkt', quiet=True)
            sentences = sent_tokenize(text)
            chunks = []
            current_chunk = ''
            current_tokens = 0
            for sentence in sentences:
                tokens = word_tokenize(sentence)
                num_tokens = len(tokens)
                if current_tokens + num_tokens <= max_tokens_per_chunk:
                    current_chunk += ' ' + sentence
                    current_tokens += num_tokens
                else:
                    chunks.append(current_chunk.strip())
                    current_chunk = sentence
                    current_tokens = num_tokens
            if current_chunk:
                chunks.append(current_chunk.strip())
            return chunks
        elif method == 'paragraphs':
            paragraphs = text.split('\n\n')
            chunks = []
            current_chunk = ''
            for para in paragraphs:
                if len(current_chunk) + len(para) <= self.chunk_size:
                    current_chunk += '\n\n' + para
                else:
                    chunks.append(current_chunk.strip())
                    current_chunk = para
            if current_chunk:
                chunks.append(current_chunk.strip())
            return chunks
        else:
            raise ValueError("Invalid method for splitting text.")

    def compute_embeddings(self, chunks):
        """
        Computes embeddings for each chunk.

        Parameters:
        - chunks (List[str]): List of text chunks.

        Returns:
        - np.ndarray: Embeddings of the chunks.
        """
        # Validate chunks
        if not isinstance(chunks, list) or not chunks:
            raise ValueError("chunks must be a non-empty list of strings.")

        try:
            embeddings = self.embeddings_model.embed_documents(chunks)
            embeddings = np.array(embeddings).astype('float32')  # FAISS requires float32
            return embeddings
        except Exception as e:
            raise RuntimeError(f"Error computing embeddings: {e}")

    def select_chunks(self, chunks, embeddings):
        """
        Selects the most relevant chunks based on the specified strategy.

        Parameters:
        - chunks (List[str]): List of text chunks.
        - embeddings (np.ndarray): Embeddings of the chunks.

        Returns:
        - List[str]: Selected chunks.
        """
        num_chunks = len(chunks)
        num_clusters = self.num_clusters

        # Automatically determine number of clusters if set to 'auto'
        if num_clusters == 'auto':
            num_clusters = max(1, int(np.ceil(np.sqrt(num_chunks))))
        else:
            num_clusters = min(num_clusters, num_chunks)

        if self.strategy == 'kmeans':
            return self._select_chunks_kmeans(chunks, embeddings, num_clusters)
        elif self.strategy == 'agglomerative':
            return self._select_chunks_agglomerative(chunks, embeddings, num_clusters)
        else:
            # This should not happen due to validation in __init__
            raise ValueError(f"Unknown strategy: {self.strategy}")

    def _select_chunks_kmeans(self, chunks, embeddings, num_clusters):
        """
        Selects chunks using KMeans clustering.

        Parameters:
        - chunks (List[str]): List of text chunks.
        - embeddings (np.ndarray): Embeddings of the chunks.
        - num_clusters (int): Number of clusters.

        Returns:
        - List[str]: Selected chunks.
        """
        if self.use_faiss:
            try:
                d = embeddings.shape[1]
                kmeans = faiss.Kmeans(d, num_clusters, niter=20, verbose=False)
                kmeans.train(embeddings)
                D, I = kmeans.index.search(embeddings, 1)
                labels = I.flatten()
            except Exception as e:
                raise RuntimeError(f"Error in FAISS KMeans clustering: {e}")
        else:
            try:
                from sklearn.cluster import KMeans
                kmeans = KMeans(n_clusters=num_clusters, random_state=1337)
                kmeans.fit(embeddings)
                labels = kmeans.labels_
            except Exception as e:
                raise RuntimeError(f"Error in KMeans clustering: {e}")

        # Find the closest chunk to each cluster centroid
        try:
            if self.use_faiss:
                centroids = kmeans.centroids
                index = faiss.IndexFlatL2(embeddings.shape[1])
                index.add(embeddings)
                D, closest_indices = index.search(centroids, 1)
                closest_indices = closest_indices.flatten()
            else:
                from sklearn.metrics import pairwise_distances_argmin_min
                closest_indices, _ = pairwise_distances_argmin_min(kmeans.cluster_centers_, embeddings)
            selected_chunks = [chunks[idx] for idx in closest_indices]
            return selected_chunks
        except Exception as e:
            raise RuntimeError(f"Error selecting chunks: {e}")

    def _select_chunks_agglomerative(self, chunks, embeddings, num_clusters):
        """
        Selects chunks using Agglomerative Clustering.

        Parameters:
        - chunks (List[str]): List of text chunks.
        - embeddings (np.ndarray): Embeddings of the chunks.
        - num_clusters (int): Number of clusters.

        Returns:
        - List[str]: Selected chunks.
        """
        try:
            from sklearn.cluster import AgglomerativeClustering
            clustering = AgglomerativeClustering(n_clusters=num_clusters)
            labels = clustering.fit_predict(embeddings)
        except Exception as e:
            raise RuntimeError(f"Error in Agglomerative Clustering: {e}")

        selected_indices = []
        for label in np.unique(labels):
            cluster_indices = np.where(labels == label)[0]
            cluster_embeddings = embeddings[cluster_indices]
            centroid = np.mean(cluster_embeddings, axis=0).astype('float32').reshape(1, -1)
            # Find the chunk closest to the centroid
            if self.use_faiss:
                index = faiss.IndexFlatL2(embeddings.shape[1])
                index.add(cluster_embeddings)
                D, I = index.search(centroid, 1)
                closest_index_in_cluster = I[0][0]
            else:
                from sklearn.metrics import pairwise_distances_argmin_min
                closest_index_in_cluster, _ = pairwise_distances_argmin_min(centroid, cluster_embeddings)
                closest_index_in_cluster = closest_index_in_cluster[0]
            selected_indices.append(cluster_indices[closest_index_in_cluster])

        selected_chunks = [chunks[idx] for idx in selected_indices]
        return selected_chunks

    def get_relevant_chunks(self, text):
        """
        Gets the most relevant chunks from the text.

        Parameters:
        - text (str): The input text.

        Returns:
        - List[str]: Selected chunks.
        """
        # Split the text into chunks
        chunks = self.split_text(
            text,
            method=self.split_method,
            max_tokens_per_chunk=self.max_tokens_per_chunk
        )

        if not chunks:
            return [], '', ''

        # first part
        first_part = chunks[0]
        # last part
        last_part = chunks[-1]

        # Compute embeddings for each chunk
        embeddings = self.compute_embeddings(chunks)

        # Select the most relevant chunks
        selected_chunks = self.select_chunks(chunks, embeddings)
        return selected_chunks, first_part, last_part

# Example usage:

if __name__ == "__main__":

    # Instantiate the selector
    try:
        selector = CriticalVectors(
            strategy='kmeans',
            num_clusters='auto',
            chunk_size=1000,
            split_method='sentences',
            max_tokens_per_chunk=100,  # Adjust as needed
            use_faiss=True  # Enable FAISS
        )
        test_str = selector.download_raw_content("https://raw.githubusercontent.com/ranfysvalle02/vanilla-agents/refs/heads/main/README.md")
        # Get the most relevant chunks using the improved method
        relevant_chunks, first_part, last_part = selector.get_relevant_chunks(test_str)
        print(first_part)
        print("======================")
        print("Selected Chunks:")
        for chunk in relevant_chunks:
            print(chunk)
        # Print the last part
        print("======================")
        print(last_part)
    except Exception as e:
        print(f"An error occurred: {e}")

```

### Example Script

```python
if __name__ == "__main__":
    # Instantiate the selector
    try:
        selector = CriticalVectors(
            strategy='kmeans',
            num_clusters='auto',
            chunk_size=1000,
            split_method='sentences',
            max_tokens_per_chunk=100,  # Adjust as needed
            use_faiss=True  # Enable FAISS
        )
        # Download content from a URL
        test_str = selector.download_raw_content(
            "https://raw.githubusercontent.com/yourusername/yourrepository/main/README.md"
        )
        # Get the most relevant chunks
        relevant_chunks, first_part, last_part = selector.get_relevant_chunks(test_str)
        print("First Part:")
        print(first_part)
        print("======================")
        print("Selected Chunks:")
        for chunk in relevant_chunks:
            print(chunk)
            print("----------------------")
        print("======================")
        print("Last Part:")
        print(last_part)
    except Exception as e:
        print(f"An error occurred: {e}")
```

### Output

```
First Part:
# Your Project Title

Introduction text from the README...

======================
Selected Chunks:
[Chunk 1]
----------------------
[Chunk 2]
----------------------
[Chunk N]
----------------------
======================
Last Part:
Concluding text from the README...
```

## Code Explanation

### Initialization

The `CriticalVectors` class is initialized with several parameters:

- `chunk_size`: The size of each text chunk in characters.
- `strategy`: The clustering strategy (`'kmeans'` or `'agglomerative'`).
- `num_clusters`: The number of clusters or `'auto'` to determine automatically.
- `embeddings_model`: The embeddings model to use. Defaults to `OllamaEmbeddings` with `'nomic-embed-text'`.
- `split_method`: The method to split text (`'sentences'` or `'paragraphs'`).
- `max_tokens_per_chunk`: The maximum number of tokens per chunk.
- `use_faiss`: Whether to use FAISS for clustering.

```python
selector = CriticalVectors(
    strategy='kmeans',
    num_clusters='auto',
    chunk_size=1000,
    split_method='sentences',
    max_tokens_per_chunk=100,
    use_faiss=True
)
```

### Downloading Content

The `download_raw_content` method fetches text data from a given URL.

```python
test_str = selector.download_raw_content(
    "https://raw.githubusercontent.com/yourusername/yourrepository/main/README.md"
)
```

### Splitting Text

The `split_text` method divides the text into chunks based on the specified method.

```python
chunks = self.split_text(
    text,
    method=self.split_method,
    max_tokens_per_chunk=self.max_tokens_per_chunk
)
```

### Computing Embeddings

Embeddings for each chunk are computed using the specified embeddings model.

```python
embeddings = self.compute_embeddings(chunks)
```

### Selecting Chunks

The most relevant chunks are selected based on the clustering strategy.

```python
selected_chunks = self.select_chunks(chunks, embeddings)
```

### Handling Clustering Internals

#### KMeans Clustering

If using KMeans, FAISS can be utilized for efficient computation.

```python
if self.use_faiss:
    kmeans = faiss.Kmeans(d, num_clusters, niter=20, verbose=False)
    kmeans.train(embeddings)
else:
    kmeans = KMeans(n_clusters=num_clusters, random_state=1337)
    kmeans.fit(embeddings)
```

#### Agglomerative Clustering

For Agglomerative clustering, scikit-learn's implementation is used.

```python
clustering = AgglomerativeClustering(n_clusters=num_clusters)
labels = clustering.fit_predict(embeddings)
```

## Potential Applications

- **Text Summarization**: Extract key sentences to create a summary.
- **Topic Modeling**: Identify representative chunks for different topics.
- **Information Retrieval**: Quickly retrieve relevant information from large texts.
- **Preprocessing for NLP Tasks**: Prepare data by selecting significant parts.

---

## Appendix:

### `max_tokens_per_chunk`

- **Definition**: The maximum number of tokens allowed in each text chunk.
- **Purpose**: Controls the size of each chunk based on token count rather than character count. This is important because models often have token limits.
- **Usage**: When splitting text, sentences are added to a chunk until adding another sentence would exceed the `max_tokens_per_chunk`.
- **Example**: If `max_tokens_per_chunk` is set to `100`, each chunk will contain up to 100 tokens.

```python
max_tokens_per_chunk=100  # Adjust as needed
```

### `num_clusters`

- **Definition**: The number of clusters to form during the clustering step. Can be an integer or `'auto'`.
- **Purpose**: Determines how many representative chunks will be selected.
- **Usage**:
  - If set to an integer, that exact number of clusters will be created.
  - If set to `'auto'`, the number of clusters is determined automatically based on the data.
- **Automatic Calculation**: When `'auto'`, the number of clusters is calculated as:

```python
num_clusters = max(1, int(np.ceil(np.sqrt(num_chunks))))
```

### `chunk_size`

- **Definition**: The size of each text chunk in characters.
- **Purpose**: Controls the maximum size of a chunk when splitting by paragraphs.
- **Usage**: Primarily used when `split_method` is set to `'paragraphs'`. Chunks are formed by combining paragraphs until the `chunk_size` limit is reached.
- **Example**: If `chunk_size` is `1000`, each chunk will contain up to 1000 characters.

```python
chunk_size=1000
```

### `split_method`

- **Definition**: The method used to split the text into chunks. Options are `'sentences'` or `'paragraphs'`.
- **Purpose**: Determines how the text is divided, affecting the granularity of the chunks.
- **Options**:
  - `'sentences'`: Splits the text into sentences and then forms chunks based on `max_tokens_per_chunk`.
  - `'paragraphs'`: Splits the text into paragraphs and then forms chunks based on `chunk_size`.
- **Usage**:

```python
split_method='sentences'  # or 'paragraphs'
```

### Example Usage of Parameters

```python
selector = CriticalVectors(
    strategy='kmeans',
    num_clusters='auto',       # Automatically determine the number of clusters
    chunk_size=1000,           # Maximum size of chunks in characters when using paragraphs
    split_method='sentences',  # Split the text by sentences
    max_tokens_per_chunk=100,  # Maximum number of tokens per chunk
    use_faiss=True             # Use FAISS for efficient clustering
)
```

### Interaction Between Parameters

- When `split_method` is `'sentences'`, the `max_tokens_per_chunk` parameter is used to control chunk sizes.
- When `split_method` is `'paragraphs'`, the `chunk_size` parameter is used instead.
- The `num_clusters` parameter affects how many chunks will be selected as the most relevant.

---
