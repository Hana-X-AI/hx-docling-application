# DocLayNet Dataset Analysis & Local Usage Guide

**Document Version**: 1.0.0
**Date**: December 15, 2025
**Dataset**: DocLayNet - Document Layout Segmentation
**Source**: IBM DocLayNet (ds4sd/DocLayNet)
**Deployment Mode**: Local (Non-Hugging Face)

---

## Executive Summary

DocLayNet is a comprehensive, human-annotated document layout segmentation dataset containing **80,863 pages** from diverse document sources. The complete dataset is deployed locally at `/home/agent0/hx-docling-application/project/0.5-test/test-data/` with two extracted archives:

1. **Core Dataset** (28 GB): PNG images (1025×1025px) + COCO format annotations
2. **Extra Files** (7.5 GB): PDF originals + JSON text cell data

This guide provides detailed instructions for integrating DocLayNet with the Docling MCP application for layout-aware document processing and testing.

---

## Dataset Overview

### Key Characteristics

| Property | Details |
|----------|---------|
| **Total Pages** | 80,863 unique document pages |
| **Document Categories** | 6 types (financial reports, scientific articles, laws, tenders, manuals, patents) |
| **Layout Classes** | 11 distinct layout element types |
| **Annotation Type** | Human-annotated bounding boxes (gold standard) |
| **Annotation Format** | COCO (Common Objects in Context) |
| **Image Format** | PNG (1025×1025px normalized) |
| **Document Format** | PDF originals + text cell JSON |
| **Train/Val/Test Split** | Pre-defined (69.4% / 6.5% / 4.1%) |
| **Redundant Annotations** | Subset includes double/triple annotations for uncertainty estimation |

### Document Categories

The dataset spans **6 document types**:

| Category | Pages | Examples |
|----------|-------|----------|
| **Financial Reports** | 22,629 | Annual reports, SEC filings, investor presentations |
| **Manuals** | 14,369 | User guides, technical documentation, instructions |
| **Scientific Articles** | 12,230 | Research papers, academic publications, journals |
| **Laws & Regulations** | 10,759 | Legal documents, regulatory text, legislation |
| **Patents** | 5,635 | Patent applications, technical specifications |
| **Government Tenders** | 3,753 | RFP documents, procurement notices, tender specs |

### Layout Element Classes (11 Types)

DocLayNet distinguishes **11 distinct layout elements**:

| ID | Class | Description | Primary Use |
|----|-------|-------------|-------------|
| 1 | **Caption** | Image/table captions and descriptions | Visual element context |
| 2 | **Footnote** | Footnotes, endnotes, references | Additional information |
| 3 | **Formula** | Mathematical equations and formulas | Scientific content |
| 4 | **List-item** | Bulleted/numbered list items | Structured content |
| 5 | **Page-footer** | Document footers, page numbers | Document structure |
| 6 | **Page-header** | Document headers, running titles | Document structure |
| 7 | **Picture** | Images, diagrams, charts | Visual content |
| 8 | **Section-header** | Section titles, headings | Content hierarchy |
| 9 | **Table** | Data tables, spreadsheet content | Structured data |
| 10 | **Text** | Body text, paragraphs | Main content |
| 11 | **Title** | Document titles, main headings | Document metadata |

---

## Dataset Structure & Layout

### Directory Organization

```
/home/agent0/hx-docling-application/project/0.5-test/test-data/
├── core-data/                          # Core dataset (extracted from DocLayNet_core.zip)
│   ├── COCO/                          # COCO format annotations
│   │   ├── train.json                 # 69,375 pages, 941,123 annotations
│   │   ├── val.json                   # 6,489 pages, 99,816 annotations
│   │   └── test.json                  # 4,999 pages, 66,531 annotations
│   └── PNG/                           # Document page images
│       ├── 000000c264503f54...png     # 1025×1025px PNG files
│       ├── 0003579dd4c3ad07...png     # ~80,863 total files
│       └── ...
│
└── extra-file-data/                    # Extra files (extracted from DocLayNet_extra.zip)
    ├── PDF/                           # Original PDF documents (single pages)
    │   ├── 000000c264503f54...pdf     # Page-extracted PDFs (1 page each)
    │   └── ...
    └── JSON/                          # Text cell data with coordinates
        ├── 000000c264503f54...json    # Page text cells with positions
        └── ...
```

### Data Split Statistics

#### Training Set (69,375 pages)

