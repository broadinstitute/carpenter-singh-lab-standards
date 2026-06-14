# Infrastructure & Computing

> [!NOTE]
> Some links in this document require Broad Institute access. These are marked with 🔒.

## Computing Infrastructure

### Overview

- **Macbook**: For most daily tasks
- **DGX-1 / spirit / oppy servers**: For GPU computing FIXME: Now DGX-1 owned by C-lab
- **AWS**: For specific project needs (use judiciously)
- **Broad's compute cluster**: For easily parallelizable jobs

### DGX-1 Server (GPU Computing) FIXME: Now DGX-1 owned by C-lab, move this to deprecated

- Maintenance details: [https://github.com/broadinstitute/ip-chores/issues/25](https://github.com/broadinstitute/ip-chores/issues/25) 🔒
- Quick start guide: [https://new.ipwiki.app/dgx-1](https://new.ipwiki.app/dgx-1) 🔒

### Spirit/Oppy Server (GPU Computing)

- We are piloting nixOS [http://github.com/leoank/neusis/](http://github.com/leoank/neusis/)

### Broad's Compute Cluster

- Best for: Easily parallelizable jobs without interactivity or cloud data access needs
- Getting started: [https://intranet.broadinstitute.org/node/5811](https://intranet.broadinstitute.org/node/5811) 🔒
- Support: #bits Slack channel
- Also see #uger and #devnull-gpu-discuss Slack channels for discussions

### AWS

- Use cases: Projects requiring interactivity, large-scale cloud data access, or specific GPU needs
- Getting started:
  - Request an account through [CloudLab](https://cloud.nih.gov/resources/cloudlab/) for $500 in free credits with a 90-day limit, which will allow you to experiment freely
  - After initial experimentation, ask Shantanu to set up your project account
- Required reading:
  - Beth's [talk](https://drive.google.com/drive/u/0/folders/1EGnz4o4kBnHtMDjoYccuIyp-FVD-xer6) 🔒 and [slides](https://docs.google.com/presentation/d/1znPS90s9Yw22FrcsRn_Sd2tC_kaXELF48oEC7CoGXw8/edit?usp=sharing) 🔒 on Imaging Platform AWS Use. This is mostly relevant to Cimini Lab's usage of AWS but the concepts are useful to know.
  - [AWS Usage Policy](https://new.ipwiki.app/aws_amazon_web_services) 🔒
- We recommend interacting via [Skypilot](https://docs.skypilot.co/en/latest/docs/index.html); see this [Slack thread](https://broadinstitute.slack.com/archives/C3QFDHXC4/p1714062848412759) 🔒

## Development Environment Setup

### Package Management

- Install applications using [brew](https://brew.sh/)
- Use [uv](https://github.com/astral-sh/uv) for catalog work and standalone Python tooling - the default for data analysis (see [catalog-skills](https://github.com/carpenter-singh-lab/catalog-skills))
- Use [pixi](https://pixi.sh/) for production pipelines with conda/GPU dependencies (RAPIDS, CuPy) - see the catalog-skills README, "Graduating to a production pipeline"

## Project Setup & Initialization

> [!IMPORTANT]
> Complete these steps before starting any new project. Most links below require Broad access 🔒.

### Project Initialization

You will rarely need to initialize a project yourself, but here are the instructions if needed:

#### For Non-Cimini Lab Projects

1. [Create Imaging Platform Project List Item](https://new.ipwiki.app/image_analyst_onboarding_and_project_tracking#imaging_platform_project_list) 🔒
2. [Send startup email](https://new.ipwiki.app/image_analyst_onboarding_and_project_tracking#send_startup_email) 🔒 - See this [blog post](https://carpenter-singh-lab.broadinstitute.org/blog/tracking-projects-with-gmail-tags-collaborating-through-email) for context
3. Continue with "All Projects" steps below

#### For Cimini Lab Projects

- Find `project_tag` and `project_name_full` in [Imaging Platform Project List](https://airtable.com/appctUGldmRNkVS19) 🔒
- CC img-emailcatcher in email correspondence
- Continue with "All Projects" steps below

#### All Projects

1. Create GitHub repo following [wiki guidelines](https://new.ipwiki.app/github#github_policy) 🔒 for lab notebook
2. Create folder in [ProjectPlanning](https://drive.google.com/drive/u/0/folders/0B9yZOJkJh5aVMlN5SHhHa0J2VTg) 🔒: `YYYY_MM_DD_Project_description`
3. Create S3 folder: `s3://imaging-platform/YYYY_MM_DD_Project_description`

### Project Naming Convention

- Format: `YYYY_MM_DD_Short_description`
- Example: `2021_08_23_Batch_effect_correction`
