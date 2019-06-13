# storage-dm
Storage System Data Model

This module contains a <a href="http://www.ivoa.net/documents/VODML/index.html">VO-DML</a> 
description of the Storage data model and test code to validate the model using the 
<a href="https://github.com/opencadc/core/tree/master/cadc-vodml">cadc-vodml</a> tools. The 
current version uses some experimental changes (compared to the IVOA proposed recommendation).

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">
<img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a>
<br />The Common Archive Observation Model is licensed under the
<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">
Creative Commons Attribution-ShareAlike 4.0 International License</a>.

Development starts with the UML diagrams and the current version may be ahead of the VO-DML and documentation. 
The current UML is <img alt="Storage System UML - development version" style="border-width:0" 
src="https://github.com/pdowler/storage/raw/master/storage-dm/src/main/resources/inventory.png" />


# Entity
Simplified version of the CaomEntity. An Entity is something 
that can be serialised and supports incremental harvesting 
and validation.

# Site
A storage site has a "sites" resource with a single entry
describing itself. A site can change it's resourceID and
name but must maintain a stable entity.id (UUID).

A global inventory service has an instance of "sites"
that it harvests from storage sites that it finds via a
registry query. This is a small and pretty static list but
sites may change resourceID so we do not want to
store it directly in the location.

# File.fileID
This URI is a globally unique identifier that is typically known 
to and may be defined by some other system (e.g. Artifact.uri 
in CAOM, or a VOSpace node URI from vault). 

- {scheme}:{path}
- {scheme}://{authority}/{path}
- no query string
- no fragment
-  the last path component is the "file name"

Basic archive usage is to use the cadc scheme: cadc:{namespace}/{path}. About namespace: denoting the first 
path component as special makes sense from an organizational and allocation point of view

For vault (vospace), data nodes would have a generated identifier and use cadc:vault/{datanode.uuid}.
This way paths within the vospace don't end up in storage. Validation requires querying vaultdb and 
building fileID(s).

For externally harvested CAOM artifacts (mast), we would use the URI as-is. Non-cadc scheme URIs would have to
be treated as different namespace(s); this usage implies that every external Artifact.uri scheme would require 
a new storage namespace.

Namespace concept seems pretty thin: maybe this is just configuration outside the model?

# storage site
A storage site tracks all files written to local storage. These files 
can be harvested by anyone: central inventory, clones, backups, etc. 
A storage site is fully consistent wrt local file operations (e.g. put + get + delete).

Storage sites:
- track writes of local files
- implement get/put/delete service API
- implement policy to harvest File(s) from global that 
  (local mirror policy)
- implement file sync from other sites using global
- implement validation of file metadata vs storage

A storage site always has 1:1 File->Location with only it's own
Location(s).

# global inventory
A global inventory service is built by harvesting sites, files, and deletions from all known sites.
A global inventory is eventually consistent.

A global inventory (may be one or more):
- harvests and merges metadata from storage sites
  where merge means "add location"
- implements transfer negotiation API

Rather than just accumulate new instances of File, the
harvested File may be a new File or an existing File with
a new Location that has to be merged into the global
inventory. Thus, File.metaChecksum and File.lastModified
in a global inventory are not equal to the values at
individual storage sites.

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