```
Images: 69,375
Annotations: 941,123 (13.6 annotations/page average)

Distribution by layout type:
  Text:             431,251 (45.8%) - Body text content
  Section-header:   118,590 (12.6%) - Section divisions
  List-item:        161,818 (17.2%) - Structured lists
  Page-header:       47,973 (5.1%)  - Document headers
  Page-footer:       61,313 (6.5%)  - Document footers
  Picture:           39,667 (4.2%)  - Images/diagrams
  Table:             30,070 (3.2%)  - Data tables
  Caption:           19,218 (2.0%)  - Image captions
  Formula:           21,167 (2.2%)  - Mathematical equations
  Footnote:           5,619 (0.6%)  - Reference notes
  Title:              4,437 (0.5%)  - Document titles
```

#### Validation Set (6,489 pages)

```
Images: 6,489
Annotations: 99,816 (15.4 annotations/page average)

Distribution by layout type:
  Text:             49,186 (49.3%)
  Section-header:   15,744 (15.8%)
  List-item:        13,320 (13.3%)
  Page-footer:       5,571 (5.6%)
  Page-header:       6,683 (6.7%)
  Picture:           2,775 (2.8%)
  Caption:           1,763 (1.8%)
  Formula:           1,894 (1.9%)
  Table:             2,269 (2.3%)
  Footnote:           312 (0.3%)
  Title:              299 (0.3%)
```

#### Test Set (4,999 pages)

```
Images: 4,999
Annotations: 66,531 (13.3 annotations/page average)

Distribution by layout type:
  Text:             29,940 (45.0%)
  Section-header:    8,550 (12.9%)
  List-item:        10,522 (15.8%)
  Page-footer:       3,994 (6.0%)
  Page-header:       3,366 (5.1%)
  Picture:           3,534 (5.3%)
  Caption:           1,543 (2.3%)
  Formula:           1,966 (3.0%)
  Table:             2,394 (3.6%)
  Footnote:           387 (0.6%)
  Title:              335 (0.5%)
```

---

## Data Format Details

### COCO Annotation Format

DocLayNet uses the standard COCO (Common Objects in Context) format with custom extensions for document metadata.

#### COCO JSON Structure

```json
{
  "info": {
    "description": "DocLayNet dataset",
    "version": "1.0.0"
  },
  "images": [
    {
      "id": 0,
      "width": 1025,
      "height": 1025,
      "file_name": "c6effb847ae7e4a80431696984fa90c98bb08c266481b9a03842422459c43bdd.png",
      
      // Custom document metadata fields:
      "doc_category": "financial_reports",  // One of 6 document types
      "collection": "ann_reports_00_04_fancy", // Source collection identifier
      "doc_name": "NYSE_F_2004.pdf",        // Original filename
      "page_no": 72,                        // Page number in original document
      "precedence": 0                       // Annotation order (0=primary, 1=secondary, 2=tertiary)
    }
  ],
  "annotations": [
    {
      "id": 0,
      "image_id": 0,                        // Reference to image
      "category_id": 6,                     // Layout class ID (1-11)
      "bbox": [
        72.35294285130719,                  // x (top-left)
        55.47565740740731,                  // y (top-left)
        372.2156845996732,                  // width
        20.452899744572278                  // height
      ],
      "segmentation": [
        [
          72.35294285130719, 55.47565740740731,   // Polygon vertices
          72.35294285130719, 75.92855715197959,   // (in clockwise order)
          444.5686274509804, 75.92855715197959,
          444.5686274509804, 55.47565740740731
        ]
      ],
      "area": 7612.890080474451,            // Bounding box area in pixels
      "iscrowd": 0,                         // 0=single object, 1=crowd
      "precedence": 0                       // Annotation order
    }
  ],
  "categories": [
    { "id": 1, "name": "Caption", "supercategory": "Caption" },
    { "id": 2, "name": "Footnote", "supercategory": "Footnote" },
    // ... (11 total categories)
  ]
}
```

#### Key Fields Explanation

| Field | Type | Description |
|-------|------|-------------|
| **image_id** | int | References image in `images` array |
| **category_id** | int | Layout class (1-11) |
| **bbox** | [x, y, w, h] | Bounding box: top-left corner + dimensions (pixels) |
| **segmentation** | [[x, y, ...]] | Polygon vertices (COCO format) |
| **area** | float | Bounding box area in pixels² |
| **doc_category** | string | Document type (6 categories) |
| **precedence** | int | Annotation order (0=primary, 1-2=secondary annotations) |

### PDF Document Format

Each page has a corresponding single-page PDF file with matching hash filename:

