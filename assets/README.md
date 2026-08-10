# Cloudborg SVG Technology Asset Pass

Four small, original SVG technology illustrations are included:

- aws.svg
- gcp.svg
- ai.svg
- kubernetes.svg

These are intentionally minimalist and are not pixel-for-pixel reproductions
of third-party brand artwork. They are designed to fit the Cloudborg
light-purple visual system.

Recommended location in the Hugo project:

assets/icons/

The accompanying svg-tech-assets.css is the CSS layer for these assets.

Important:
The current homepage markup uses text/glyph technology identifiers. The
cleanest implementation is to replace those identifiers with img elements
pointing at these SVG assets rather than continuing to simulate marks with CSS.
