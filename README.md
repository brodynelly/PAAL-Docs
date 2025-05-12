# PAAL Documentation

Welcome to the PAAL  Documentation site! This repository contains the source files for the documentation of the PAAL system, which provides comprehensive information about the system's architecture, components, and implementation details.

## Overview

The documentation is built using [MkDocs](https://www.mkdocs.org/) with the Material theme. It is designed to help developers, contributors, and stakeholders understand the PAAL system and its components.



## Prerequisites

Before you begin, ensure you have the following installed on your system:

1. **Python 3.8 or higher**  
   [Download Python](https://www.python.org/downloads/)

2. **pip** (Python package manager)  
   Comes pre-installed with Python. Verify by running:
   ```bash
   pip --version
   ```

3. **MkDocs**  
   Install MkDocs using pip:
   ```bash
   pip install mkdocs
   ```

4. **MkDocs Material Theme**  
   Install the Material theme for MkDocs:
   ```bash
   pip install mkdocs-material
   ```

5. **Additional MkDocs Plugins**  
   Install the required plugins:
   ```bash
   pip install mkdocs-git-revision-date-plugin mkdocs-awesome-pages-plugin
   ```

---

## Installation Steps

Follow these steps to set up the documentation site:

1. **Clone the Repository**  
   Clone the PAAL Documentation repository to your local machine:
   ```bash
   git clone https://github.com/brodynelly/PAAL-Docs.git
   cd PAAL-Docs
   ```

2. **Serve the Documentation Locally**  
   Start a local development server to preview the documentation:
   ```bash
   mkdocs serve
   ```
   This will start a server at `http://127.0.0.1:8000`. Open this URL in your browser to view the site.

---

## Deployment

To deploy the documentation site to GitHub Pages:

1. **Build the Site**  
   Generate the static files for the documentation:
   ```bash
   mkdocs build
   ```

2. **Deploy to GitHub Pages**  
   Use the following command to deploy:
   ```bash
   mkdocs gh-deploy
   ```

   This will push the built site to the `gh-pages` branch of your repository.

---


## Additional Resources

- [MkDocs Documentation](https://www.mkdocs.org/)
- [MkDocs Material Theme](https://squidfunk.github.io/mkdocs-material/)
- [GitHub Pages Deployment](https://www.mkdocs.org/user-guide/deploying-your-docs/)

---

