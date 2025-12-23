# GDS Layer Mapping Rules for MAGICAL Anaroute

This document explains the mapping rules used in the `CirDB::visualize()` function to generate GDS layer numbers from the internal circuit database layers. The mapping helps visualize the routed layout by offsetting layer indices to avoid conflicts in the GDS file.

## Overview

The `visualize()` function exports the circuit's pins, blocks, and access points to a GDS file for debugging and visualization. It applies offsets to the original LEF layer indices to create unique GDS layer numbers:

- **Pins (Pins)**: Shapes on routing layers, offset by net index.
- **Blocks (Blks)**: Blockages on routing layers, offset by a fixed value.
- **Access Points (AcsPt)**: Routing direction indicators, using fixed layer numbers.

Original layers are defined in `mock.techfile` (e.g., OD=6, PO=17, M1=31, etc.), and LEF layer indices map to these via `layerIdx2MaskIdx()`.

## LEF Layer Index Structure

Based on the LEF file (`mock.lef`) and code analysis, layers are ordered as follows (approximate for this mock PDK):

- **Masterslice Layers** (for poly):
  - 0: PO (Poly-Silicon, number=17)

- **Cut Layers** (for contacts/vias):
  - 1: CO (Contact, number=30)

- **Routing and Cut Layers** (alternating):
  - 2: M1 (Metal 1, number=31)
  - 3: VIA1 (Via 1, number=51)
  - 4: M2 (Metal 2, number=32)
  - 5: VIA2 (Via 2, number=52)
  - 6: M3 (Metal 3, number=33)
  - 7: VIA3 (Via 3, number=53)
  - 8: M4 (Metal 4, number=34)
  - 9: VIA4 (Via 4, number=54)
  - 10: M5 (Metal 5, number=35)
  - 11: VIA5 (Via 5, number=55)
  - 12: M6 (Metal 6, number=36)
  - 13: VIA6 (Via 6, number=56)
  - 14: M7 (Metal 7, number=37)
  - ... (higher layers follow the same pattern)

Routing layers (metals) have even indices starting from 2, cut layers (vias) have odd indices.

## Mapping Formulas

### For Pins
- **GDS Layer Number** = `layerIdx` + 100 * (`netIdx` + 1)
- `layerIdx`: LEF layer index (e.g., 2 for M1).
- `netIdx`: Net index (0-based, from 0 to numNets-1).
- **Example**: For a pin on layerIdx=2 (M1), netIdx=0: GDS layer = 2 + 100*1 = 102.
- All pins in this circuit are on layerIdx=2 (M1), so GDS layers are 102, 202, 302, ..., up to 2602 (for netIdx=0 to 25).

### For Blocks (Blockages)
- **GDS Layer Number** = `blkLayerIdx` + 10000
- `blkLayerIdx`: LEF layer index for the blockage (e.g., 2 for M1 blockages).
- **Example**: Blockage on layerIdx=2 (M1): GDS layer = 2 + 10000 = 10002.
- In this circuit, blockages are on layerIdx=2,4,6,8,10,12 (M1 to M6), so GDS layers are 10002, 10004, ..., 10012.

### For Access Points (AcsPt)
- Fixed GDS layers:
  - 20000: Center points of access points.
  - 30000: Direction indicators (crosses showing routing directions like NORTH, SOUTH, etc.).
- These do not correspond to original layers.

## Reverse Mapping (From GDS to Original Layers)

To find the original layer from a GDS layer number:

1. **Identify Type**:
   - If GDS layer >= 10000: Blockage.
   - Else if GDS layer == 20000 or 30000: Access point (no original layer).
   - Else: Pin.

2. **For Pins**:
   - `netIdx` = (GDS layer / 100) - 1
   - `layerIdx` = GDS layer % 100
   - Original layer number = `layerIdx2MaskIdx(layerIdx)` (from techfile, e.g., layerIdx=2 → 31 for M1).

3. **For Blockages**:
   - `blkLayerIdx` = GDS layer - 10000
   - Original layer number = `layerIdx2MaskIdx(blkLayerIdx)`.

### Example Reverse Mapping
- GDS layer 102 (Pin): netIdx = (102/100)-1 = 0, layerIdx = 102%100 = 2 → Original: M1 (31)
- GDS layer 10004 (Blk): blkLayerIdx = 10004 - 10000 = 4 → Original: M2 (32)
- GDS layer 20000: Access point center.

## Notes
- **Circuit-Specific**: This mapping is based on the mock PDK and the specific circuit. LEF layer order may vary in other designs.
- **No Conflicts**: Offsets ensure GDS layers don't overlap (e.g., pins start from 100+, blocks from 10000+).
- **Visualization Purpose**: The GDS file is for debugging; original layers are preserved in the database.
- **Techfile Reference**: See `mock.techfile` for layer names and numbers.
- **Code Reference**: See `CirDB::visualize()` in `src/db/dbCir.cpp` for implementation.

If the mapping seems incorrect for your design, check the LEF file layer order or run `CirDB::printInfo()` to verify `_tech.mStr2LayerMaskIdx()`.