```
File naming: <SHA256_hash>.pdf
Example: 000000c264503f54eea3adfd8fabafe47248c76f7d688cb8f26b4d24876fccbe.pdf

Properties:
- Exactly 1 page per PDF
- Preserves original formatting and fonts
- Maintains spatial relationships with COCO coordinates
- Coordinate system: Original PDF dimensions (typically 595×842 for letter-size)
```

### JSON Text Cell Format

Each PDF page has corresponding JSON metadata with text cells and their coordinates.

#### JSON Cell Structure

```json
{
  "metadata": {
    "page_hash": "000000c264503f54eea3adfd8fabafe47248c76f7d688cb8f26b4d24876fccbe",
    "original_filename": "EN-Declaration on Honour.pdf",
    "page_no": 2,
    "num_pages": 6,
    "original_width": 595,
    "original_height": 842,
    "coco_width": 1025,
    "coco_height": 1025,
    "collection": "eu_tenders",
    "doc_category": "government_tenders"
  },
  "cells": [
    {
      "bbox": [
        93.30067495798319,    // x (original PDF coordinates)
        945.7663009204275,    // y (original PDF coordinates)
        74.2886011344538,     // width
        10.769413271971501    // height
      ],
      "text": "Page 3 of 6 ",     // Extracted text content
      "font": {
        "color": [0, 0, 0, 255],  // RGBA values
        "name": "/KQVYUV+ArialMT", // Font name
        "size": 8.64              // Font size (points)
      }
    }
    // ... more cells
  ]
}
```

#### Key Fields Explanation

| Field | Type | Description |
|-------|------|-------------|
| **page_hash** | string | SHA256 identifier matching PNG/PDF filenames |
| **original_filename** | string | Source document filename |
| **page_no** | int | Page number in original document (1-indexed) |
| **num_pages** | int | Total pages in original document |
| **original_width/height** | float | Original PDF page dimensions (points) |
| **coco_width/height** | int | Normalized PNG dimensions (1025×1025) |
| **cells.bbox** | [x, y, w, h] | Text cell bounding box (original PDF coords) |
| **cells.text** | string | Extracted text content |
| **cells.font.color** | [R, G, B, A] | RGBA color values |
| **cells.font.name** | string | Font name from PDF |
| **cells.font.size** | float | Font size in points |

---

## Coordinate Systems & Transformations

### Critical: Two Coordinate Systems

DocLayNet uses **two distinct coordinate systems** that must be carefully managed:

#### System 1: COCO Coordinates (PNG images)

```
- Origin: Top-left corner (0, 0)
- Range: 0 to 1025 (normalized square images)
- Reference: PNG file (1025×1025 pixels)
- Format: [x_topleft, y_topleft, width, height]
- Usage: Training layout detection models
```

#### System 2: Original PDF Coordinates

```
- Origin: Bottom-left corner (0, 0) [PDF standard]
- Range: 0 to original_width, 0 to original_height
- Reference: PDF metadata
- Format: [x, y, width, height]
- Usage: Original document analysis, text extraction
```

### Coordinate Transformation

To convert between systems:

```python
# COCO (normalized 1025×1025) → Original PDF coordinates
coco_bbox = [x_coco, y_coco, w_coco, h_coco]
orig_width = 595  # from metadata.original_width
orig_height = 842  # from metadata.original_height
coco_size = 1025

# Scale transformation
x_orig = (coco_bbox[0] / coco_size) * orig_width
y_orig = (coco_bbox[1] / coco_size) * orig_height  # Note: Y origin differs
w_orig = (coco_bbox[2] / coco_size) * orig_width
h_orig = (coco_bbox[3] / coco_size) * orig_height

# Flip Y-axis if needed for PDF coordinate system
y_orig_pdf = orig_height - y_orig
```

---

## File Locations & Sizes

### Local Deployment Locations

```
Dataset Paths:
├── PNG Images:          /home/agent0/hx-docling-application/project/0.5-test/test-data/core-data/PNG/
├── COCO Annotations:    /home/agent0/hx-docling-application/project/0.5-test/test-data/core-data/COCO/
├── PDF Documents:       /home/agent0/hx-docling-application/project/0.5-test/test-data/extra-file-data/PDF/
└── JSON Cell Data:      /home/agent0/hx-docling-application/project/0.5-test/test-data/extra-file-data/JSON/

Storage Requirements:
├── PNG Images:          ~30 GB (80,863 images × ~370 KB each)
├── COCO JSON files:     ~630 MB (3 files)
├── PDF Documents:       ~8.3 GB (80,863 PDFs × ~103 KB each)
├── JSON Cell Data:      ~3.0 GB (80,863 JSON files × ~37 KB each)
└── Total:              ~41.9 GB (after extraction)
```

