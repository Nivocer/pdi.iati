.. pdi.iati documentation master file
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Pentaho Data Integration for IATI files
=======================================

This repository provides scripts for the Pentaho_ data integration tools
to build a data store of information published in the :term:`IATI standard`.

.. _Pentaho: http://www.pentaho.com

Overview
--------

Our aim is to build a data warehouse based on data files published in the
:term:`IATI Standard`.

*  The `IATI Registry`_ provides links to all available data files with 
   metadata on each datafile.
   :doc:`Read more ></iati_files>`

*  We can then process each data file, to build the data warehouse.
   :doc:`Read more ></process_iati_data>`

*  To enrich the information available about organisations, we want to reconcile
   the data with the `Open Corporates`_ data set.

*  Since IATI data can contain financial information in many currencies, we also
   want to use the detailed conversion rates available from 
   `Open Exchange Rates`_.
  
*  We can then generate reports for different entities (and eventually provide
   more ways to access the data).
  

.. _`IATI Registry`: http://iatiregistry.org
.. _`Open Corporates`: http://opencorporates.com
.. _`Open Exchange Rates`: https://openexchangerates.org

.. uml::
   @startuml

   interface "IATI\nRegistry" as reg
   [Handle\nIATI files] as repo
   [Process\nIATI data] as pdi_iati
   database "Data\nWarehouse" as dw
   
   reg -right- repo
   repo -right- pdi_iati
   pdi_iati -down-> dw
   
   interface "Open Exchange\nRates" as curr
   [Update\ncurrency rates] as pdi_curr
   
   curr -right- pdi_curr
   pdi_curr -up-> dw
   
   interface "Open\nCorporates" as corp
   [Add data from\nOpen Corporates] as pdi_corp
   
   corp -right- pdi_corp
   pdi_corp -right-> dw

   [Generate\nReports] as reports
   dw -right-> reports
   reports -right- IATI.me

   @enduml
   
Table of Contents
-----------------

.. toctree::
   :maxdepth: 2
   
   iati_files
   process_iati_data
   
   glossary
   colophon
