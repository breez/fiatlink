# FiatLink - Fiat On/Off ramp spec 
API specification for bitcoin on and off ramps. Read more [here](https://medium.com/breez-technology/building-on-lightning-flash-is-reinventing-caribbean-money-with-the-breez-sdk-d26ef8b81fb8).

The goal of this project is to provide a unified API specifications for fiat ramps to create interoperability and easier integration of multiple on and off-ramps into apps.

As first priority, we want to support on/off-ramps standardization with lightning.

[Telegram](https://t.me/+6wVEdfztX1Y1NTFk) invite for the group.

## Status Specification 
All FiatLink specifications include a "Status" field. "Status" can be one of the following:

- **Draft** - The specification is still under active development and subject to change. Implementation is not recommended at this time.
- **For Implementation** - The specification has been widely reviewed by Fiatlink participants, is believed to have addressed all raised issues, and participants recommend this specification to be implemented.
- **Stable** - The specification has been implemented by at least one client and one broker, which are developed by at least two different organizations or open-source project teams, and both development teams have reported interoperability without further modifications or clarifications of the specification.

## Specs
1. [FLS00 - Transport layer](./FLS00/README.md) **DRAFT**
2. [FLS01 - Non-kyc on-ramp API spec](./FLS01/README.md) **For Implementation**

## API and SDK documentation
- [OpenAPI](https://docs.fiatlink.co/)