---

## Python Integration Guide

### 1. Loading COCO Annotations

#### Basic Setup

```python
import json
from pathlib import Path

# Load COCO dataset
coco_path = Path("/home/agent0/hx-docling-application/project/0.5-test/test-data/core-data/COCO")

# Load train set
with open(coco_path / "train.json") as f:
    train_data = json.load(f)

# Access components
train_images = train_data["images"]          # List of 69,375 image records
train_annotations = train_data["annotations"] # List of 941,123 annotation records
categories = train_data["categories"]        # List of 11 layout classes
```

#### Iterate Through Images

```python
import json
from pathlib import Path
from PIL import Image

coco_path = Path("/home/agent0/hx-docling-application/project/0.5-test/test-data/core-data")

with open(coco_path / "COCO" / "train.json") as f:
    coco_data = json.load(f)

# Create mapping of image_id to image metadata
image_map = {img["id"]: img for img in coco_data["images"]}

# Create mapping of image_id to annotations
img_annotations = {}
for ann in coco_data["annotations"]:
    img_id = ann["image_id"]
    if img_id not in img_annotations:
        img_annotations[img_id] = []
    img_annotations[img_id].append(ann)

# Iterate through images
for image_id, image_info in image_map.items():
    # Load PNG image
    png_path = coco_path / "PNG" / image_info["file_name"]
    img = Image.open(png_path)
    
    # Get annotations for this image
    annotations = img_annotations.get(image_id, [])
    
    # Process document metadata
    doc_category = image_info["doc_category"]      # financial_reports, patents, etc.
    doc_name = image_info["doc_name"]              # Original PDF filename
    page_no = image_info["page_no"]                # Page number
    
    # Process each annotation
    for ann in annotations:
        bbox = ann["bbox"]                         # [x, y, width, height]
        layout_class_id = ann["category_id"]       # 1-11
        area = ann["area"]                         # Pixels²
        
        # Find category name
        category_name = next(
            (c["name"] for c in coco_data["categories"] if c["id"] == layout_class_id),
            None
        )
```

### 2. Loading PDF & JSON Cell Data

#### Access PDF and Text Cells

```python
import json
from pathlib import Path
from PyPDF2 import PdfReader

extra_path = Path("/home/agent0/hx-docling-application/project/0.5-test/test-data/extra-file-data")

# Example: Load data for a specific page
page_hash = "000000c264503f54eea3adfd8fabafe47248c76f7d688cb8f26b4d24876fccbe"

# Load PDF
pdf_path = extra_path / "PDF" / f"{page_hash}.pdf"
with open(pdf_path, "rb") as f:
    pdf_reader = PdfReader(f)
    # PDF has exactly 1 page
    page = pdf_reader.pages[0]
    text_content = page.extract_text()

# Load JSON cell data
json_path = extra_path / "JSON" / f"{page_hash}.json"
with open(json_path) as f:
    cell_data = json.load(f)

# Access metadata
metadata = cell_data["metadata"]
print(f"Document: {metadata['original_filename']}")
print(f"Page: {metadata['page_no']} of {metadata['num_pages']}")
print(f"Size: {metadata['original_width']}×{metadata['original_height']}")

# Access text cells
for cell in cell_data["cells"]:
    bbox = cell["bbox"]              # [x, y, width, height] in original coords
    text = cell["text"]              # Extracted text
    font_color = cell["font"]["color"]  # [R, G, B, A]
    font_name = cell["font"]["name"]    # Font name
    font_size = cell["font"]["size"]    # Size in points
```

### 3. Unified Document Loader

#### Comprehensive Integration Function

