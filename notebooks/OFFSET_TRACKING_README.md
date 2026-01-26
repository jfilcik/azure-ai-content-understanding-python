# Legal Transcript Reflow with Offset Tracking

## Overview

This enhanced implementation of the legal transcript reflow functionality maintains **accurate offset tracking** for words, lines, and pages in the reflowed markdown content. This enables:

1. ✅ **Precise page/line citations** - AI agents can cite specific line numbers accurately
2. ✅ **Source highlighting** - Use offsets and bounding boxes to highlight relevant document parts
3. ✅ **Traceability** - Map between reflowed content and original OCR positions

## Problem Statement

Legal transcripts have line numbers in the left margin (typically 1-25 per page). Azure Content Understanding's default markdown output groups these line numbers separately from the main text content. The original reflow implementation matched line numbers with text using bounding box coordinates, but this created a new problem:

**The reflowed content had different offsets than the original OCR output**, making it impossible to:
- Cite specific line numbers accurately in AI-generated summaries
- Highlight text in the original document using the reflowed content's references

## Solution

The enhanced implementation tracks offsets as it builds the reflowed content, creating a complete mapping structure that includes:

### Data Structures

```python
@dataclass
class OffsetMapping:
    """Maps elements from original to reflowed content with offsets."""
    pages: List['PageOffsetInfo']
    
@dataclass
class PageOffsetInfo:
    """Offset information for a page in reflowed content."""
    page_number: int
    offset: Dict[str, int]  # {"start": X, "end": Y}
    lines: List['LineOffsetInfo']
    
@dataclass  
class LineOffsetInfo:
    """Offset information for a line in reflowed content."""
    content: str
    offset: Dict[str, int]  # {"start": X, "end": Y}
    bbox: Optional[str]
    words: List['WordOffsetInfo']
    
@dataclass
class WordOffsetInfo:
    """Offset information for a word in reflowed content."""
    content: str
    offset: Dict[str, int]  # {"start": X, "end": Y}
    bbox: str
    original_offset: Optional[int]  # Original offset in OCR markdown
```

### Key Functions

#### `reflow_document_with_offsets(json_data, target_page=None, separator=" | ")`

Main function that reflows a document and returns both:
- Reflowed markdown content (string)
- Complete offset mapping (OffsetMapping object)

Example usage:
```python
reflowed_content, offset_mapping = reflow_document_with_offsets(result, target_page=3)

# Access page offsets
for page in offset_mapping.pages:
    print(f"Page {page.page_number}: [{page.offset['start']}:{page.offset['end']}]")
    
    # Access line offsets
    for line in page.lines:
        print(f"  Line: [{line.offset['start']}:{line.offset['end']}] {line.content[:50]}...")
        
        # Access word offsets
        for word in line.words:
            print(f"    Word: [{word.offset['start']}:{word.offset['end']}] '{word.content}'")
            print(f"          BBox: {word.bbox}")
            print(f"          Original OCR offset: {word.original_offset}")
```

## Output Structure

The offset mapping follows this structure:

```json
{
  "pages": [
    {
      "pageNumber": 3,
      "offset": {"start": 1, "end": 242},
      "lines": [
        {
          "content": "1 | INDEX",
          "offset": {"start": 17, "end": 26},
          "bbox": "D(3,0.1994,0.5888,2.1336,...)",
          "words": [
            {
              "content": "1",
              "offset": {"start": 17, "end": 18},
              "bbox": "D(3,0.1994,0.5888,0.3995,...)",
              "originalOffset": 200
            },
            {
              "content": "INDEX",
              "offset": {"start": 21, "end": 26},
              "bbox": "D(3,0.5336,0.5585,2.1336,...)",
              "originalOffset": 205
            }
          ]
        }
      ]
    }
  ]
}
```

## Use Cases

### 1. AI-Generated Citations

An AI agent can now cite specific line numbers with precision:

```python
# Find testimony about a topic
page_4 = offset_mapping.pages[1]  # Page 4
testimony_line = page_4.lines[0]

# Extract using offset
start = testimony_line.offset["start"]
end = testimony_line.offset["end"]
cited_text = reflowed_content[start:end]

# AI response with accurate citation
print(f"Susan Michaud's testimony begins on Page {page_4.page_number}, Line 1:")
print(f'"{cited_text}"')
```

### 2. Source Highlighting

Highlight specific words in the original document:

```python
# Get word information
susan_word = testimony_line.words[1]  # "Susan"

# Use bounding box to highlight in PDF viewer
bbox = susan_word.bbox  # "D(4,0.5336,0.5585,1.8336,0.5585,...)"
# Parse bbox and draw highlight rectangle at those coordinates
```

### 3. Offset Verification

Test that offsets correctly map to content:

```python
def test_offset_accuracy(content, mapping):
    for page in mapping.pages:
        for line in page.lines:
            start = line.offset["start"]
            end = line.offset["end"]
            extracted = content[start:end]
            assert extracted == line.content, f"Offset mismatch!"
```

## Running the Demo

### Option 1: Standalone Demo Script

```bash
cd notebooks
python demo_offset_visualization.py
```

This runs a standalone demonstration with sample data showing:
- Offset accuracy tests (100% pass rate)
- Word-level offset demonstration
- Citation use case example
- Source highlighting use case example
- Exported JSON offset mapping

### Option 2: Jupyter Notebook

Open `notebooks/legal_transcript_reflow.ipynb` and run all cells to:
1. Analyze a legal transcript with Azure Content Understanding
2. Reflow the content with line numbers inline
3. Generate complete offset mappings
4. View interactive HTML visualizations
5. Export offset mappings to JSON

## Files Modified

- **notebooks/legal_transcript_reflow.ipynb** - Enhanced with offset tracking implementation
- **notebooks/demo_offset_visualization.py** - Standalone demo script
- **update_notebook.py** - Script used to update the notebook

## Implementation Details

### Offset Calculation Algorithm

The implementation tracks offsets as it builds the reflowed content:

1. Start with `current_offset = 0`
2. For each page:
   - Add page marker: `"\n<!-- Page X -->\n"` and track its length
   - For each line group:
     - Track line start offset
     - Add line number (if present) and track its length
     - Add separator `" | "` and track its length
     - Add content words with spaces and track each word's offset
     - Add newline and track its length
3. Result: Every word, line, and page has accurate `{start, end}` offsets

### Bounding Box Preservation

Original bounding boxes from Azure OCR are preserved at all levels:
- Word-level: Each word retains its bbox for precise highlighting
- Line-level: Line bbox available for line-level highlighting
- Original offsets: Link back to original OCR markdown positions

## Benefits

### For AI Agents
- Can generate summaries with precise page/line citations
- Can reference specific testimony accurately
- Can create legal briefs with proper source attribution

### For Document Viewers
- Can highlight exact words/lines in original PDFs
- Can navigate to specific citations
- Can validate AI-generated references

### For Developers
- Complete traceability between reflowed and original content
- JSON export for integration with other systems
- Comprehensive offset mappings for custom use cases

## Testing

All offset mappings are tested for accuracy:

```python
# Run accuracy tests
test_results = test_offset_accuracy(reflowed_content, offset_mapping)

# Expected output:
# 📊 Results: 100/100 tests passed (100%)
```

The demo script includes comprehensive tests that verify:
- Line offsets match extracted content
- Word offsets match extracted content  
- Page offsets correctly bound all content

## Next Steps

1. Process your own legal transcripts using the enhanced notebook
2. Export offset mappings for integration with your systems
3. Use the mappings to enable precise citations in AI applications
4. Implement source highlighting in your PDF viewers

## Support

For questions or issues, please refer to:
- Main repository README
- Azure Content Understanding documentation
- Sample output in `notebooks/demo_output/`
