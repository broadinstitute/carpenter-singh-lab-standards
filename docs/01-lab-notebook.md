# Lab Notebook Guidelines

## Purpose

These guidelines establish our lab's standard practices for using GitHub repositories as lab notebooks.
Following these practices ensures consistency, reproducibility, and efficient collaboration across all lab projects.

## Repository Structure

- README.md: Project overview and key links
- Analysis modules: Folders named `00.<name>`, `01.<name>`, etc.
  - Each module contains:
    - Notebooks/scripts: `00.<name>.ipynb`, `01.<name>.py`, etc.
    - Subfolders: `input/`, `output/`, `figures/`

## Code Standards

- Use for most analyses (rmd, jupytext, ipynb)
- Use scripts for computationally intensive tasks
- Format code with ruff
- Follow [Google's Python Style Guide](https://google.github.io/styleguide/pyguide.html)

## Data Management

- Use DVC for data versioning
- For large/dynamic datasets:
  1. Create separate `-data` repository with the same base name as the main project
  2. Set up DVC in data repo:
     - Clone the `-data` repository and navigate to it
     - Install DVC: `pip install dvc`
     - Initialize DVC: `dvc init`
     - Add remote storage: `dvc remote add -d storage s3://<YOUR S3 URI>`
     - Commit and push changes
  3. Manage data:
     - Create a folder for tracked data (e.g., "data")
     - Add data to DVC: `dvc add ./data`
     - Commit and push data.dvc and .gitignore files
  4. Update data:
     - When adding or updating data, use:

       ```sh
       dvc add ./data
       git commit data.dvc -m "dataset updates"
       git push
       dvc push
       ```

  5. Add data repo as submodule to main project:
     - In the main project repo:

       ```sh
       git submodule add <data submodule git repo URL>
       git submodule update --init --recursive
       git commit -m "added submodule"
       git push
       ```

  6. Clean up: Delete the local copy of the data submodule repository created in step 2 to avoid confusion

## Environment Management

- Use Conda: one environment per analysis module

## Figure Generation

- Save as SVG for final versions
- Use `x.generate-figures.ipynb` in each module to
  - Read data from the output folder
  - Produce figures
  - Allow figure reproduction without redoing all analysis

## Experiment notes and discussions

Issues:

- Primary method for notes and discussions
- Use deliberate titles (hypothesis/question initially, conclusion when completed)
- Paste figures and analysis snippets for discussion

Pull Requests:

- Use for implementation-specific discussions
- Keep PRs small and focused
- Use fork-and-branch approach

## Best Practices

- Use .gitignore and precommit hooks
- No large files in main repository

## Addressing Pitfalls

This section is for known issues we're working to improve.
We welcome ideas for solutions.

- Analysis Iterations:
  - Consider adding optional datetime suffixes to folders/files for multiple iterations of an analysis
  - Example: `00.initial_analysis_2023-06-24`, `00.revised_analysis_2023-07-15`
- Project Management:
  - While using GitHub Issues, consider:
    1. Automating conversion of issues to SQLite files for improved searchability
    2. Exploring markdown-based project management tools (e.g., [HedgeDoc](https://hedgedoc.org/))
    3. Investigating RAG (Retrieval-Augmented Generation) tools like [Greptile](https://app.greptile.com/) for better information retrieval
