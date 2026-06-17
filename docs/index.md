---
icon: lucide/home
hide:
  - navigation
---

<div align="center" markdown>

<h1 style="font-size: 2.5rem; font-weight: 900; margin-bottom: 1rem;">ResXR</h1>

<p style="font-size: 1rem; max-width: 750px; margin: 0 auto 2rem auto; line-height: 1.6;">
  ResXR is a toolkit for XR behavioral research with two independent components connected only by a data contract — the Unity component writes CSV data and JSON metadata, and the Python component reads them. Enjoy complete independence: with no shared dependencies or complex build systems, you have the flexibility to use them seamlessly together or integrate each component individually into your own workflow.
</p>

---

<div style="display: flex; flex-wrap: wrap; justify-content: center; gap: 2rem; margin-top: 3rem; text-align: center;" markdown="1">

<div style="flex: 1; min-width: 300px;" markdown="1">

<h2 style="font-size: 1.6rem; font-weight: bold; margin-bottom: 0.5rem; margin-top: 0;">Python Pipeline</h2>

<p style="font-size: 0.9rem; max-width: 400px; margin: 0 auto; color: var(--md-default-fg-color--light);" markdown="1">
A Python (≥3.10) pipeline that converts the recorded data into [Motion-BIDS](https://bids.neuroimaging.io/) datasets.
</p>

<br>

[Read Documentation](python/index.md){ .md-button .md-button--primary style="font-size: 0.95rem; padding: 0.4rem 0.8rem; margin: 0.2rem;" }
[GitHub Repository](https://github.com/ResXR/resxr-python-pipeline){ .md-button style="font-size: 0.95rem; padding: 0.4rem 0.8rem; margin: 0.2rem;" }

</div>

<div style="flex: 1; min-width: 300px;" markdown="1">

<h2 style="font-size: 1.6rem; font-weight: bold; margin-bottom: 0.5rem; margin-top: 0;">Unity Research Template</h2>

<p style="font-size: 0.9rem; max-width: 400px; margin: 0 auto; color: var(--md-default-fg-color--light);" markdown="1">
A Unity 6 experiment template for Meta Quest standalone headsets that records sensor and behavioral data.
</p>

<br>

[Read Documentation](unity/index.md){ .md-button .md-button--primary style="font-size: 0.95rem; padding: 0.4rem 0.8rem; margin: 0.2rem;" }
[GitHub Repository](https://github.com/ResXR/resxr-unity-research-template){ .md-button style="font-size: 0.95rem; padding: 0.4rem 0.8rem; margin: 0.2rem;" }

</div>

</div>
