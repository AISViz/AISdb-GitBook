---
icon: message-bot
---

# Building a RAG Chatbot

One of the most potent applications enabled by LLMs is the development of question-answering chatbots. These are applications that can answer questions about specific source information. This tutorial demonstrates how to build a chatbot that can answer questions about AISViz documentation using a technique known as Retrieval-Augmented Generation (RAG).

## Overview&#x20;

A typical RAG application has two main components:&#x20;

* **Indexing**: a pipeline for scraping data from documentation and indexing it.&#x20;
  * This usually happens offline.&#x20;
* **Retrieval and generation**: the actual RAG chain, which takes the user query at runtime and retrieves the relevant data from the index, then passes it to the model.&#x20;

The most common complete sequence from raw docs to answer looks like:&#x20;

### Indexing

1. **Scrape**: First, we need to scrape all documentation pages. This includes the GitBook documentation and related pages.
2. **Split**: Text splitters break large documents into smaller chunks. This is useful both for indexing data and passing it into a model, since large chunks are harder to search over and won't fit in a model's finite context window.
3. **Store**: We need somewhere to store and index our splits, so that they can be searched over later. This is done using the Chroma vector database and embeddings.

### Retrieval and generation

1. Retrieve: Given a user input, relevant splits are retrieved from Chroma using similarity search.
2. Generate: An LLM produces an answer in response to a system prompt that combines both the question and the retrieved context.

## Setup

### Installation

This tutorial requires these dependencies:

```sh
pip install langchain chromadb openai gradio beautifulsoup4 requests
```

### API Keys

You'll need a GOOGLE LLM API key (or another LLM provider). Set it as an environment variable:

```sh
export GOOGLE_API_KEY="your-api-key-here"
```

### Components

We need to select three main components:

* **LLM**: We'll use Google's Gemini models through LangChain
* **Embeddings**: Hugging Face SentenceTransformers for creating document embeddings
* **Vector Store**: Chroma for storing and searching document embeddings

<pre class="language-python"><code class="lang-python"><strong>import getpass
</strong>import os

from sentence_transformers import SentenceTransformer
from langchain.chat_models import init_chat_model

if not os.environ.get("GOOGLE_API_KEY"):
  os.environ["GOOGLE_API_KEY"] = getpass.getpass("Enter API key for Google Gemini: ")

model = init_chat_model("gemini-2.5-flash", model_provider="google_genai")
embeddings_model = SentenceTransformer('all-MiniLM-L6-v2')
</code></pre>

## Preview

We can create a simple indexing pipeline and RAG chain to do this in about 100 lines of code.

```python
import os
import getpass

from bs4 import BeautifulSoup
import requests

from langchain.chat_models import init_chat_model
from langchain_community.vectorstores import Chroma
from langchain.docstore.document import Document
from langchain.chains import RetrievalQA
from langchain.prompts import PromptTemplate

from sentence_transformers import SentenceTransformer
from langchain.embeddings import HuggingFaceEmbeddings


# ----------------------
# Setup API keys
# ----------------------
if not os.environ.get("GOOGLE_API_KEY"):
    os.environ["GOOGLE_API_KEY"] = getpass.getpass("Enter API key for Google Gemini: ")

# ----------------------
# Initialize models
# ----------------------
# Use HuggingFace wrapper so LangChain understands SentenceTransformer
embeddings_model = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")

# Gemini model (via LangChain)
llm = init_chat_model("gemini-2.5-flash", model_provider="google_genai")

# ----------------------
# Initialize Chroma vector store
# ----------------------
persist_directory = "./chroma_db"
vectorstore = Chroma(
    collection_name="aisviz_docs",
    embedding_function=embeddings_model,
    persist_directory=persist_directory,
)

# ----------------------
# Scraper (example)
# ----------------------
def scrape_aisviz_docs(base_url="https://aisviz.example.com/docs"):
    """
    Scrape AISViz docs and return list of LangChain Documents.
    Adjust selectors based on actual site structure.
    """
    response = requests.get(base_url)
    response.raise_for_status()
    soup = BeautifulSoup(response.text, "html.parser")

    docs = []
    for section in soup.find_all("div", class_="doc-section"):
        text = section.get_text(strip=True)
        docs.append(Document(page_content=text, metadata={"source": base_url}))

    return docs

# Example: index docs once
def build_index():
    docs = scrape_aisviz_docs()
    vectorstore.add_documents(docs)
    vectorstore.persist()

# ----------------------
# Retrieval QA
# ----------------------
QA_PROMPT = PromptTemplate(
    input_variables=["context", "question"],
    template="""
You are an assistant for question-answering tasks about AISViz documentation. 
Use the following context to answer the user’s question. 
If the answer is not contained in the context, say you don't know.

Context:
{context}

Question:
{question}

Answer concisely:
""",
)

retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=retriever,
    chain_type="stuff",
    chain_type_kwargs={"prompt": QA_PROMPT},
    return_source_documents=True,
)

# ----------------------
# Usage
# ----------------------
def answer_question(question: str):
    result = qa_chain({"query": question})
    return result["result"], result["source_documents"]


if __name__ == "__main__":
    # Uncomment below line if first time indexing
    # build_index()

    ans, sources = answer_question("What is AISViz used for?")
    print("Answer:", ans)
    print("Sources:", [s.metadata["source"] for s in sources])

```

