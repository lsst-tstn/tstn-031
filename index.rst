:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. note::

   This technote describes the steps required to execute Integration Milestone pf (IMpf).
   Some of the steps described here are required to setup the environment and won't need to be repeated on the verification step.

This milestone consists of driving the MTAOS to process simulated Corner Rafts data, compute the associated corrections and applying those corrections to the AOS components (Camera Hexapod, M2 Hexapod, M1M3 and M2).
After the process is completed the results are inspected from a jupyter notebook.

It is important to emphasize that here we are more interested in verifying the control flow and not necessarily the results of the execution.
Validation of the values can be done as an additional exercice, and requires some knowledge of the details of both the `WEP <ts-wep.lsst.io>`__ and `OFC <ts-ofc.lsst.io>`__.

The process was executed at the Tucson Test Stand (TTS), using a mix of simulators (for the hardware components) and non-simulators (for pure software components).
In addition, a set of simulated LSSTCam data was provided


For more information about Integration Milestones (and IMpf) see `sitcomtn-006`_.

.. _environment-setup:

Environment Setup
=================

Before executing IMpf, there are some steps that need to be taken to prepare the environment.
In principle these steps need only be done once and there is no need to repeat them in order to reproduce the milestone.
These involves moslty gathering simulated data and catalogs, and ingesting them into a butler repository that is available to the MTAOS component.

The setup was done entirely from a nublado instance, using the ``/scratch`` directory for the data.
This directory is a nfs mount that is also available to the MTAOS.

The first step is to create a new butler repository in ``/scratch/IM_Pf/LSSTCam/``.
This is done with the following command sequence:

.. prompt:: bash

   # create directory
   mkdir -p /scratch/IM_Pf/LSSTCam
   # create butler registry
   butler create /scratch/IM_Pf/LSSTCam
   # register LsstCam
   butler register-instrument /scratch/IM_Pf/LSSTCam lsst.obs.lsst.LsstCam

Then download the data from NCSA:

.. prompt:: bash

   rsync -avz --progress lsst-login02.ncsa.illinois.edu:/project/scichris/aos/AOS/DM-28360/lsstCam/high/focal/9010021/DATA/LSSTCam/raw/all/raw/20211231 /scratch/IM_Pf/LSSTCam/

And ingested the data with:

.. prompt:: bash

   butler ingest-raws -t link /scratch/IM_Pf/LSSTCam /scratch/IM_Pf/LSSTCam/20211231/MC_H_20211231_010021/raw_LSSTCam_r_MC_H_20211231_010021_R*fits

Next we download the PanStarrs catalog with:

.. prompt:: bash

   rsync -avz --progress lsst-login02.ncsa.illinois.edu:/datasets/refcats/htm/v1/ps1_pv3_3pi_20170110 /scratch/refcats/refcats/htm/v1

In order to ingest the catalog to the butler, we need the catalog definition configuration file.

.. prompt:: bash

   rsync -avz --progress lsst-login02.ncsa.illinois.edu:/repo/main_20210215/butler.yaml /scratch/main/

And the import script:

.. prompt:: bash

   scp lsst-login02.ncsa.illinois.edu:/project/aos/data_repos/refcatImportScripts/ps1_export_gen3_lsst_devl.yaml /scratch/refcats/

The script needs to be modified so the paths matches those on the TTS.
This is done with the following commands, which replaces the original file with the modified version at the end.

.. prompt:: bash

   sed "s|/dat^Cets/|/scratch/refcats/|g" /scratch/refcats/ps1_export_gen3_lsst_devl.yaml >temp.sh
   mv temp.sh /scratch/refcats/ps1_export_gen3_lsst_devl.yaml

Finally, we are ready to import the catalog:

.. prompt:: bash

   butler import /scratch/IM_Pf/LSSTCam/ /scratch/main/ --export-file /scratch/refcats/ps1_export_gen3_lsst_devl.yaml --transfer link

As a final step, we need to make sure everyone has write access to the butler, so the MTAOS can create the data products:

.. prompt:: bash

   chmod a+rw -R /scratch/IM_Pf/LSSTCam/

With this final step, the environment is ready to execute the IMpf.

Executing IMpf
==============

.. note::

   Before executing IMpf on TTS, make sure to announce your intention in the ``rubinobs-tucson-teststand``.
   This test requires all MTCS components, though not all of them will be involved in the test directly.

This section describes in detail the steps in executing IMpf.
These are mainly reproduced in the `execute_impf`_ jupyter notebook available in the `ts_notebooks`_ repository.
An example execution of the notebook, rendered as a doc page, can be seen in :ref:`integration-milestone-pf`.

.. _ts_notebooks: https://github.com/lsst-ts/ts_notebooks
.. _execute_impf: https://github.com/lsst-ts/ts_notebooks/tree/develop/procedures/MT/IMPf/

As mentioned before, the idea of this milestone is to drive the MTAOS throught processing LSSTCam corner wavefront sensor data, and then apply the resulting corrections to the optical components.

Before starting the test we need to make sure the CSCs involved are all in ENABLED state, configured with the appropriate configuration and setup execute the required sequence of commands.

For M1M3, in additional to enabling it with the ``Default`` configuration, we also need to make sure the mirror is raised, the balance for system is enabled and that the forces are reset to the zero point.

For M2, we need to make sure the balance system is enabled and that the forces are reset to the zero point.

For M2 and Camera hexapods, we need to make sure compensation mode is enabled and the position is reset to the zero point.

Finally, we need to make sure the MTAOS is ENABLED with the appropriate configuration to execute the test, labeled ``impf``.
In addition to configuring the MTAOS to process LSSTCam data, this configuration points the MTAOS to the butler instance in ``/scratch/IM_Pf/LSSTCam/`` created in :ref:`environment-setup`.

All these steps are executed in the section :ref:`setting-up-the-system` of the notebook.

The next step is to command the MTAOS to process the corner wavefront sensors from the image we ingested in the butler.
In order to execute this step we need to provide the `visitId` of the exposure along with a configuration for the wavefront estimation pipeline (WEP).
This step is executed in the :ref:`processing-data-with-runwep-command` section of the notebook.
After the ``runWEP`` command is executed we also verify that the expected set of events were published.

After the corner wavefront sensors are processed, we instruct the MTAOS to process the results through the optical feedback control (OFC) algorithm.
This converts the wavefront erros estimated by the WEP into corrections for the optical components; force offsets for the M1M3 and M2 and motion offset for the Camera and H2 Hexapods.
This is done in the :ref:`run-ofc` section of the notebook.
After processing the data, the MTAOS publishes a set of events with the results, which is also verified in the notebook.

The final step in the process is to issue the correction to the optical components.
We do this by sending the command ``issueCorrection`` to the MTAOS.
This command will fail if, for some reason, the MTAOS is unable to apply the corrections to any of the components.
There are no events published associated with this command so, once it executes successfully, the test is completed.
This step is done in the :ref:`issue-correction` section of the notebook.

.. toctree::
    execute_impf/execute_impf
    :maxdepth: 1

Verifiying IMpf
===============

The page below is an exported version of the ``inspect_wep`` notebook.
This notebook display some of the data written to the butler registry by the WEP when the MTAOS executes the command ``runWEP``.

.. toctree::
    inspect_wep/inspect_wep
    :maxdepth: 1
      

.. _sitcomtn-006: https://sitcomtn-006.lsst.io 

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
