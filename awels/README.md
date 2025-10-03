```markdown
# CastShop Knowledge Base

Welcome to the CastShop Knowledge Base project. This project is designed to efficiently process, analyze, and store data from the CastShop repository. Below, you'll find detailed information about the tools utilized, processing statistics, vector database collections, and usage instructions.

## Table of Contents

1. [Project Overview](#project-overview)
2. [Processing Summary](#processing-summary)
3. [Tools Used](#tools-used)
4. [Statistics](#statistics)
5. [Usage Instructions](#usage-instructions)

## Project Overview

The CastShop Knowledge Base is a structured data storage and retrieval system optimized for handling code, documentation, and images from the CastShop project. By leveraging advanced AI tools and embedding techniques, this knowledge base provides a comprehensive and interactive experience for developers and contributors.

**Repository**: [https://github.com/fbeawels/castshop.git](https://github.com/fbeawels/castshop.git)

## Processing Summary

In processing the contents of the CastShop repository, we have focused on extracting and organizing data from different file types. This involved the application of specific tools to process code files, documentation files, and image files, resulting in a detailed vector database collection.

- **Code Files Processed**: 1374
- **Documentation Files Processed**: 0
- **Image Files Processed**: 224

## Tools Used

The project employs a variety of advanced tools to ensure accurate and efficient data processing and vectorization.

- **LLM**: 
  - OpenAI GPT-4o for context generation and code analysis
- **Embeddings**: 
  - Ollama with nomic-embed-text model for text embeddings
- **Vector Database**: 
  - Qdrant for scalable vector storage and retrieval
- **Code Analysis**: 
  - `build_code.py` script for code processing
- **Document Analysis**: 
  - `build_doc.py` script for document processing
- **Image Analysis**: 
  - `build_multi.py` script for image processing

## Statistics

The extracted data has been organized into the following vector database collections, each representing a different aspect of the repository:

| Collection Name               | Data Points |
|-------------------------------|-------------|
| cast-shopizer-code            | 1922        |
| cast-shopizer-doc             | 0           |
| cast-shopizer-multi           | 296         |

### Generated Files

To facilitate understanding and usage of the knowledge base, the following files have been generated:

- **CONTEXT.md**: Provides context about the repository
- **PROMPT.md**: Contains the system prompt for the AI agent
- **SPECS.md**: Contains specifications for creating a Langflow agent

## Usage Instructions

To effectively utilize the CastShop Knowledge Base, follow these general steps:

1. **Cloning the Repository**:
   - Clone the CastShop repository using the command:
     ```bash
     git clone https://github.com/fbeawels/castshop.git
     ```

2. **Data Processing**:
   - Use the provided scripts (`build_code.py`, `build_doc.py`, `build_multi.py`) to process new data as needed.

3. **Accessing Vector Databases**:
   - Interact with the Qdrant vector database to access processed data points for your queries and analysis.

4. **Referencing Files**:
   - Utilize the `CONTEXT.md`, `PROMPT.md`, and `SPECS.md` files for guidance and additional information on system configuration and usage.

With structured guidelines and comprehensive file processing, the CastShop Knowledge Base stands as an essential tool for contributors working on the CastShop project, facilitating a seamless experience in exploring and understanding the repository data.
```
