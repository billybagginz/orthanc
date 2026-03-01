---
name: Extract Text from PDF
description: Automated skill for extracting text content from PDF files with support for both digital and scanned PDFs, including error handling and quality assurance.
---

# Extract Text from PDF

## Overview
This skill extracts text content from PDF documents, handling both digitally-created PDFs (with selectable text) and scanned PDFs (requiring OCR). Ideal for legal documents, contracts, and case files in a legal office environment.

**Last updated: 2026-02-28 17:45:00**

## Inputs Required

### Primary Input
- **PDF File Path** (required): Full path to the PDF file to be processed
  - Must be a valid PDF file format
  - File must exist and be readable
  - No size limit imposed, but performance degrades with files >500MB

### Optional Parameters
- **Target Format** (default: markdown): Output format (plain_text, markdown, json)
- **Language** (default: english): Hint for OCR processing if scanned PDF detected
- **Quality Threshold** (default: 0.7): Minimum confidence level (0-1) for OCR results to include
- **Include Metadata** (default: true): Extract document metadata (title, author, creation date, etc.)

## Outputs Provided

### Primary Output
- **Extracted Text** (string): The full text content extracted from the PDF
- **Format**: Structured by page with clear page breaks and section markers
- **Encoding**: UTF-8

### Metadata Output (if included)
```json
{
  "file_path": "path/to/document.pdf",
  "file_size_bytes": 1024000,
  "page_count": 15,
  "created_date": "2024-01-15T10:30:00Z",
  "modified_date": "2024-01-20T14:45:00Z",
  "title": "Contract Agreement",
  "author": "Legal Dept",
  "subject": "Service Agreement",
  "producer": "Adobe Acrobat",
  "extraction_method": "direct_text|ocr",
  "confidence_score": 0.95,
  "processing_time_seconds": 2.34,
  "warnings": ["page_5_low_quality", "corrupted_image_detected"]
}
```

### Detailed Output Structure
- **pages**: Array of page objects
  - `page_number`: Sequential page number
  - `text`: Extracted text for that page
  - `confidence`: OCR confidence (if applicable)
  - `lines`: Array of line objects with bounding box information
  - `images_count`: Number of images on page

## Step-by-Step Execution

### Step 1: Validate Input
1. Check that the provided file path is valid and file exists
2. Verify file is a PDF (check magic bytes: `%PDF`)
3. Check file is readable (not locked by another process)
4. If file is not accessible, return error with specific reason (not_found, permission_denied, locked, invalid_format)

### Step 2: Analyze PDF Type
1. Open the PDF without rendering to text
2. Check if PDF contains selectable text streams:
   - **If YES**: Proceed to Step 3A (Direct Text Extraction)
   - **If NO**: Proceed to Step 3B (OCR Processing)
3. Determine page count and document structure

### Step 3A: Direct Text Extraction (Digital PDFs)
1. Use text extraction library (PyPDF2, pdfplumber, or pypdf) to extract text from each page
2. Preserve text ordering and structure:
   - Maintain column layouts if possible
   - Preserve paragraph breaks
   - Keep line breaks within paragraphs
3. Extract text in order: left-to-right, top-to-bottom
4. For each page:
   - Extract raw text
   - Clean whitespace (normalize multiple spaces/tabs to single space)
   - Preserve intentional line breaks
   - Note any extraction anomalies
5. Combine pages with clear page break markers: `\n\n--- PAGE X ---\n\n`

### Step 3B: OCR Processing (Scanned PDFs)
1. Detect that direct text extraction is insufficient or impossible
2. For each page:
   - Render PDF page to image (300 DPI minimum for accuracy)
   - Apply image preprocessing:
     - Convert to grayscale
     - Auto-contrast adjustment
     - Deskew if tilted (rotate to correct orientation)
     - Remove noise
   - Run OCR engine (Tesseract or cloud-based service)
   - Extract text with confidence scores
   - If confidence < quality threshold, flag for manual review
3. Combine OCR results with clear page markers
4. Add warning annotations for low-confidence sections

### Step 4: Extract Metadata
1. If `include_metadata` is true:
   - Extract standard PDF metadata (title, author, subject, creator, creation date, modification date)
   - Count total pages
   - Note file size
   - Record extraction method used (direct_text or ocr)
   - Calculate overall confidence score (direct=1.0, OCR=average page confidence)
