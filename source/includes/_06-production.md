# Production

After testing and accreditation are complete, follow the production readiness checklist below:

- [Create production Sale System build](#production-create-production-sale-system-build)
- [Complete PVT](#production-complete-pvt)
- [Organise a pilot](#production-organise-a-pilot)

## Create a production Sale System build 

- If the Sale System is using a DataMesh library, ensure it is updated to the latest version
- Update the Sale System "static" settings to the production values provided by DataMesh. Settings to be updated are listed below. See [Sale System settings](#design-your-integration-sale-system-settings) for more information.
  - `ProviderIdentification`, the name of the busniess which creates the Sale System.
  - `ApplicationName`, the name of the Sale System.
  - `CertificationCode`, a GUID which uniquly identifies the Sale System. 
  - `SoftwareVersion`, the internal build version of this Sale System. DataMesh must configure this value on Unify. 
- Ensure the Sale System is connecting to the [production endpoint](#cloud-api-reference-endpoints).
  - The production endpoint uses a different [SSL certificate](#cloud-api-reference-security).

## Complete PVT

[Contact DataMesh]((mailto:integrations@datameshgroup.com)) to orgainse a production verification testing window.

This process involves working with a DataMesh representative to pair your production Sale System build with a production POI Terminal, and running through some basic tests.

## Organise a pilot

Work with DataMesh to co-ordinate on a pilot site. 

- Before the pilot, ensure they have been updated to the correct Sale System version 
- If the POI Terminal will be utilising Wi-Fi, ensure it is available at all required locations in the pilot site
- Check that any network firewalls allow outgoing connections to ports 443, 4000, and 5000
- Ensure a Sale System support representative is available for the pilot 
- DataMesh will work with the customer to create the required accounts, configure and deliver terminals 