## Detailed walkthrough

### 1. Indexing

**Scraping Documentation**

We need to first scrape all the AISViz documentation pages.

```python
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin, urlparse
import time

def scrape_aisviz_docs():
    """Scrape all AISViz documentation pages"""
    
    base_urls = [
        "https://aisviz.gitbook.io/documentation",
        "https://aisviz.cs.dal.ca",
    ]
    
    scraped_content = []
    visited_urls = set()
    
    for base_url in base_urls:
        # Get the main page
        response = requests.get(base_url)
        soup = BeautifulSoup(response.content, 'html.parser')
        # Extract main content
        main_content = soup.find('div', class_='page-content') or soup.find('main')
        if main_content:
            text = main_content.get_text(strip=True)
            scraped_content.append({
                'url': base_url,
                'title': soup.find('title').get_text() if soup.find('title') else 'AISViz Documentation',
                'content': text
            })
        
        # Find all links to other documentation pages
        links = soup.find_all('a', href=True)
        for link in links:
            href = link['href']
            full_url = urljoin(base_url, href)
            
            # Only follow links within the same domain
            if urlparse(full_url).netloc == urlparse(base_url).netloc and full_url not in visited_urls:
                visited_urls.add(full_url)
                try:
                    time.sleep(0.5)  # Be respectful to the server
                    page_response = requests.get(full_url)
                    page_soup = BeautifulSoup(page_response.content, 'html.parser')
                    
                    page_content = page_soup.find('div', class_='page-content') or page_soup.find('main')
                    if page_content:
                        text = page_content.get_text(strip=True)
                        scraped_content.append({
                            'url': full_url,
                            'title': page_soup.find('title').get_text() if page_soup.find('title') else 'AISViz Page',
                            'content': text
                        })
                except Exception as e:
                    print(f"Error scraping {full_url}: {e}")
                    continue

    return scraped_content

# Scrape all documentation
docs_data = scrape_aisviz_docs()
print(f"Scraped {len(docs_data)} pages from AISViz documentation")
```

**Splitting documents**

Our scraped documents can be quite long, so we need to split them into smaller chunks. We'll use a simple text splitter that breaks documents into chunks of specified size with some overlap.

