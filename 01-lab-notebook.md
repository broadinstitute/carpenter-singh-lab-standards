# Lab Notebook Guidelines

## Purpose

These guidelines establish our lab's standard practices for using GitHub repositories as lab notebooks.
Our past [discussions](https://github.com/carpenterlab/open-science-rules/issues/18) led to these guidelines.

## Repository Structure

- README.md: Project overview and key links
- Analysis modules: Folders named `00.<name>`, `01.<name>`, etc.
- Each module contains:
  - Notebooks/scripts: `00.<name>.ipynb`, `01.<name>.py`, etc.
  - Subfolders: `input/`, `output/`, `figures/`

## Experiment notes and discussions

Issues:

- Primary method for notes and discussions
- Use deliberate titles (hypothesis/question initially, conclusion when completed)
- Paste figures and analysis snippets for discussion

Pull Requests:

- Use for implementation-specific discussions
- Keep PRs small and focused
- Use fork-and-branch approach

## Code Standards

- Use for most analyses (rmd, jupytext, ipynb)
- Use scripts for computationally intensive tasks
- Format code with ruff (see [Best Practices](#best-practices))

## Data Management

Use DVC for data versioning.

For large/dynamic datasets:

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

            dvc add ./data
            git commit data.dvc -m "dataset updates"
            git push
            dvc push

5. Add data repo as submodule to main project:
      - In the main project repo:

            git submodule add <data submodule git repo URL>
            git submodule update --init --recursive
            git commit -m "added submodule"
            git push

6. Clean up: Delete the local copy of the data submodule repository created in step 2 to avoid confusion

## Environment Management

- Use Conda: one environment per analysis module

## Figure Generation

- Save as SVG for final versions
- Use `x.generate-figures.ipynb` in each module to
  - Read data from the output folder
  - Produce figures
  - Allow figure reproduction without redoing all analysis

## Best Practices

- Use `.gitignore`
- Use [precommit hooks](https://pre-commit.com/) (see webpage for local installation instructions)
   - Use [`.pre-commit-config.yaml`](.pre-commit-config.yaml) as the starting point
   - Run `pre-commit run --all-files` locally
   - Install precommit CI on your repo <https://pre-commit.ci/>
- Use [ruff](https://github.com/charliermarsh/ruff) to enforce code style
  At present, we use the default settings, so `.ruff.toml` is not required, but we have included and empty config for completeness (and `.pre-commit-config.yaml` requires it)

## Addressing Pitfalls

_This section is for known issues we're working to improve._
_We welcome ideas for solutions._

- Analysis Iterations:
  - Consider adding optional datetime suffixes to folders/files for multiple iterations of an analysis
  - Example: `00.initial_analysis_2023-06-24`, `00.revised_analysis_2023-07-15`
- Project Management:
  - While using GitHub Issues, consider:
    - Automating conversion of issues to SQLite files for improved searchability
    - Exploring markdown-based project management tools (e.g., [HedgeDoc](https://hedgedoc.org/))
    - Investigating RAG (Retrieval-Augmented Generation) tools like [Greptile](https://app.greptile.com/) for better information retrieval
