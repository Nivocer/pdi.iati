Handle IATI files
=================

Overview
--------

The `IATI Registry`_ provides metadata about available IATI data files: URI,
license, publisher, etc. At the moment, we're interested in the data files
themselves: we want to retrieve an up-to-date snapshot of all data files by
all publishers.

.. _`IATI Registry`: http://iatiregistry.org

.. uml::
   @startuml

   package "IATI Data Snapshot" {
      interface "IATI\nRegistry" as reg
      [Retrieve\nIATI files] as getfiles
      [IATI files\ngit repository] as repo
      
      reg - getfiles
      getfiles -> repo
   }
   
   [Local\ngit repository] as clone
   repo --> clone

   [Check changed\nIATI files] as queue
   database "Data\nWarehouse" as dw {
      [IATI files\nmetadata] as files_table
   }
   
   clone - queue
   queue - files_table

   @enduml

Retrieve IATI files
-------------------

We use the `IATI Data Snapshot repository`_ which makes IATI files available
as a git repository on Github. We simply clone this repository.

From our local repository, we can generate a list of files that have changed. 
We can then use this list to update metadata about those files in our data 
warehouse.

.. _`IATI Data Snapshot repository`: https://github.com/idsdata/IATI-Data-Snapshot


Use git for file changes
------------------------

We use the commit hashes in the git repository to record the version of the 
files we use. The commit hash is a unique identifier for a repository state, 
including all historical versions.

The first time files are processed, they all need to be included (excluding
anything in the .git directory, and the README.md file)::

    find . -path "./.git" -prune -o -type f ! -name README.md -print

On subsequent updates, we can use the commit hash of the last snapshot 
processed, and let git create a list of all files that have changed::

    git diff --name-status LASTHASH..CURRENTHASH


Check changed IATI files
------------------------

Before processing the content of the files, we first do a few checks.

.. uml::
    @startuml

    start
   
    :get the list of file names
    and process each file;
    

    if (file exists?) then (yes)
        if (file is\nvalid XML) then (yes)
        
            if (has IATI-activities) then (yes)
            
            :determine\nIATI version;
            if (validates?) then (yes)
                #ccffcc:valid
                activities
                file;
            else (no)
                #ffcccc:invalid
                activities
                file;
            endif
            
            else (no)
                if (has IATI-organisations) then (yes)

                    :determine\nIATI version;
                    if (validates?) then (yes)
                        #ccffcc:valid
                        organisation
                        file;
                    else (no)
                        #ffcccc:invalid
                        organisation
                        file;
                    endif
                
                else (no)
                    #ffcccc:unknown
                    XML;
                endif
            
            endif
        
        else
            #ffcccc:invalid
            XML;
        endif
        
    else (no)
        #ccffcc:deleted
        file;
    endif

    :add/update file
    metadata as
    <b>unprocessed;
    stop

    @enduml

IATI files metadata
-------------------

Our data warehouse contains a table for the IATI files:

.. uml::
   @startuml
    
   class IATI_file {
      filename: varchar[255]
      commit hash: char[40] <b>[HV]
      
      file_type: {activities, organisation, unknown}
      IATI reported version
      validation_output: text <b>[HV]
      is_valid: boolean
      is_processed: boolean

      date_from(): timestamp
      date_to(): timestamp
      is_current(): boolean
   }
   
   class organisation {
   }
   
   class IATI_version {
   }

   class date{
      date
      month
      quarter
      year
      fiscal year
   }
   
   IATI_file -> organisation: publisher <b>[HV]
   IATI_file -left-> date: updated <b>[HV]
   IATI_file -down-> IATI_version: IATI verified version <b>[HV]
   
   @enduml

Concerns, caveats and future updates
------------------------------------

This part of the system is based on the assumption there is a git repository
of IATI files. It should not make assumptions about the actual repository used.

*  If the IATI Data Snapshot stops working, we'll need to run the update
   scripts ourselves.

*  Initialising the data warehouse from the current version of the snapshot 
   should result in the same state of "current" information as processing each
   commit version. Obviously, the "as-was" versions may differ.

*  Most files will be IATI activities files, so checking for those first will
   reduce the amount of processing done with the organisations standard.