2. Generate metadata object in JSON format

### Step 5: Post-Processing & Quality Assurance
1. **Text Cleanup**:
   - Remove control characters except newlines and tabs
   - Replace multiple consecutive newlines with exactly two
   - Fix common OCR errors (manual review list):
     - "rn" → "m" (common OCR mistake)
     - "l" → "1" or "I" (context-dependent)
     - "0" → "O" (context-dependent)
   - Trim leading/trailing whitespace from document and pages

2. **Validation**:
   - Verify text is non-empty (failed if <50 characters extracted from multi-page doc)
   - Check for character encoding issues
   - Count total words extracted
   - Flag if extraction seems incomplete (e.g., <10% of expected content)

3. **Generate Quality Report**:
   - Pages successfully extracted: N/M
   - Average confidence: X%
   - Warnings generated: [list]
   - Estimated completeness: X%

### Step 6: Format Output
1. Convert extracted text to requested format:
   - **plain_text**: Raw text with page breaks
   - **markdown**: Text with markdown headers for pages (# Page 1, etc.)
   - **json**: Structured object with metadata, pages array, and statistics
2. Ensure output is valid UTF-8
3. Return both text content and metadata object

### Step 7: Return Results
Return success response with:
```json
{
  "success": true,
  "extraction_method": "direct_text|ocr",
  "page_count": 15,
  "extracted_text": "...",
  "metadata": { ... },
  "quality_report": {
    "pages_processed": 15,
    "pages_successful": 15,
    "average_confidence": 0.98,
    "total_words": 5420,
    "warnings": [],
    "completeness_estimate": 0.99
  }
}
```

## Edge Cases & Error Handling

### Case 1: Scanned PDF (Image-Only)
- **Detection**: Zero text extracted in Step 2
- **Action**: Automatically trigger OCR in Step 3B
- **Fallback**: If OCR fails, return error with suggestion to upload original digital file

### Case 2: Corrupted PDF
- **Detection**: PDF library throws parse error or file structure is invalid
- **Action**: Attempt recovery with alternative library
- **Fallback**: If unrecoverable, return error: `corrupted_pdf_unrecoverable`
- **Note**: Corrupted files may have partial recoverable content; attempt extraction from readable sections

### Case 3: Password-Protected PDF
- **Detection**: PDF requires password for text extraction
- **Action**: Attempt without password first, if failed return error asking for password
- **Fallback**: If password provided, decrypt and retry extraction
- **Security Note**: Never log or store passwords; include in request only

### Case 4: Mixed Content (Text + Images)
- **Action**: Extract all selectable text in Step 3A
- **Images**: Count images but do not attempt image-based OCR unless text extraction is empty
- **Hybrid Approach**: If partial text available with low coverage, offer to supplement with image-based OCR

### Case 5: Complex Layouts (Tables, Multi-Column)
- **Detection**: Text extraction includes unusual spacing or unusual character sequences
- **Action**: Attempt to preserve structure using pdfplumber's table detection
- **Fallback**: Return text as-is with warning about layout complexity
- **Manual Review**: Flag pages with complex layouts for QA review

### Case 6: Large Files (>500MB)
- **Action**: Process in chunks (100MB at a time) to manage memory
- **Streaming**: Return results as they're processed (streaming response)
- **Timeout**: Set reasonable timeout (5 minutes per 100MB chunk)
- **Partial Success**: If timeout occurs, return successfully processed pages with warning

### Case 7: Unsupported PDF Features
- **Embedded Fonts**: Ignore special font rendering, extract text content
- **Encryption**: Return error if encrypted without password
- **Digital Signatures**: Include but note that signature validity cannot be verified
- **Form Fields**: Extract field values if possible, flag unfillable fields

### Case 8: Empty or Near-Empty PDF
- **Detection**: <50 characters extracted from document
- **Action**: 
  - Return empty text result with confidence=0
  - Check if file is actually a PDF (not misnamed)
  - Suggest OCR if file appears to be scanned
- **Warning**: Include "document_appears_empty_or_blank" in warnings

### Case 9: OCR Confidence Too Low
- **Detection**: Average confidence score <quality_threshold
- **Action**: 
  - Still return extracted text
  - Include confidence scores per page
  - Add prominent warning about quality
  - Suggest manual review
- **Recommendation**: Flag for human verification in legal office workflow

### Case 10: Timeout or Resource Exhaustion
- **Detection**: Process exceeds time or memory limits
- **Action**: Return partial results with pages successfully processed
- **Recovery**: Include `pages_attempted` vs `pages_successful` in quality report
- **User Action**: Suggest splitting file and reprocessing separately

## Success Criteria

A successful extraction meets these criteria:
1. ✅ File successfully parsed (no corruption errors)
2. ✅ Text extraction returns >0 characters
3. ✅ Metadata extracted (if requested)
4. ✅ Page count matches actual PDF pages
5. ✅ Output format is valid (JSON/Markdown/Text)
6. ✅ Quality report generated with no critical warnings
7. ✅ Confidence score acceptable (>0.7 for legal use)

## Failure Criteria

Fail immediately if:
- ❌ File does not exist or is inaccessible
- ❌ File is not a valid PDF (magic bytes fail)
- ❌ PDF is encrypted without password provided
- ❌ Extraction timeout exceeded
- ❌ OCR confidence critically low (<0.5) for scanned PDFs
- ❌ Output validation fails (invalid UTF-8, structure malformed)

## Dependencies & Tools

**Recommended Libraries:**
- Python: `pypdf` (formerly PyPDF2), `pdfplumber`, `pdf2image`
- OCR Engine: `pytesseract` (wraps Tesseract) or cloud service (Google Cloud Vision, AWS Textract)
- Image Processing: `Pillow` (PIL), `OpenCV`

**System Requirements:**
- Python 3.8+
- For OCR: Tesseract-OCR installed (`tesseract-ocr` package)
- For scanned PDFs: `poppler` utilities

**External Services (Optional):**
- AWS Textract: Higher accuracy OCR, handles complex layouts
- Google Cloud Vision: Good for multilingual documents
- Azure Computer Vision: Enterprise option with better SLA

## Performance Expectations

| Document Type | Typical Time | Output Size | Confidence |
|---|---|---|---|
| Digital PDF (1-10 pages) | <1 second | 2-50 KB | >0.99 |
| Digital PDF (50+ pages) | 2-5 seconds | 50-500 KB | >0.99 |
| Scanned PDF (1-10 pages) | 3-10 seconds | 2-50 KB | 0.85-0.95 |
| Scanned PDF (50+ pages) | 30-120 seconds | 50-500 KB | 0.80-0.90 |
| Mixed/Complex Layout | 2x baseline | Same | -0.1 to baseline |

## Legal Office Context

**Applicable Use Cases:**
- Contract extraction and analysis
- Case file digitization
- Document discovery preparation
- Automated document intake and categorization
- Compliance document processing

**Compliance Considerations:**
- Preserve document integrity (no modifications to extracted text)
- Maintain audit trail of extraction operations
- Ensure extracted content is confidential-ready
- Document extraction method for legal admissibility

**Quality Standards:**
- Minimum 95% accuracy for direct text extraction
- Minimum 85% accuracy for OCR on good-quality scans
- All extractions logged with confidence metadata
- Extraction method documented (critical for legal proceedings)

## Example Usage

```json
// Input Request
{
  "pdf_file_path": "/documents/contract_2024.pdf",
  "target_format": "markdown",
  "include_metadata": true,
  "quality_threshold": 0.85
}

// Output Response
{
  "success": true,
  "extraction_method": "direct_text",
  "page_count": 8,
  "extracted_text": "# Page 1\n\nCONTRACT AGREEMENT...",
  "metadata": {
    "title": "Service Agreement",
    "author": "Jane Doe",
    "created_date": "2024-01-15T10:30:00Z",
    "page_count": 8
  },
  "quality_report": {
    "pages_processed": 8,
    "pages_successful": 8,
    "average_confidence": 0.994,
    "total_words": 3847,
    "warnings": [],
    "completeness_estimate": 0.99
  }
}
```

## Maintenance & Updates

- **Version**: 1.0.0
- **Last Updated**: 2026-02-28
- **OCR Engine Versions**: Tesseract 5.x or later
- **Breaking Changes**: None currently
- **Known Limitations**: 
  - Handwritten text in PDFs requires specialized models
  - Extreme font corruption may prevent extraction
  - Right-to-left text languages may have ordering issues
