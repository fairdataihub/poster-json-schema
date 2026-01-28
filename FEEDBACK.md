# Schema Feedback: UC3 and DataCite

This document tracks feedback received from UC3 (University of California Curation Center) and DataCite regarding the poster JSON schema, along with proposed changes and rationale.

## Feedback Summary

### Source: UC3 Team Member
**Date Received**: 2026-01-09

#### Issue 1: `posterContent` is too poster-specific
> "Consider doing something more general for posterContent (e.g. just content), so this could be used for other resource types"

**Current Implementation**:
- Field: `posterContent`
- Contains: `sections` array with `sectionTitle` and `sectionContent`
- Also: `unstructuredContent` for non-sectioned content

**Proposed Change**: Rename `posterContent` → `content`

**Rationale**: 
- Makes the schema extensible to other visual/document resource types (presentations, infographics, one-pagers)
- Maintains backward compatibility through schema versioning
- The `sections` structure is already generic enough to apply to other document types

**Migration Notes**:
- Update schema `$id` to reflect new version (v0.2)
- Document the rename in CHANGELOG
- Consider keeping `posterContent` as deprecated alias temporarily

---

#### Issue 2: `domain` and `species` terminology confusion
> "Using domain and species with discrete meanings from the two different ontologies could be confusing (meaning there is domain in the colloquial sense but then also biological domain as the highest level above species)."

**Current Implementation**:
- Field: `domain` - "Primary domain or field of study for the research presented in the poster"
- Examples: "Machine Learning", "Clinical Medicine", "Bioinformatics"

**Issue**: 
- In biological taxonomy, "domain" is the highest rank (above kingdom/phylum/class/order/family/genus/**species**)
- Having both `domain` (general research field) and potentially `species` (biological organism) creates semantic confusion
- A poster about "Domain Bacteria" in the domain of "Microbiology" would be confusing

**Proposed Changes**:
1. Option A: Rename `domain` → `researchField` or `disciplineArea`
2. Option B: Rename `domain` → `subject` and merge with existing `subjects` array
3. Option C: Keep `domain` but add clear documentation distinguishing from biological taxonomy

**Recommendation**: Option A - Rename to `researchField`
- Clear, unambiguous terminology
- Distinct from biological taxonomy terms
- Self-documenting field name

---

#### Issue 3: Conference metadata may already exist in DataCite
> "I might also look into how conference name or venue is already being modeled in the DataCite schema, as I expect this already has some form of representation"

**Current Implementation**:
Our `conference` object contains:
- `conferenceName`
- `conferenceLocation`
- `conferenceUri`
- `conferenceIdentifier`
- `conferenceIdentifierType`
- `conferenceSchemaUri`
- `conferenceStartDate`
- `conferenceEndDate`
- `conferenceAcronym`
- `conferenceSeries`

**DataCite Analysis**:

DataCite 4.6 provides `relatedItem` (property 20) for describing container/host relationships. This is commonly used for:
- Journal articles within journals
- Book chapters within books
- **Conference papers within proceedings**

The `relatedItem` structure includes:
- `relatedItemType` (e.g., "ConferenceProceeding")
- `titles`
- `volume`, `issue`, `number`
- `firstPage`, `lastPage`
- `contributors`

However, DataCite's `relatedItem` does NOT natively include:
- Conference dates (start/end)
- Conference location
- Conference series
- Conference acronym

**Proposed Approach**:
1. Use DataCite's `relatedItem` with `relatedItemType: "ConferenceProceeding"` for linking to proceedings
2. Keep our `conference` extension for poster-specific conference metadata not covered by DataCite
3. Document the relationship between `relatedItem` and `conference` in the schema
4. Consider proposing conference-specific extensions to DataCite through their RFC process

**Recommendation**: Hybrid approach
- Add support for `relatedItem` per DataCite 4.6 for standard proceedings links
- Maintain `conference` extension for richer conference metadata
- Document clearly which fields are DataCite-native vs. poster-schema extensions

---

## Implemented Schema Changes

| Change | From | To | Status |
|--------|------|-----|--------|
| Rename posterContent | `posterContent` | `content` | ✅ Done |
| Rename domain | `domain` | `researchField` | ✅ Done |
| Add relatedItem | N/A | Add DataCite 4.6 `relatedItem` | Pending |
| Document conference | N/A | Clarify as extension | ✅ Done |

---

## Action Items

- [x] Discuss proposed changes with UC3 and DataCite contacts
- [x] Implement schema changes
- [ ] Submit RFC to DataCite for conference metadata enhancements
- [x] Update example JSON files to reflect changes
- [ ] Update posters.science extraction pipeline for new field names

---

## Contact Points

- **UC3**: (feedback via Bhavesh Patel)
- **DataCite**: Kelly (email pending)

---

## References

- [DataCite Metadata Schema 4.6](https://schema.datacite.org/)
- [DataCite relatedItem documentation](https://support.datacite.org/docs/datacite-metadata-schema-v46-properties-overview)
- [DataCite Schema RFC Process](https://schema.datacite.org/contribute.html)

