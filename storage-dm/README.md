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
This URI is a globally unique identifier that is typically known 
to and may be defined by some other system (e.g. an Artifact.uri 
in CAOM). 

{scheme}:{name}/{path}

ivo://{authority}/{name}?{path}

For URIs of the simple form, the namespace would be {scheme}:{name} and the filename would be the past component of {path}.

For resolvable ivo URIs, the resourceID can be extracted by dropping the query string. The resourceID 
can be found in a registry and allows clients to find data services. This form allows for generic tools to resolve
and sync files from external systems. Example usage of equivalent fileID values:

ad:{archive}/{filename} *classic*

cadc:{archive}/{filename} *new*

ivo://cadc.nrc.ca/data?{archive}/{filename} *resolvable data service?*

ivo://cadc.nrc.ca/data/{collection}?{archive}/{filename} *resolvable data collection?*

Basic archive usage is to use the cadc scheme: cadc:{name}/{path}. The namespace would be "cadc:{name}". Validation of 
CAOM versus storage requires querying CAOM (1+ collections) and storage (single namespace) and cross-matching URIs to 
look for anomolies. 

For externally harvested and sync'ed CAOM artifacts, we would use the URI as-is in the fileID. For the simple form 
(e.g. mast:HST/path/to/file.fits) the namespace would be "mast:HST". For the resolvable (ivo) form, it is not clear 
if the whole resourceID would be the namespace or if one might want a single namespace per {authority} in some or all 
cases. Validation of CAOM versus storage requires querying CAOM (1+ collections) and storage (single namespace) and 
cross-matching URIs to look for anomolies.

For vault (vospace), data nodes would have a generated identifier and use cadc:vault/{datanode.uuid}.
This way paths within the vospace don't end up in storage and move operations in vospace do not effect
the storage system. There should be no use of the "vos" scheme in a fileID. Validation of vault versus storage
requires querying vault (and building fileID values programmatically unless vaultdb stores the full URI) and storage
and cross-matching to look for anomolies.

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

