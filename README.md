# JSON Schema for Machine Actionable Scientific Poster

JSON Schema for machine-actionable scientific poster metadata. Derived from and compatible with [DataCite Metadata Schema 4.6](https://datacite.org/), with poster-specific extensions for conference presentations.

## Overview

Scientific posters are a primary means of scholarly communication at conferences. They are usually shared in PDF or similar format. Given that the structure vary greatly from poster to poster, their content remains largely inaccessible to programmatic discovery and analysis. We propose here a JSON schema for representing poster that enables:
- **Machine-actionable** analysis of poster content
- **FAIR compliance** when sharing posters 
- **AI-ready**, structured representation for automated processing

This schema is developed as par of our development of [posters.science](https://posters.science), where we are building a tool for automatically creating a JSON Schema version given the PDF file of a psoter. Our vision is that anytime a poster is shared (as PDF or other similar format), it will be accompanied with a poster.json file that comply with this JSON schema that enables greater findabily and reusability. We hope that this schema and its assciated tools can be useful to anyone wanting to represent scientific posters into a machine-actionable format.

[ADD Figure with screenshot of poster on left and screenshot of json representation on the right.]

The schema extends DataCite's mandatory and recommended properties with poster-specific fields:

## Development Approach

The schema is based on the [DataCite Metadata Schema 4.6](https://datacite.org/), with poster-specific adjustement, including extensions for including conference-related information.

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


## Example
We provide an example in the [example](example) folder using the poster available at https://zenodo.org/records/17268692.


## Related Standards
- **DataCite Metadata Schema 4.6**: [schema.datacite.org](https://schema.datacite.org)
- **ORCID**: [orcid.org](https://orcid.org) - Author identifiers
- **ROR**: [ror.org](https://ror.org) - Organization identifiers
- **Crossref Funder Registry**: Funding organization identifiers

## License
This work is shared under the MIT License. See [LICENSE](LICENSE) for details.

## Contributing
Contributions and feedback are welcomed. Please open an issue or pull request on [GitHub](https://github.com/fairdataihub/posters-science-json-schema).

## How to cite
If you use this work, please cite this repository following the instructions on provided in [CITATION.cff](CITATION.cff).
