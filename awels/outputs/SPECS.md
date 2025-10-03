# Langflow Agent Specifications

## Overview
This document provides specifications for creating a Langflow agent that uses the knowledge bases created by the GitHub Repository Analyzer.

## Knowledge Bases
The agent should use the following knowledge bases:

1. **Code Knowledge Base**
   - Collection Name: `cast-shopizer-code`
   - Content: Source code files and analysis
   - Use for: Answering questions about code structure, functionality, and implementation details

2. **Documentation Knowledge Base**
   - Collection Name: `cast-shopizer-doc`
   - Content: Markdown documentation files
   - Use for: Answering questions about project documentation, guides, and explanations

3. **Image Knowledge Base**
   - Collection Name: `cast-shopizer-multi`
   - Content: Images and diagrams
   - Use for: Answering questions about visual content and diagrams

## Agent Configuration
1. Use the system prompt provided in `PROMPT.md`
2. Configure the agent with the following tools:
   - Qdrant vector database access to all three collections
   - Sequential thinking tool for complex reasoning
   - Code execution capability (optional)

## Deployment Instructions
1. Import the exported databases from the `outputs/databases` directory
2. Create a new Langflow agent with the system prompt
3. Configure the knowledge base tools with the appropriate collection names
4. Test the agent with sample questions about the repository
