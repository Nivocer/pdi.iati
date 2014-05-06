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

   [Check (changed)\nIATI files] as queue
   database "Data\nWarehouse" as dw {
      [IATI files\nmetadata] as files_table
   }
   
   clone - queue
   queue - files_table

   @enduml

Retrieve a local copy of the IATI files
---------------------------------------

We use the `IATI Data Snapshot repository`_ which makes IATI files available
as a git repository on Github. We simply clone this repository.

From our local repository, we can generate a list of files that have changed. 
We can then use this list to update metadata about those files in our data 
warehouse.

.. _`IATI Data Snapshot repository`: https://github.com/idsdata/IATI-Data-Snapshot


Generate a list of (changed) files
----------------------------------

We use the commit hashes in the git repository to record the version of the 
files we use. The commit hash is a unique identifier for a repository state, 
including all historical versions.

There are two paths to follow:

.. uml::
   @startuml
    
   start
    
   if (is a previous commit hash available\nand do we only process updated files?) then (yes)
      :Generate list of changed files;
      note: Based on git or the file system
      :Add fields for parts of filenames;
   else (no)
      :Generate list of all data files in the repository;
      :Combine with the list of current files;
   endif
   
   :Add current commit hash;
   
   stop
    
   @enduml
   
The result should be a list with relevant fields for all current or changed
files.

Based on git or the file system
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The first time files are processed, they all need to be included (excluding
anything in the .git directory, and the README.md file)::

    find . -path "*/.git" -prune -o -type f ! -name README.md -print
    
.. note::
   We use Pentaho's own component to list files in the repository.
   This transformation step offers information in different fields straight 
   away.

On subsequent updates, we can use the commit hash of the last snapshot 
processed, and let git create a list of all files that have changed::

    git diff --name-status LASTHASH..CURRENTHASH

In the end, we should end up with a list of filenames to check.

Check changed IATI files
------------------------

Before processing the content of the files, we first do a few checks.

.. uml::
    @startuml

    start
   
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

       :mark file as
       <b>unprocessed;
        
    else (no)
        #ccffcc:deleted
        file;
    endif

    :add/update file metadata;
    stop

    @enduml

IATI files metadata
-------------------

Our data warehouse contains a table for the IATI files:

.. uml::
   @startuml
    
   class IATI_file {
      -- file info --
      filename
      commit hash: char[40] <b>[HV]
      publisher <b>[HV]
      -- content --
      type: {activities, organisation, unknown}
      reported IATI version
      generated_at <b>[HV]
      is_processed: boolean
      -- validation --
      validated IATI version <b>[HV]
      validation_output: text <b>[HV]
      is_valid: boolean
      -- historical records --
      date_from: timestamp
      date_to: timestamp
      is_current: boolean
      version: int
   }
   
   class organisation {
      publisher code
      ...
   }
   
   class IATI_version {
      version
      is_official: boolean
      date_published
   }

   class date{
      date
      month
      quarter
      year
      ...
   }
   
   IATI_file -> organisation
   IATI_file -left-> date
   IATI_file -down-> IATI_version
   
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