```python
import json
from pathlib import Path
from PIL import Image
from typing import Dict, List, Tuple

class DocLayNetLoader:
    """Load and access DocLayNet dataset locally"""
    
    def __init__(self, dataset_root: str = "/home/agent0/hx-docling-application/project/0.5-test/test-data"):
        self.root = Path(dataset_root)
        self.core_path = self.root / "core-data"
        self.extra_path = self.root / "extra-file-data"
        
        # Load COCO metadata for all splits
        self.train_coco = self._load_coco("train")
        self.val_coco = self._load_coco("val")
        self.test_coco = self._load_coco("test")
        
        # Create efficient lookup maps
        self._build_indices()
    
    def _load_coco(self, split: str) -> Dict:
        """Load COCO JSON for specified split"""
        coco_file = self.core_path / "COCO" / f"{split}.json"
        with open(coco_file) as f:
            return json.load(f)
    
    def _build_indices(self):
        """Build efficient lookup dictionaries"""
        self.categories = {
            c["id"]: c["name"] for c in self.train_coco["categories"]
        }
        
        self.image_map = {}
        self.annotations_by_image = {}
        
        for split_data in [self.train_coco, self.val_coco, self.test_coco]:
            for img in split_data["images"]:
                self.image_map[img["file_name"]] = img
            
            for ann in split_data["annotations"]:
                img_id = ann["image_id"]
                if img_id not in self.annotations_by_image:
                    self.annotations_by_image[img_id] = []
                self.annotations_by_image[img_id].append(ann)
    
    def load_page(self, page_hash: str) -> Dict:
        """Load complete page data from hash"""
        # Load PNG
        png_path = self.core_path / "PNG" / f"{page_hash}.png"
        image = Image.open(png_path)
        
        # Load PDF
        pdf_path = self.extra_path / "PDF" / f"{page_hash}.pdf"
        pdf_exists = pdf_path.exists()
        
        # Load JSON cells
        json_path = self.extra_path / "JSON" / f"{page_hash}.json"
        with open(json_path) as f:
            cell_data = json.load(f)
        
        return {
            "png": image,
            "pdf_path": str(pdf_path) if pdf_exists else None,
            "cells": cell_data["cells"],
            "metadata": cell_data["metadata"]
        }
    
    def get_annotations_for_image(self, page_hash: str) -> List[Dict]:
        """Get COCO annotations for a page"""
        # Find image_id from filename
        image_info = self.image_map.get(f"{page_hash}.png")
        if not image_info:
            return []
        
        annotations = self.annotations_by_image.get(image_info["id"], [])
        
        # Enrich with category names
        for ann in annotations:
            ann["category_name"] = self.categories[ann["category_id"]]
        
        return annotations
```

### 4. Statistics & Filtering

#### Dataset Analysis

```python
import json
from pathlib import Path
from collections import Counter

def analyze_dataset(dataset_root: str):
    """Generate dataset statistics"""
    root = Path(dataset_root)
    coco_path = root / "core-data/COCO"
    
    for split in ["train", "val", "test"]:
        with open(coco_path / f"{split}.json") as f:
            data = json.load(f)
        
        # Count images and annotations
        n_images = len(data["images"])
        n_annotations = len(data["annotations"])
        
        # Count by category
        category_counts = Counter()
        for ann in data["annotations"]:
            cat_id = ann["category_id"]
            cat_name = next(
                (c["name"] for c in data["categories"] if c["id"] == cat_id),
                "Unknown"
            )
            category_counts[cat_name] += 1
        
        # Count by doc type
        doc_type_counts = Counter()
        for img in data["images"]:
            doc_type_counts[img["doc_category"]] += 1
        
        print(f"\n=== {split.upper()} SET ===")
        print(f"Images: {n_images:,}")
        print(f"Annotations: {n_annotations:,}")
        print(f"Avg annotations/image: {n_annotations/n_images:.1f}")
        
        print(f"\nLayout classes:")
        for layout_class, count in category_counts.most_common():
            pct = 100 * count / n_annotations
            print(f"  {layout_class:20s}: {count:7,} ({pct:5.1f}%)")
        
        print(f"\nDocument categories:")
        for doc_type, count in doc_type_counts.most_common():
            pct = 100 * count / n_images
            print(f"  {doc_type:25s}: {count:6,} ({pct:5.1f}%)")

# Run analysis
analyze_dataset("/home/agent0/hx-docling-application/project/0.5-test/test-data")
```

### 5. Filtering for Specific Document Types

```python
import json
from pathlib import Path

def get_pages_by_category(dataset_root: str, doc_category: str, split: str = "train") -> List[str]:
    """Get all page hashes for a specific document category"""
    coco_path = Path(dataset_root) / "core-data/COCO"
    
    with open(coco_path / f"{split}.json") as f:
        data = json.load(f)
    
    # Filter images by doc_category
    matching_images = [
        img["file_name"].replace(".png", "")  # Return page hash
        for img in data["images"]
        if img["doc_category"] == doc_category
    ]
    
    return matching_images

# Example usage:
financial_pages = get_pages_by_category(
    "/home/agent0/hx-docling-application/project/0.5-test/test-data",
    doc_category="financial_reports",
    split="train"
)
print(f"Found {len(financial_pages)} financial report pages")

# Load first financial report page
loader = DocLayNetLoader()
page_hash = financial_pages[0]
page_data = loader.load_page(page_hash)
annotations = loader.get_annotations_for_image(page_hash)
```

