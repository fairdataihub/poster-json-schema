# posters-science-json-schema

JSON Schema for machine-actionable scientific poster metadata. Derived from and compatible with [DataCite Metadata Schema 4.6](https://datacite.org/), with poster-specific extensions for conference presentations.

## Overview

Scientific posters are a primary means of scholarly communication at conferences, yet their content remains largely inaccessible to programmatic discovery and analysis. This schema enables:

- **Machine-actionable metadata** extraction from poster content
- **DOI registration** compatibility via DataCite
- **FAIR compliance** for poster artifacts
- **AI-ready** structured data for automated processing
  
## Real-World Example

To demonstrate the schema in practice, we provide a complete extraction example from a real poster archived on Zenodo:

### Source Poster

| Field | Value |
|-------|-------|
| **Title** | Presenting the Actionable Guidelines for FAIR Research Software Task Force |
| **DOI** | [10.5281/zenodo.17268692](https://zenodo.org/records/17268692) |
| **Conference** | US Research Software Engineering Conference 2025 (USRSE'25) |
| **Authors** | Bhavesh Patel, Daniel Garijo, Actionable FAIR4RS Task Force |

### Extracted JSON

The JSON extraction for this poster is available in our examples repository:

- **JSON files**: [fairdataihub/posters-science-json-examples/17268692](https://github.com/fairdataihub/posters-science-json-examples/tree/main/17268692)
- **Poster GitHub repo**: [fairdataihub/actionableFAIR4RS-USRSE-2025](https://github.com/fairdataihub/actionableFAIR4RS-USRSE-2025)

This example shows how a poster PDF is transformed into machine-actionable metadata following the schema, including creator information with ORCIDs, conference metadata, and structured poster content sections.

## Extraction Method

The Posters.science platform uses a two-stage AI pipeline to convert poster PDFs and images into structured JSON:

### Stage 1: Text Extraction

The system automatically selects the extraction method based on file type:

| Input Format | Extraction Tool | Description |
|--------------|-----------------|-------------|
| **PDF** | pdfalto | Converts PDF to ALTO XML preserving layout structure and reading order |
| **Images** (JPG/PNG) | Qwen2-VL-7B | Vision-language model for direct pixel-to-text extraction |

### Stage 2: JSON Structuring

Raw text is converted to structured JSON using **Llama 3.1 8B Instruct** (Q8 quantized) served via Ollama:

- Section-aware extraction identifying: Abstract, Introduction, Methods, Results, Discussion, Conclusions, References
- Verbatim text preservation to maintain scientific accuracy
- Adaptive token allocation with fallback for long documents

### Validation Performance

The pipeline achieves **100% compliance** on validation metrics (≥0.75 threshold) across 10 reference posters:

| Metric | Score | Description |
|--------|-------|-------------|
| Word Capture | 0.963 | Lexical completeness |
| ROUGE-L | 0.887 | Sequential text preservation |
| Number Capture | 0.936 | Quantitative data integrity |
| Field Proportion | 1.105 | Structural completeness |

For implementation details, see the [poster extraction pipeline](https://github.com/fairdataihub/posters-science-posterextraction-beta).

## Schema Structure

The schema extends DataCite's mandatory and recommended properties with poster-specific fields:

### DataCite Core Properties

| Property | Requirement | Description |
|----------|-------------|-------------|
| `creators` | Mandatory | Authors with ORCID and affiliation support |
| `titles` | Mandatory | Poster title(s) |
| `publisher` | Mandatory | Conference organizer or institution |
| `publicationYear` | Mandatory | Year of presentation |
| `subjects` | Recommended | Keywords and classification terms |
| `dates` | Recommended | Presentation and conference dates |
| `language` | Recommended | Primary language (ISO 639) |
| `types` | Mandatory | Resource type (Conference Poster) |
| `relatedIdentifiers` | Recommended | Links to papers, datasets, software |
| `formats` | Recommended | File format (PDF, PNG, etc.) |
| `rightsList` | Optional | License information |
| `descriptions` | Recommended | Abstract, Methods, Technical Info |
| `fundingReferences` | Recommended | Grant and funder information |

### Poster-Specific Extensions

| Property | Description |
|----------|-------------|
| `conference` | Conference name, location, dates, URI, acronym |
| `posterContent` | Structured sections extracted from poster |
| `imageCaption` | Captions for figures in the poster |
| `tableCaption` | Captions for tables in the poster |
| `domain` | Research domain or field of study |
| `species` | Species information if applicable |

## Accepted Poster File Formats

### Primary Formats (Recommended)

| Format | Extension | Notes |
|--------|-----------|-------|
| **PDF** | `.pdf` | Most common. Must be single-page. |
| **PNG** | `.png` | Lossless image format. |
| **JPEG** | `.jpg`, `.jpeg` | Widely used image format. |
| **TIFF** | `.tiff`, `.tif` | High-quality, print-ready. |
| **PowerPoint** | `.pptx` | Must be single-slide. |
| **SVG** | `.svg` | Vector format from Figma/Illustrator. |

### Additional Supported Formats

| Format | Extension | Notes |
|--------|-----------|-------|
| PowerPoint Legacy | `.ppt` | Older PowerPoint format. |
| OpenDocument | `.odp` | LibreOffice/OpenOffice. |
| Keynote | `.key` | Apple Keynote (macOS). |
| EPS | `.eps` | Encapsulated PostScript. |
| WebP | `.webp` | Modern web image format. |
| GIF | `.gif` | Limited colors. |
| BMP | `.bmp` | Uncompressed bitmap. |
| HEIF | `.heic`, `.heif` | Apple high-efficiency. |

### Requirements

- **Single-page/single-slide** documents only for multi-page formats
- Multi-page documents are flagged for review
- Recommended export order: PDF → SVG → PNG

## Example JSON Structure

```json
{
  "$schema": "https://posters.science/schema/v0.1/poster_schema.json",
  "creators": [
    {
      "name": "O'Neill, Jamey",
      "nameType": "Personal",
      "nameIdentifiers": [
        {
          "nameIdentifier": "https://orcid.org/0009-0001-8532-8405",
          "nameIdentifierScheme": "ORCID",
          "schemeURI": "https://orcid.org"
        }
      ],
      "affiliation": [
        {
          "name": "San Diego State University",
          "affiliationIdentifier": "https://ror.org/01zj77w89",
          "affiliationIdentifierScheme": "ROR"
        }
      ]
    }
  ],
  "titles": [
    {
      "title": "CarD-T: LLM Automated Literature Review for Carcinogen Analysis"
    }
  ],
  "publisher": {
    "name": "American Association for Cancer Research"
  },
  "publicationYear": 2025,
  "conference": {
    "conferenceName": "AACR Annual Meeting 2025",
    "conferenceLocation": "Chicago, Illinois, USA",
    "conferenceStartDate": "2025-04-25",
    "conferenceEndDate": "2025-04-30",
    "conferenceAcronym": "AACR 2025"
  },
  "posterContent": {
    "sections": [
      {
        "sectionTitle": "Abstract",
        "sectionContent": "Pipeline combining transformer ML with probabilistic analysis..."
      },
      {
        "sectionTitle": "Methods",
        "sectionContent": "Named Entity Recognition ELECTRA LLM trained on PubMed..."
      }
    ]
  },
  "types": {
    "resourceType": "Conference Poster",
    "resourceTypeGeneral": "Image"
  }
}
```

## Ground Truth Examples

The [fairdataihub/posters-science-json-examples](https://github.com/fairdataihub/posters-science-json-examples) repository contains manually annotated posters serving as ground truth for AI extraction. Each annotation includes:

| File | Description |
|------|-------------|
| `{id}.pdf` | Original poster file |
| `{id}_raw.md` | Extracted text in structured markdown |
| `{id}.json` | Full metadata JSON (complete schema) |
| `{id}_sub-json.json` | Poster content subset (AI-ready) |

## Related Standards

- **DataCite Metadata Schema 4.6**: [schema.datacite.org](https://schema.datacite.org)
- **ORCID**: [orcid.org](https://orcid.org) - Author identifiers
- **ROR**: [ror.org](https://ror.org) - Organization identifiers
- **Crossref Funder Registry**: Funding organization identifiers

## License

MIT License. See [LICENSE](LICENSE) for details.

## Contributing

Contributions welcome. Please open an issue or pull request on [GitHub](https://github.com/fairdataihub/posters-science-json-schema).
