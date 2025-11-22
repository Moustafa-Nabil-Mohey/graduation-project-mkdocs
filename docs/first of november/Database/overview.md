## Purpose

* This project parses ARM CMSIS SVD files and reference-manual PDFs for an STM32 microcontroller family, extracts peripherals, registers and fields and stores a flattened, query-friendly representation in MongoDB. The dataset quickly look up register addresses, field bit offsets/widths, and textual descriptions for use in tooling, documentation generation, or verification.

## Target database
* MongoDB Atlas . The code uses pymongo and expects a standard connection URI. The main collection stores one document per register-field pair (or one document per register if no fields are present).
