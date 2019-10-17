# Risks and Mitigations for the Storage project
![](https://www.pivotpointsecurity.com/wp-content/uploads/2016/08/Updated-Risk-Matrix.jpg)

## Cutout Risks on CEPH
Streaming a file from a CEPH object store to a cutout service will be slower that the current cutout system that uses direct file access. The may result in unacceptable time to access pixel or worse, timeouts on synchronous requests.

 * Likelihood: High
 * Impact: High

### Mitigations
1. Don't use CEPH object store
 * Changing both SSC and Westgrid deployements
 * Converting the disk management parts of AD to Java and Postgres or finding an open source disk management softwaree
 * Development effort required
2. Develop python clients to use async requests to the cutout service
 * Addresses the timeout issue but not the speed issue
 * Changes the current user workflow
 * Development effort required
 * User transition effort required
3. If offset reads are reasonably supported by CEPH then gathering more file structure metadata in CAOM databases:
 * Develop the data models for FITS and eventually HDF
 * Extract the header info from every FITS file in the collections and ingest into the new data models
 * Requires development effort
 * Requires operations effort to ingest

## Kubernetes Risks
SSC or UVic RCS do not deliver an operational Kubernetes on the project time scale.

 * Likelihood: Medium
 * Impact: Medium

### Mitigations

1. Deploy containers in VMs at each site
 * More complicated configuration and maintenance
 * Requires Ops effort

## External Schedule Risks
SSC or UVic RCS want us off the old storage hardware before the storage site software stack is ready.

 * Likelihood: ???
 * Impact: ???

### Mitigations

1. Requires input from John

## Internal Schedule Risks
Storage space is required before the CEPH site software stack is available.

 * Likelihood: ???
 * Impact: ???

### Mitigations

1. Requires input from John

## Urgent Treasury Board or NRC changes
TB or NRC request major CLF changes that will divert developer resources away from the storage project.

 * Likelihood: Low
 * Impact: High

 ### Mitigations

1. Find a minimally acceptable adoption of CLF until storage project is completed.
2. Contract hiring of individuals to focus on CLF.
3. Divert effort from other project areas to enable CLF conversions.

 
 