---

## Integration with Docling MCP

### Use Case 1: Training Layout Detection Model

```python
from pathlib import Path
import torch
from torchvision import transforms
from PIL import Image

class DocLayNetDataset(torch.utils.data.Dataset):
    """PyTorch dataset for layout detection training"""
    
    def __init__(self, dataset_root: str, split: str = "train"):
        self.loader = DocLayNetLoader(dataset_root)
        self.split = split
        
        # Load appropriate COCO data
        if split == "train":
            coco = self.loader.train_coco
        elif split == "val":
            coco = self.loader.val_coco
        else:
            coco = self.loader.test_coco
        
        self.images = coco["images"]
        self.coco_data = coco
    
    def __len__(self):
        return len(self.images)
    
    def __getitem__(self, idx):
        image_info = self.images[idx]
        page_hash = image_info["file_name"].replace(".png", "")
        
        # Load image
        png_path = self.loader.core_path / "PNG" / image_info["file_name"]
        image = Image.open(png_path).convert("RGB")
        
        # Get annotations
        annotations = self.loader.annotations_by_image.get(image_info["id"], [])
        
        # Convert to format for object detection model
        boxes = []
        labels = []
        
        for ann in annotations:
            x, y, w, h = ann["bbox"]
            boxes.append([x, y, x+w, y+h])  # Convert to [x1, y1, x2, y2]
            labels.append(ann["category_id"])
        
        # Transform image
        transform = transforms.ToTensor()
        image = transform(image)
        
        return {
            "image": image,
            "boxes": torch.tensor(boxes, dtype=torch.float32),
            "labels": torch.tensor(labels, dtype=torch.int64),
            "image_id": image_info["id"],
            "doc_category": image_info["doc_category"]
        }
```

### Use Case 2: Document Analysis Pipeline

```python
from pathlib import Path
import json

def analyze_document_layout(page_hash: str, dataset_root: str) -> Dict:
    """Complete layout analysis for a document page"""
    
    loader = DocLayNetLoader(dataset_root)
    
    # Load all data
    page_data = loader.load_page(page_hash)
    annotations = loader.get_annotations_for_image(page_hash)
    
    metadata = page_data["metadata"]
    
    # Organize by layout type
    layout_analysis = {
        "document": metadata["original_filename"],
        "page": metadata["page_no"],
        "size": {
            "width": metadata["original_width"],
            "height": metadata["original_height"]
        },
        "elements_by_type": {}
    }
    
    # Group annotations by type
    for ann in annotations:
        layout_type = ann["category_name"]
        
        if layout_type not in layout_analysis["elements_by_type"]:
            layout_analysis["elements_by_type"][layout_type] = []
        
        layout_analysis["elements_by_type"][layout_type].append({
            "bbox": ann["bbox"],
            "area": ann["area"],
            "confidence": 1.0  # Annotations are gold standard
        })
    
    # Get text content
    text_analysis = {
        "cells": page_data["cells"],
        "total_text_cells": len(page_data["cells"])
    }
    
    return {
        "layout": layout_analysis,
        "text": text_analysis,
        "metadata": metadata
    }

# Example:
analysis = analyze_document_layout(
    "000000c264503f54eea3adfd8fabafe47248c76f7d688cb8f26b4d24876fccbe",
    "/home/agent0/hx-docling-application/project/0.5-test/test-data"
)
```

### Use Case 3: Data Sampling for Testing

```python
import random

def get_test_sample(dataset_root: str, n_pages: int = 10, doc_category: str = None) -> List[str]:
    """Get random sample of pages for testing"""
    
    coco_path = Path(dataset_root) / "core-data/COCO"
    
    with open(coco_path / "test.json") as f:
        test_data = json.load(f)
    
    # Filter by category if specified
    images = test_data["images"]
    if doc_category:
        images = [img for img in images if img["doc_category"] == doc_category]
    
    # Sample random pages
    sample = random.sample(images, min(n_pages, len(images)))
    
    return [img["file_name"].replace(".png", "") for img in sample]

# Example:
financial_test_pages = get_test_sample(
    "/home/agent0/hx-docling-application/project/0.5-test/test-data",
    n_pages=5,
    doc_category="financial_reports"
)

for page_hash in financial_test_pages:
    analysis = analyze_document_layout(page_hash, dataset_root)
    print(f"\n{analysis['metadata']['original_filename']} (page {analysis['metadata']['page_no']})")
    for layout_type, elements in analysis['layout']['elements_by_type'].items():
        print(f"  {layout_type}: {len(elements)}")
```