```python
def split_text(text, chunk_size=1000, chunk_overlap=200):
    """Split text into overlapping chunks"""
    chunks = []
    start = 0
    
    while start < len(text):
        # Find the end of this chunk
        end = start + chunk_size
        
        # If this isn't the last chunk, try to break at a sentence or word boundary
        if end < len(text):
            # Look for sentence boundary
            last_period = text.rfind('.', start, end)
            last_newline = text.rfind('\n', start, end)
            last_space = text.rfind(' ', start, end)
            
            # Use the best boundary we can find
            if last_period > start + chunk_size // 2:
                end = last_period + 1
            elif last_newline > start + chunk_size // 2:
                end = last_newline
            elif last_space > start + chunk_size // 2:
                end = last_space
        
        chunk = text[start:end].strip()
        if chunk:
            chunks.append(chunk)
        # Move start position for next chunk (with overlap)
        start = end - chunk_overlap if end < len(text) else end
    
    return chunks

# Split all documents into chunks
all_chunks = []
chunk_metadata = []

for doc_data in docs_data:
    chunks = split_text(doc_data['content'])
    for i, chunk in enumerate(chunks):
        all_chunks.append(chunk)
        chunk_metadata.append({
            'source': doc_data['url'],
            'title': doc_data['title'],
            'chunk_id': i
        })

print(f"Split documentation into {len(all_chunks)} chunks")
```

**Storing documents with SentenceTransformers**

Now we need to create embeddings for our chunks using Hugging Face SentenceTransformers and store them in the Chroma vector database.

```python
from sentence_transformers import SentenceTransformer
import chromadb

# Initialize SentenceTransformer model
embeddings_model = SentenceTransformer('all-MiniLM-L6-v2')

# Create Chroma client and collection
chroma_client = chromadb.PersistentClient(path="./chroma_db")
collection = chroma_client.get_or_create_collection(name="aisviz_docs")

# Create embeddings for all chunks (this may take a few minutes)
print("Creating embeddings for all chunks...")
chunk_embeddings = embeddings_model.encode(all_chunks, show_progress_bar=True)
# Prepare data for Chroma
chunk_ids = [f"chunk_{i}" for i in range(len(all_chunks))]
metadatas = chunk_metadata

# Add everything to Chroma collection
collection.add(
    embeddings=chunk_embeddings.tolist(),
    documents=all_chunks,
    metadatas=metadatas,
    ids=chunk_ids
)

print("Documents indexed and stored in Chroma database")
```

This completes the **Indexing** portion of the pipeline. At this point, we have a queryable vector store containing the chunked contents of all the documentation with embeddings created by SentenceTransformers. Given a user question, we should be able to return the most relevant snippets.

### 2. Retrieval and Generation

Now let's write the actual application logic. We aim to create a simple function that takes a user question, searches for relevant documents using SentenceTransformers embeddings, and generates an answer using Google Gemini. A high-level breakdown:

* **API Key Setup** – Load LLM (Gemini in this example) API key from environment or prompt user.
* **Model Initialization** – Wrap Google Gemini (`gemini-2.5-flash`) with LangChain.
* **Embedding** – Convert the user's question into a vector with SentenceTransformers.
* **Retrieval** – Query Chroma to fetch the top-k most relevant document chunks.
* **Context Building** – Assemble retrieved docs + metadata into a context string.
* **Prompting LLM** – Combine system + user prompts and send to LLM.
* **Answer Generation** – Return a concise response along with sources and context.

<pre class="language-python"><code class="lang-python">import getpass
import os

# ----------------------
# 1. API Key Setup
# ----------------------
# Check if GOOGLE_API_KEY is already set in the environment.
# If not, securely prompt the user to enter it (won’t show in terminal).
if not os.environ.get("GOOGLE_API_KEY"):
    os.environ["GOOGLE_API_KEY"] = getpass.getpass("Enter API key for Google Gemini: ")

from langchain.chat_models import init_chat_model

# ----------------------
# 2. Initialize Gemini Model (via LangChain)
# ----------------------
# We create a chat model wrapper around Gemini.
# "gemini-2.5-flash" is a lightweight, fast generation model.
# LangChain handles the API calls under the hood.
model = init_chat_model("gemini-2.5-flash", model_provider="google_genai")


