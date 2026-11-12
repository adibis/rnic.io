# rnic.io

Source for [rnic.io](https://rnic.io), a hardware-level guide to RDMA NIC architecture, block by
block: queue pairs, memory registration, the transmit and receive pipelines, completion queues,
reliability, congestion control, DMA and the IOMMU. Written by
[Aditya Shevade](https://chipdv.io/about).

## Building locally

Requires [Hugo](https://gohugo.io) extended (0.169.0+) and the theme submodule:

```
git submodule update --init --recursive
hugo server
```

## Deployment

Pushes to `master` build and deploy to GitHub Pages via `.github/workflows/deploy.yml`.

## License

The text and diagrams (`content/`, `static/images/`, `data/`) are licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) (see `LICENSE-CONTENT`): reuse them
freely, with credit to Aditya Shevade and a link to the original page. Everything else (Hugo
configuration, layouts, shortcodes, scripts) is MIT (see `LICENSE`). The Hextra theme in
`themes/hextra` keeps its own MIT license.

To cite the guide, see `CITATION.cff`, or use GitHub's "Cite this repository" button.