---

## Performance Optimization Tips

### 1. Efficient Data Loading

```python
# Use memory mapping for large COCO files
import json
import mmap

def load_coco_mmap(coco_file: str) -> Dict:
    """Load COCO JSON efficiently"""
    with open(coco_file, 'r+b') as f:
        with mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ) as mmapped:
            return json.loads(mmapped.read().decode())
```

### 2. Batch Processing

```python
from pathlib import Path
import concurrent.futures

def process_pages_batch(page_hashes: List[str], dataset_root: str, batch_size: int = 32):
    """Process multiple pages in parallel"""
    loader = DocLayNetLoader(dataset_root)
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=4) as executor:
        futures = []
        for page_hash in page_hashes:
            future = executor.submit(loader.load_page, page_hash)
            futures.append((page_hash, future))
        
        for page_hash, future in futures:
            try:
                page_data = future.result()
                yield page_hash, page_data
            except Exception as e:
                print(f"Error loading {page_hash}: {e}")
```

### 3. Caching Strategy

```python
from functools import lru_cache

class DocLayNetLoaderCached(DocLayNetLoader):
    """Add caching for frequently accessed data"""
    
    @lru_cache(maxsize=1000)
    def _load_cell_data(self, page_hash: str) -> Dict:
        """Cache JSON cell data"""
        json_path = self.extra_path / "JSON" / f"{page_hash}.json"
        with open(json_path) as f:
            return json.load(f)
    
    def load_page(self, page_hash: str) -> Dict:
        """Use cached cell data"""
        # Load PNG and PDF as before
        png_path = self.core_path / "PNG" / f"{page_hash}.png"
        image = Image.open(png_path)
        
        # Use cached JSON
        cell_data = self._load_cell_data(page_hash)
        
        return {
            "png": image,
            "cells": cell_data["cells"],
            "metadata": cell_data["metadata"]
        }
```

### 4. Subset Extraction for Development

```python
import shutil
from pathlib import Path

def create_development_subset(dataset_root: str, output_root: str, n_pages: int = 100):
    """Create smaller subset for faster development"""
    
    source_root = Path(dataset_root)
    output_root = Path(output_root)
    output_root.mkdir(parents=True, exist_ok=True)
    
    # Create directory structure
    (output_root / "core-data/COCO").mkdir(parents=True, exist_ok=True)
    (output_root / "core-data/PNG").mkdir(parents=True, exist_ok=True)
    (output_root / "extra-file-data/PDF").mkdir(parents=True, exist_ok=True)
    (output_root / "extra-file-data/JSON").mkdir(parents=True, exist_ok=True)
    
    # Get sample pages
    sample_pages = get_test_sample(dataset_root, n_pages=n_pages)
    
    # Copy PNG images
    for page_hash in sample_pages:
        src = source_root / "core-data/PNG" / f"{page_hash}.png"
        dst = output_root / "core-data/PNG" / f"{page_hash}.png"
        shutil.copy2(src, dst)
    
    # Copy PDF files
    for page_hash in sample_pages:
        src = source_root / "extra-file-data/PDF" / f"{page_hash}.pdf"
        dst = output_root / "extra-file-data/PDF" / f"{page_hash}.pdf"
        if src.exists():
            shutil.copy2(src, dst)
    
    # Copy JSON files
    for page_hash in sample_pages:
        src = source_root / "extra-file-data/JSON" / f"{page_hash}.json"
        dst = output_root / "extra-file-data/JSON" / f"{page_hash}.json"
        shutil.copy2(src, dst)
    
    # Create filtered COCO files
    # (This is more complex - see full implementation)
    
    print(f"Created subset with {n_pages} pages at {output_root}")
```

---

## Troubleshooting & Common Issues

### Issue 1: Missing Files

