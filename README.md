# BDA: Google Maps Restaurant Reviews Pipeline

Integrated pipeline for large-scale extraction, processing, and analysis of Google Maps restaurant data. This repository serves as the central orchestrator for venue discovery and review scraping.

## 📁 Repository Structure

This project is organized using Git Submodules to maintain modularity between the scraping engines:

*   **[mapScraper](mapScraper/)**: Engine for massive venue metadata extraction and geographic filtering.
*   **[reviewsScraper](reviewsScraper/)**: Parallelized engine for high-volume review extraction using Playwright.
*   **[notebooks/](notebooks/)**: Data processing pipelines, NLP analysis, and Medallion architecture implementation.
*   **[docs/](docs/)**: Technical documentation and the final project report (`bda_reporte.pdf`).

## 🚀 Getting Started

To clone the repository along with all its submodules:

```bash
git clone --recursive https://github.com/fabrizziomcl/bda_gmaps_restreviews.git
cd bda_gmaps_restreviews
```

If you have already cloned the repository without submodules, run:

```bash
git submodule update --init --recursive
```

## 🛠 Pipeline Workflow

1.  **Venue Discovery**: Use `mapScraper` to generate a database of target restaurants based on geographic coordinates and search queries.
2.  **Review Extraction**: Use the output from step 1 as input for `reviewsScraper` to perform parallelized scraping of historical reviews.
3.  **Data Processing**: Execute `notebooks/Medallon.ipynb` to transform raw data through Bronze, Silver, and Gold layers using Spark.
4.  **Analysis**: Explore `notebooks/BDA_entrega_parcial.ipynb` for descriptive statistics, NLP sentiment insights, and geographic visualizations.

## 📄 Documentation

For a detailed explanation of the architecture, methodology, and results, please refer to:
👉 **[docs/bda_reporte.pdf](docs/bda_reporte.pdf)**
