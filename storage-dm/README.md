<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">
<img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a>
<br />The Common Archive Observation Model is licensed under the
<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">
Creative Commons Attribution-ShareAlike 4.0 International License</a>.

# high level features

one or more independent global inventory(ies) of all files and their locations
- specialised global inventory(ies) can be created (eg vault can have it's own copy of global)

arbitrary organisation of archive files (File.uri)
- enables arbitrary mirroring policies at each site

incremental metadata propagation and robust metadata validation
- site to global
- global to site

scalable validation using uri, uriBucket
- CAOM vs storage
- vault vs storage
- global vs site
- site vs global

multiple storage backends in use at any one time

# transition plan features

no throw-away transition tools or development (CT: continuous transition)

no "big switch": 
- start with AD site
- operate with AD in parallel with other (types of) sites
- eventually retire AD?

# what's NOT included in design/plan

monitoring, control, or config of back end storage systems (use their tools)

monitoring or control of processes and services (use kubernetes)

reporting (get logs from containers)

# storage-dm
Storage Inventory (SI) Data Model

This module contains a <a href="http://www.ivoa.net/documents/VODML/index.html">VO-DML</a> 
description of the Storage data model and test code to validate the model using the 
<a href="https://github.com/opencadc/core/tree/master/cadc-vodml">cadc-vodml</a> tools.

Development starts with the UML diagrams and the current version may be ahead of the 
<a href="https://github.com/pdowler/storage/blob/master/storage-dm/docs/index.html">SI VO-DML</a> documentation. 

<img alt="Storage System UML - development version" style="border-width:0" 
src="https://github.com/pdowler/storage/raw/master/storage-dm/src/main/resources/storage-inventory-0.2.png" />

# File.uri
This URI is a globally unique identifier that is typically known to and may be defined by some other system 
(e.g. an Artifact.uri in CAOM or storageID in a storage-system backed VOSpace like ivo://cadc.nrc.ca/vault). 

{scheme}:{path}

ivo://{authority}/{name}?{path}

For all URIs, the filename would be the last component of {path}. There is no explicit namespace: arbitrary sets of files
can be defined with a prefix-pattern match on fileID. Such sets of files would be used in validation and in site mirroring 
policies.

For resolvable ivo URIs, the resourceID can be extracted by dropping the query string. The resourceID 
can be found in a registry and allows clients to find data services. This form allows for generic tools to resolve
and access files from external systems. Example usage of equivalent fileID values:

ad:{archive}/{filename} *classic*

cadc:{path}/{filename} *new*

ivo://cadc.nrc.ca/{archive}?{path}/{filename} *resolvable archive*

ivo://cadc.nrc.ca/{srv}?{path}/{filename} *resolvable multi-archive service*

The directly resolveable fileID (ivo scheme) can be used to extract a resourceID (up to the ?) and perform a registry lookup.
The resulting record would contain at least two capabilities: a transfer negotiation or a files endpoint and a permissions endpoint. Classic (ad) and basic (cadc scheme) usage is to use the a shortcut scheme that is configured to be equivalent to the multi-archive data centre. The first form (with the archive in the resourceID) allows for changes from a common to a different
permission service (for that archive); the second form is the resolvable version of current practice. The short forms (ad and cadc schemes) require configuration at storage sites to map the scheme (e.g. cadc -> ivo://cadc.nrc.ca/archive) for capability lookup. For externally harvested and sync'ed CAOM artifacts, we would use the URI as-is in the fileID. For the simple form 
(e.g. mast:HST/path/to/file.fits) we would configure the scheme (mast) as a shortcut to the CAOM archive metadata service.

Validation of CAOM versus storage requires querying CAOM (1+ collections) and storage (single namespace) and 
cross-matching URIs to look for anomolies.

For vault (vospace), data nodes would have a generated identifier and use something like 
ivo://cadc.nrc.ca/vault?{uuid}/{filename} or vault:{uuid}/{filename}. For the latter, storage sites would require
configuration to support the shortcut (as would other instances). There should be no use of the "vos" scheme in a fileID so paths within the vospace never get used in storage and move/rename operations (container or data nodes) do not effect the storage system.  Validation of vault versus storage requires querying vault (and building fileID values programmatically unless
vaultdb stores the full URI) and storage and cross-matching to look for anomolies.

Since we need to support mast and vault schemes (at least), it is assumed that we would use the "cadc" scheme going 
forward and support (configure) the "ad" scheme for backwards compatibility. 

# external services
Services that make up the storage site or global inventory depend on other CADC services. The registry lookup API permits caching; this is already implemented and effective. The permissions API could be designed to support caching or mirrors. The user and group APIs could be modified to support caching or mirroring. Caching: solve a problem by adding another problem.

<img alt="storage site deployment" style="border-width:0" 
src="https://github.com/pdowler/storage/raw/master/storage-dm/docs/storage-external-services.png" />

# storage site
A storage site tracks all files written to local storage. These files can be harvested by anyone: central inventory, 
clones, backups, etc. A storage site is fully consistent w.r.t local file operations (e.g. put + get + delete).

Storage sites:
- track writes of local files (persist File + StorageLocation)
- implement get/put/delete service API
- implement policy to sync File(s) from global
- implement file sync from other sites using global transfer negotiation
- implement validation of file metadata vs storage

A storage site maintains StorageLocation->File for File(s) in local storage. Local policy syncs a subset of File(s)
from global; new File(s) from other sites would have no matching StorageLocation and file-sync would know to retrieve
the file. Local put creates the StorageLocation directly.

<img alt="storage site deployment" style="border-width:0" 
src="https://github.com/pdowler/storage/raw/master/storage-dm/docs/storage-site-deploy.png" />

# global inventory
A global inventory service is built by harvesting sites, files, and deletions from all known sites.
A global inventory is eventually consistent.

Global inventory (may be one or more):
- harvests Site metadata from storage sites
- harvests File + DeletedFile metadata/events from storage sites (persist File + SiteLocation)
- implements transfer negotiation API

Rather than just accumulate new instances of File, the harvested File may be a new File or an existing File 
at a new site that has to be merged into the global inventory. Thus, File.lastModified in a global inventory is not 
equal to the values at individual storage sites. TBD: should global store the min(lastModified from sites)? max(lastModified from sites)? act like origin and store curent timestamp?.

The metadata-sync tools could allow for global policy so one could maintain a global view of a defined set of file. 
This might be useful so vault could maintain it's own global view and not rely on an external global inventory service 
for transfer negotiation.

<img alt="global storage inventory deployment" style="border-width:0" 
src="https://github.com/pdowler/storage/raw/master/storage-dm/docs/global-inventory-deploy.png" />

# patterns, ideas, and incomplete thoughts...

files (basic put/get/delete):
- PUT /srv/files/{uri}
- GET /srv/files/{uri}
- DELETE /srv/files/{uri}
- cannot modify File.uri without delete+put with new uri (chose wisely)
- mirroring policy at sites will be based on information in the File.uri (chose wisely)
- cannot directly access vault files at a site - must use transfer negotiation (no real change)
- vault transfer negotiation maps vos {path} to DataNode.uuid and then negotiates with global storage inventory
- vault implementation could maintain it's own global inventory of vault files

locate (transfer negotiation):
- negotiate with global to get: locate available copies and return URL(s), order by proximity
- negotiate with global to put: sites that are writable, order by proximity; try to match policy?
- negotiate with global to delete: same as get
- global should implement heartbeat check with sites so negotiated transfers likely to work

overwrite a file at storage site:
- write with new File UUID, File.lastModified/metaChecksum must change, global harvests, sites harvest
- add DeletedFile record for old UUID
- before global harvests: eventually consistent (but detectable by clients in principle)
- after global harvests: consistent (only new file accessible, sites limited until sync'ed
- direct access: other storage sites would deliver previous version of file until they sync
- race condition - put at two sites -> two File(s) with same fileID but different id: keep max(File.lastModified)

how does delete file get propagated?
- delete from any site with a copy; delete harvested to global and then to other sites
- process harvested delete by File UUID so it is decoupled from put new file with same name
- negotiate with global to find a copy to delete, delete one or more copies?
- race condition - delete and put at different sites: the put always wins
- delete by File UUID is idempotent so duplicate DeletedFile records are ok

how does global learn about copies at sites other than the original?
- when a site syncs a file (adds a local StorageLocation): update the File.lastModified
- global metadata-sync from site(s) will see this and add a local SiteLocation
- the File.id and File.metaChecksum never change during sync
- global's view of the File.lastModified will be the latest change at all sites, but that doesn't stop it from getting an
  "event" out of order and still merging in the new SiteLocation

how would a temporary cache instance at a site be maintained?
- site could accept writes to a "cache" instance for File(s) that do not match local policy
- site could delete once global has other SiteLocation(s), update File.lastModified so global will detect 
  and remove SiteLocation
- files could sit there orphaned if no one else wants/copies them

how does global inventory validate vs site?  how does site validate vs global (w.r.t. policy)?
- get list of File(s) from the site and compare with File+SiteLocation
- add missing File(s)
- add misssing SiteLocation(s)
- remove orphaned SiteLocation(s)
- remove File(s) with no SiteLocation??
- use uriBucket prefix to batch validation

how does a storage site validate vs local storage system?
- compare List of File+StorageLocation vs storage content using storageID
- use uriBucket prefix to batch validation??
- if file in inventory & not in storage: pending file-sync job
- if file in storage and not in inventory && can generate File.uri: query global, create StorageLocation, maybe create File
- if file in storage and not in inventory && cannot generate File.uri: delete from storage? alert operator?

what happens when a storage site detects that a File.contentChecksum != storage.checksum?
- if copies-in-global: delete the File and the stored file and re-sync w.r.t policy?
- if !copies-in-global: mark it bad && human intervention

should harvesting detect if site File.lastModified stream is out of whack?
- non-monotonic, except for volatile head of stack?
- clock skew