# ----------------------
# 3. Define RAG Function
# ----------------------
def answer_question(question, k=4):
    """
    Answer a question using Retrieval-Augmented Generation (RAG)
    over AISViz documentation stored in ChromaDB.
    
    Args:
        question (str): User question.
        k (int): Number of top documents to retrieve.
    Returns:
        dict with keys: 'answer', 'sources', 'context'.
    """

    # Step 1: Create an embedding for the question
    # Encode the question into a dense vector using SentenceTransformers.
    # This embedding will be used to search for semantically similar docs.
    question_embedding = embeddings_model.encode([question])

    # Step 2: Retrieve relevant documents from Chroma
    # Query the Chroma vector store for the top-k most relevant docs.
    results = collection.query(
        query_embeddings=question_embedding.tolist(),
        n_results=k
    )

    # Step 3: Build context from retrieved documents
    # Extract both content and metadata for each retrieved doc.
    retrieved_docs = results['documents'][0]
<strong>    retrieved_metadata = results['metadatas'][0]
</strong>
    # Join all documents into one context string.
    # Each doc is prefixed with its source/title for attribution.
    context = "\n\n".join([
        f"Source: {meta.get('title', 'AISViz Documentation')}\n{doc}"
        for doc, meta in zip(retrieved_docs, retrieved_metadata)
    ])

    # Step 4: Construct prompts
    # System prompt: instructs Gemini to behave like a doc-based assistant.
    system_prompt = """You are an assistant for question-answering tasks about AISViz documentation and maritime vessel tracking. 
    Use the following pieces of retrieved context to answer the question. 
    If you don't know the answer based on the context, just say that you don't know. 
    Keep the answer concise and helpful.
    Always mention which sources you're referencing when possible."""

    # User prompt: combines the retrieved context with the actual user question.
    user_prompt = f"""Context:
{context}

Question: {question}

Answer:"""

    # Step 5: Generate answer using Gemini
    # Combine system + user prompts and send to Gemini.
    full_prompt = f"{system_prompt}\n\n{user_prompt}"

    try:
        response = gemini_model.generate_content(full_prompt)
        answer = response.text  # Extract plain text from Gemini response
    except Exception as e:
        # If Gemini API call fails, catch error and return message
        answer = f"Sorry, I encountered an error generating the response: {str(e)}"

    # Return a structured result: answer text, sources, and raw context.
    return {
        'answer': answer,
        'sources': [meta.get('source', '') for meta in retrieved_metadata],
        'context': context
    }


# ----------------------
# 4. Test the Function
# ----------------------
result = answer_question("What is AISViz?")
print("Answer:", result['answer'])
print("Sources:", result['sources'])

</code></pre>

### 3. Building a Gradio Interface

Now let's create a simple web interface using Gradio so others can interact with our chatbot:

```python
import gradio as gr

def chatbot_interface(message, history):
    """Interface function for Gradio chatbot"""
    try:
        result = answer_question(message)
        response = result['answer']
        
        # Add source information to the response
        if result['sources']:
            unique_sources = list(set(result['sources'][:3]))
            sources_text = "\n\n**Sources:**\n" + "\n".join([f"- {source}" for source in unique_sources])
            response += sources_text
            
        return response
    except Exception as e:
        return f"Sorry, I encountered an error: {str(e)}"

# Create Gradio interface
demo = gr.ChatInterface(
    fn=chatbot_interface,
    title="🚢 AISViz Documentation Chatbot",
    description="Ask questions about AISViz, AISdb, and maritime vessel tracking!",
    examples=[
        "What is AISViz?",
        "How do I get started with AISdb?",
        "What kind of data does AISViz work with?",
        "How can I analyze vessel trajectories?"
    ],
    retry_btn=None,
    undo_btn="Delete Previous",
    clear_btn="Clear History",
)

if __name__ == "__main__":
    demo.launch()
```

The full code can be found here: [https://huggingface.co/spaces/mapslab/AISVIZ-BOT/tree/main](https://huggingface.co/spaces/mapslab/AISVIZ-BOT/tree/main)&#x20;

## Below is a working chatbot for testing!

{% embed url="https://huggingface.co/spaces/mapslab/AISVIZ-BOT/tree/main" %}