```python
# Verify file integrity
def verify_dataset_integrity(dataset_root: str) -> Dict[str, int]:
    """Check for missing or corrupted files"""
    
    root = Path(dataset_root)
    issues = {
        "missing_png": 0,
        "missing_pdf": 0,
        "missing_json": 0,
        "corrupted_json": 0
    }
    
    coco_path = root / "core-data/COCO/train.json"
    with open(coco_path) as f:
        coco = json.load(f)
    
    for img in coco["images"]:
        page_hash = img["file_name"].replace(".png", "")
        
        # Check PNG
        png_path = root / "core-data/PNG" / img["file_name"]
        if not png_path.exists():
            issues["missing_png"] += 1
        
        # Check PDF
        pdf_path = root / "extra-file-data/PDF" / f"{page_hash}.pdf"
        if not pdf_path.exists():
            issues["missing_pdf"] += 1
        
        # Check JSON
        json_path = root / "extra-file-data/JSON" / f"{page_hash}.json"
        if not json_path.exists():
            issues["missing_json"] += 1
        else:
            try:
                with open(json_path) as f:
                    json.load(f)
            except json.JSONDecodeError:
                issues["corrupted_json"] += 1
    
    return issues
```

### Issue 2: Memory Issues with Large Batch Processing

```python
# Use generators for memory efficiency
def batch_loader(dataset_root: str, batch_size: int = 32):
    """Memory-efficient batch loading"""
    
    loader = DocLayNetLoader(dataset_root)
    
    # Get all image filenames
    all_images = [img["file_name"] for img in loader.train_coco["images"]]
    
    for i in range(0, len(all_images), batch_size):
        batch_files = all_images[i:i+batch_size]
        batch_data = []
        
        for filename in batch_files:
            page_hash = filename.replace(".png", "")
            try:
                data = loader.load_page(page_hash)
                batch_data.append(data)
            except Exception as e:
                print(f"Error loading {page_hash}: {e}")
        
        yield batch_data
```

### Issue 3: Coordinate System Confusion

```python
# Helper to manage coordinate transformations
class CoordinateConverter:
    """Convert between COCO and original PDF coordinates"""
    
    def __init__(self, page_hash: str, dataset_root: str):
        loader = DocLayNetLoader(dataset_root)
        metadata = loader._load_cell_data(page_hash)["metadata"]
        
        self.coco_size = 1025
        self.orig_width = metadata["original_width"]
        self.orig_height = metadata["original_height"]
    
    def coco_to_original(self, bbox: List[float]) -> List[float]:
        """Convert COCO [x, y, w, h] to original PDF coordinates"""
        x, y, w, h = bbox
        
        # Scale coordinates
        x_orig = (x / self.coco_size) * self.orig_width
        y_orig = (y / self.coco_size) * self.orig_height
        w_orig = (w / self.coco_size) * self.orig_width
        h_orig = (h / self.coco_size) * self.orig_height
        
        return [x_orig, y_orig, w_orig, h_orig]
    
    def original_to_coco(self, bbox: List[float]) -> List[float]:
        """Convert original PDF coordinates to COCO"""
        x, y, w, h = bbox
        
        # Scale coordinates
        x_coco = (x / self.orig_width) * self.coco_size
        y_coco = (y / self.orig_height) * self.coco_size
        w_coco = (w / self.orig_width) * self.coco_size
        h_coco = (h / self.orig_height) * self.coco_size
        
        return [x_coco, y_coco, w_coco, h_coco]
```

---

## References & Additional Resources

### DocLayNet Official Resources

- **Repository**: https://github.com/DS4SD/DocLayNet
- **Paper**: "DocLayNet: A Large-Scale Document Layout Segmentation Dataset"
- **Labeling Guide**: [DocLayNet_Labeling_Guide_Public.pdf](./assets/DocLayNet_Labeling_Guide_Public.pdf)

### Related Datasets

- **PubLayNet**: Layout analysis for scientific papers
- **DocBank**: Layout analysis with NLP
- **RVL-CDIP**: Document image classification

### Layout Detection Models

- **Faster R-CNN**: Traditional object detection baseline
- **Mask R-CNN**: Instance segmentation variant
- **YOLO**: Real-time detection
- **Detectron2**: Facebook's detection framework

---

## Summary

DocLayNet provides a comprehensive, human-annotated dataset for document layout analysis with:

✅ **80,863 diverse document pages** from 6 categories
✅ **11 distinct layout element classes** with detailed annotations
✅ **Multiple data formats** (PNG + COCO + PDF + JSON)
✅ **Pre-split train/val/test sets** with proper distribution
✅ **Locally deployed** without Hugging Face dependency
✅ **Redundant annotations** for uncertainty estimation

This guide enables seamless integration with the Docling MCP application for layout-aware document processing, model training, and comprehensive testing.

---

**Document Information**
- Version: 1.0.0
- Last Updated: December 15, 2025
- Dataset Version: 1.0.0 (ds4sd/DocLayNet)
- Local Deployment: Complete
- Status: Ready for Development & Testing
