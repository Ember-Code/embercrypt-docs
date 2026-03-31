# Database Visualizer

Visualizes a binary database file as a PNG image. Each byte maps to one grayscale pixel, making entropy differences between plaintext and encrypted databases immediately visible.

## Usage
```
dbviz <input_file> <output.png>
```

Requires ImageMagick (`magick`) to be installed for PGM-to-PNG conversion.

## Output format

Image width is fixed at 4096 pixels (one page per row). Height is `ceil(file_size / 4096)`. The last row is zero-padded if the file does not end on a page boundary.

## Examples
```sh
# Visualize a plaintext database - structured, low-entropy regions visible
dbviz plain.db plain.png

# Visualize an encrypted database - uniform high-entropy noise expected
dbviz encrypted.db encrypted.png
```

A well-encrypted database should produce a uniform gray noise image with no visible structure. Patterns or gradients in the encrypted output may indicate issues with key material or IV reuse.

## Dependencies

- ImageMagick (`magick` must be on `PATH`)