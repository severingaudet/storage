<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">
<img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a>
<br />The Common Archive Observation Model is licensed under the
<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">
Creative Commons Attribution-ShareAlike 4.0 International License</a>.

# storage-dm
Storage Inventory (SI) Data Model

This module contains a <a href="http://www.ivoa.net/documents/VODML/index.html">VO-DML</a> 
description of the Storage data model and test code to validate the model using the 
<a href="https://github.com/opencadc/core/tree/master/cadc-vodml">cadc-vodml</a> tools.

Development starts with the UML diagrams and the current version may be ahead of the 
<a href="https://github.com/pdowler/storage/blob/master/storage-dm/docs/index.html">SI VO-DML</a> documentation. 

<img alt="Storage System UML - development version" style="border-width:0" 
src="https://github.com/pdowler/storage/raw/master/storage-dm/src/main/resources/storage-inventory-0.1.png" />

# File.fileID thoughts...
This URI is a globally unique identifier that is typically known to and may be defined by some other system 
(e.g. an Artifact.uri in CAOM or storageID in a storage-system backed VOSpace like ivo://cadc.nrc.ca/vault). 

{scheme}:{path}

ivo://{authority}/{name}?{path}

For all URIs, the filename would be the last component of {path}. There is no explicit namespace: arbitrary sets of files
can be defined with a prefix-pattern match on fileID. Such sets of files would be used in validation and in site mirroring 
policies.

For resolvable ivo URIs, the resourceID can be extracted by dropping the query string. The resourceID 
can be found in a registry and allows clients to find data services. This form allows for generic tools to resolve
and sync files from external systems. Example usage of equivalent fileID values:

ad:{archive}/{filename} *classic*

cadc:{archive}/{filename} *new but still flat*

ivo://cadc.nrc.ca/data?{archive}/{filename} *resolvable data service but still flat*

Basic archive usage is to use the cadc scheme: cadc:{path} where the path would contain {archive}/{filename} if we
want/need to match the current flat structure. Validation of CAOM versus storage requires querying CAOM (one or 
more collections) and storage (prefix-pattern match on fileID) and cross-matching URIs to look for anomolies. 

For externally harvested and sync'ed CAOM artifacts, we would use the URI as-is in the fileID
(e.g. mast:HST/path/to/file.fits). Validation of CAOM versus storage requires querying CAOM (one or more collections) 
and storage (prefix-pattern match on fileID) and cross-matching URIs to look for anomolies.

For the resolvable (ivo) form, the service would provide one or more file-access APIs (transfer negotiation and/or
simple URL). This sort of usage means less code on the client side to work with more data providers, unlike the custom
scheme approach currently used in CAOM).

For vault (vospace), data nodes would have a generated identifier (e.g. cadc:vault/{uuid}).
This way paths within the vospace don't end up in storage and move operations in vospace do not effect
the storage system. There should be no use of the "vos" scheme in a fileID. Validation of vault versus storage
requires querying vault (and building fileID values programmatically unless vaultdb stores the full URI) and storage (prefix-pattern match on fileID) and cross-matching to look for anomolies.

# storage site
A storage site tracks all files written to local storage. These files can be harvested by anyone: central inventory, 
clones, backups, etc. A storage site is fully consistent wrt local file operations (e.g. put + get + delete).

Storage sites:
- track writes of local files
- implement get/put/delete service API
- implement policy to sync File(s) from global
- implement file sync from other sites using global transfer negotiation
- implement validation of file metadata vs storage

A storage site always has 1:1 File->Location with only it's own Location.

<img alt="storage site deployment" style="border-width:0" 
src="https://github.com/pdowler/storage/raw/master/storage-dm/docs/storage-site-deploy.png" />

# global inventory
A global inventory service is built by harvesting sites, files, and deletions from all known sites.
A global inventory is eventually consistent.

Global inventory (may be one or more):
- harvests and merges metadata from storage sites where merge means "add location"
- implements transfer negotiation API

Rather than just accumulate new instances of File, the harvested File may be a new File or an existing File 
with a new Location that has to be merged into the global inventory. Thus, File.metaChecksum and File.lastModified
in a global inventory are not equal to the values at individual storage sites.

<img alt="global storage inventory deployment" style="border-width:0" 
src="https://github.com/pdowler/storage/raw/master/storage-dm/docs/global-inventory-deploy.png" />

# patterns and incomplete thoughts

basic put/get/delete:
- PUT /srv/files/{fileID}
- GET /srv/files/{fileID}
- DELETE /srv/files/{fileID}

overwrite a file at storage site:
- write with new storageID, File.lastModified/metaChecksum changes, global harvests
- allows users of global transfer negotiation to detect consistency issues if they want to
- allows storage site to deliver new or previous file until cleanup; cleanup could be after global 
  consistency restored

how does delete file get propagated?
- delete from any site with a copy
- delete by File UUID so it is decoupled from put new file with same name
- negotiate with global to find a copy to delete, delete one or more copies OK?

how would a temporary cache instance at a site be maintained?
- site could accept writes to a "cache" instance for File(s) that do not belong
  in it's local "files" resource
- site could delete files once global has other Location(s)
- files could sit there orphaned if no one wants them

how does global inventory validate vs site?  how does site validate vs global (w.r.t. policy)?
- when it merges File(s) from sites with new Location(s) the metaChecksum 
  of the File in global will change: global File.metaChecksum != site File.metaChecksum
- if you query for File+Location where location.id = $site.id you could recompute 
  the metaChecksum of the site... 

what happens when a storage site detects that a File.contentChecksum != storage.checksum?
- if copies-in-global: delete the File and the stored file and re-sync w.r.t policy?
- if !copies-in-global: mark it bad && human intervention

