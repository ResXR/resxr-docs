# ResXR Documentation

This repository contains the official documentation for the **ResXR Toolkit**, built using [Zensical](https://zensical.org) (a wrapper for Material for MkDocs).

## Viewing the Documentation Locally

To preview or edit the documentation on your own machine, follow these steps:

### 1. Prerequisites
Make sure you have [uv](https://docs.astral.sh/uv/getting-started/installation/) installed. `uv` is an extremely fast Python package manager that handles all the dependencies for this project automatically.

### 2. Setup
Clone this repository and open the folder in your terminal:
```bash
git clone https://github.com/ResXR/resxr-docs.git
cd resxr-docs
```

### 3. Run the Local Server
Run the following command to automatically install dependencies and start a live-reloading web server:
```bash
uv run zensical serve
```

Once the server starts, open your browser and go to `http://localhost:8000`. 

*(Tip: Any changes you make to the markdown files inside the `docs/` folder will instantly reload in your browser!)*