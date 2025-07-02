# Digital Backbone Request for Comments

Welcome to the Request for Comments (RFC) repository for the Digital Backbone of the UK Public Sector (UKPS).

This repository is managed by the Digital Backbone (DBB) team within the Government Digital Service (GDS).

Guided by the [blueprint for modern digital government](https://www.gov.uk/government/publications/a-blueprint-for-modern-digital-government/a-blueprint-for-modern-digital-government-html),
this repository is for proposals on the Digital Backbone: the shared integration, orchestration, and technology for
building joined-up public services and people across the UK public sector.

Our goal is a lightweight and collaborative process. For significant items, final approval is managed by the GDS
Technical Design Council (TDC). Finalised decisions will be documented in our
[decision-records](https://github.com/govuk-digital-backbone/decision-records) repository.

Proposals can include, but are not limited to:

- APIs: Mandating and standardising APIs for all new central government services.  
- Events: Having a common technical language and promote event-based architectures.
- Common Components: The strategy and extension of shared components like GOV.UK One Login, Pay, and Notify.  
- Developer Experience (DevEx): Improving the tools, documentation, and processes for developers working inside and
outside of UKPS.
- Cross-Public Sector Collaboration: Common tools to enable public servants to work more easily across organisational boundaries.
- Shared Data Infrastructure: Proposals relating to the National Data Library and enabling data to be shared and reused
with appropriate safeguards.  
- Verifiable Credentials: Promoting verifiable credentials and proposals primarily related to civil and public
servants.  

It is recommended to first discuss a proposal within the
[discussions area](https://github.com/orgs/govuk-digital-backbone/discussions/categories/request-for-comments).

## The RFC Process

We use a simple status system for proposals:

- **Proposed**: The RFC is open for discussion and feedback.
- **Accepted**: The proposal has been agreed upon and will be implemented.
- **Superseded**: A previously accepted RFC has been replaced by a newer one.
- **Archived**: The RFC is no longer under active consideration. This may be because the proposal was withdrawn, is no
longer relevant, or was not selected for implementation.

Anyone can submit a proposal by creating a pull request. The DBB team is responsible for allocating RFC numbers and will
review all proposals. Please note that the team may choose not to allocate a number if the proposal is not within the
scope of the Digital Backbone.
See the [template for more details](rfcs/dbb-rfcXXX-template.md).

## Editing past RFCs

RFCs should not be substantially altered after they are accepted as they intended to be kept as a point-in-time record
of a decision. There are however a few reasons why you may change one that has been accepted:

- to fix typos and other minor mistakes  
- to record a status change of the RFC in the YAML header (remember to update the last_reviewed date)  
- to mark an RFC as being superseded with a link to the RFC that supersedes it  
- any relevant post implementation, or post abandonment, supplementary details that would be useful for someone  
interested in the area.

## License

The repository and its contents are [© Crown copyright](https://www.nationalarchives.gov.uk/information-management/re-using-public-sector-information/uk-government-licensing-framework/crown-copyright/)
and available under the terms of the [Open Government Licence v3.0](https://github.com/govuk-digital-backbone/.github/blob/main/LICENCE.md).
