================================
Installing and Configuring ARGUS
================================

Installation
############

The ARGUS documentation on read-the-docs is clear and concise.  This should be followed to install ARGUS, and can be found `here <http://argus-documentation.readthedocs.io/en/latest/misc/argus_deployment.html#argus-deployment>`_.

Configuration
#############

Grid Mappings
-------------

First, you need to create a user on the server for every local user / pool account that you need at your site for the VOs you wish to support.  Then ``touch`` a file in /etc/grid-security/gridmapdir for each user name.  


.. note::
   A list of approved VOs can be found here: https://www.gridpp.ac.uk/wiki/GridPP_approved_VOs for approved Vos
   

.. warning::
   If you want to replace / rebuild your server without taking your farm offline, you will need to copy the gridmapdir off of your old server before rebuilding and copy it back with a way that preserves the hardlinks.


   
You then need a /etc/grid-security/grid-mapfile. This contains a list of VOs, their roles and the usernames associated to those roles/VOs.

The layout is one section per each VO, following the examples below. ``<vo>`` should be replaced with the vo name from the approved vo list.  the ``<vo_specific_suffix>`` should be a suffix that matches the username you have created for the VO, and ``<vo_username>`` should be the  VO username created for the VO.

.. code-block:: bash

   "/<vo>/sgm" .sgm<vo_specific_suffix>
   "/<vo>/lcgprod/Role=NULL/Capability=NULL" .prd<vo_specific_suffix>
   "/<vo>/lcgprod" .prd<vo_specific_suffix>
   "/<vo>/Role=lcgadmin/Capability=NULL" .sgm<vo_specific_suffix>
   "/<vo>/Role=lcgadmin" .sgm<vo_specific_suffix>
   "/<vo>/Role=production/Capability=NULL" .prd<vo_specific_suffix>
   "/<vo>/Role=production" .prd<vo_specific_suffix>
   "/<vo>/Role=pilot/Capability=NULL" .pil<vo_specific_suffix>
   "/<vo>/Role=pilot" .pil<vo_specific_suffix>
   "/<vo>/Role=NULL/Capability=NULL" .<vo_username>
   "/<vo>" .<vo_username>
   "/<vo>/*/Role=NULL/Capability=NULL" .<vo_username>
   "/<vo>/*" .<vo_username>

Example:

.. code-block:: bash

   "/atlas/sgm" .sgmatl
   "/atlas/lcgprod/Role=NULL/Capability=NULL" .prdatl
   "/atlas/lcgprod" .prdatl
   "/atlas/Role=lcgadmin/Capability=NULL" .sgmatl
   "/atlas/Role=lcgadmin" .sgmatl
   "/atlas/Role=production/Capability=NULL" .prdatl
   "/atlas/Role=production" .prdatl
   "/atlas/Role=pilot/Capability=NULL" .pilatl
   "/atlas/Role=pilot" .pilatl
   "/atlas/Role=NULL/Capability=NULL" .atlas
   "/atlas" .atlas
   "/atlas/*/Role=NULL/Capability=NULL" .atlas
   "/atlas/*" .atlas

.. note::
   A fullstop ``.`` is not needed infront of the username if there is only one.  The fullstop indicates that this is a group of usernames, not a individual one.
   
Firewall
----------

ARGUS was designed to be spread across several servers, and accesses each daemon by using the fqdn (full qualified domain name) and the port.  As such the requests are “external” and ports need to be opened. 

Nothing to be accessed from outside world.
Need to be firewalled ports:

- 8150
- 8152
- 8154

These are accessed via the fqdn so need to be openned to the local network.  It may be possible to lock them to the local host?  

No firewall needed ports:

- 8150
- 8153
- 8151
  
All of these are bound to listen on the local host only so no firewall needed.


ARGUS Policy file
#################

The ARGUS policy file checks what is allowed to access the services.  There is only a need to do this at the VO level.  If you are happy for all services to allow and ban the exact same things, and for the whole cluster to be the same, then only one resource type is needed:

Example Policy file:

.. code-block:: bash

   resource "http://authz-interop.org/xacml/resource/resource-type/wn" {
       obligation "http://glite.org/xacml/obligation/local-environment-map" {}
        action "http://glite.org/xacml/action/execute" {
          rule permit {pfqan = "/atlas/Role=pilot" }
          rule permit {pfqan = "/atlas/Role=lcgadmin" }
          rule permit {pfqan = "/atlas/Role=production" }
          rule permit {pfqan = "/atlas" }

   resource "http://authz-interop.org/xacml/resource/resource-type/arc" {
       obligation "http://glite.org/xacml/obligation/local-environment-map" {}
        action ".*" {
          rule permit { vo = "ops" }
          rule permit { vo = "dteam" }
          rule permit { vo = "atlas" }


Then you need to create this policy file and apply it:
	  
.. code-block:: bash

   # run the pap-admin commands to apply policy file
   pap-admin apf <argus_policy_file>
   # load policy
   pap-admin lp
   # then restart the pap services:
   Systemctrl restart argus-pap
   Systemctrl restart argus-pdp
   Systemctrl restart argus-pepd
   

Central Banning
###############

Central banning is enabled using the pap admin commands.  These are found here: https://www.gridpp.ac.uk/wiki/Argus_Server#Configuring_Argus_for_Central_Banning.    

Additional notes here:
https://wiki.nikhef.nl/grid/Argus_Global_Banning_Setup_Overview
http://northgrid-tech.blogspot.com/2014/02/central-argus-banning-at-liverpool.html (from 2014!)

Testing
#######

It is easiest to test this from either ARC CE or DPM.  Perhaps a curl command could be constructed.

ARC CE Testing
----------------------

After configuring lcas and lcmaps on your ArcCE, Log onto the ArcCE and get a certificate.

.. code-block:: console

   $ voms-proxy-init --voms cms
   Enter GRID pass phrase for this identity:
   Contacting lcg-voms2.cern.ch:15002 [/DC=ch/DC=cern/OU=computers/CN=lcg-voms2.cern.ch] "cms"...
   Remote VOMS server contacted succesfully.
 
   Created proxy in /tmp/x509_proxy.
 
   Your proxy is valid until Fri Mar 16 04:08:21 GMT 2018
   $ voms-proxy-info
   subject   : /C=UK/O=eScience/OU=CLRC/L=RAL/CN=tweetie pie/CN=828088554
   issuer    : /C=UK/O=eScience/OU=CLRC/L=RAL/CN=tweetie pie
   identity  : /C=UK/O=eScience/OU=CLRC/L=RAL/CN=tweetie pie
   type      : RFC3820 compliant impersonation proxy
   strength  : 1024
   path      : /tmp/x509up_proxy
   timeleft  : 11:58:50
   key usage : Digital Signature, Key Encipherment, Data Encipherment 
   $ export X509_USER_PROXY=/tmp/x509up_proxy

arc-lcas can then be used to check the authorisation

.. code-block:: console
		
   $ /usr/libexec/arc/arc-lcas " /C=UK/O=eScience/OU=CLRC/L=RAL/CN=tweetie pie/CN=828088554" $X509_USER_PROXY liblcas.so /usr/lib64 /etc/lcas/lcas.db
   LCAS   2: LCAS authorization request
   LCAS   0: 2018-03-15.16:09:18 :     lcas_plugin_voms-plugin_confirm_authorization_from_x509(): voms plugin succeeded
   LCAS   1: lcas.mod-lcas_run_va(): succeeded
   LCAS   1: Termination LCAS
   

and arc-lcmaps can be used to get the mapping

.. code-block:: console

   $ sudo /usr/libexec/arc/arc-lcmaps "/C=UK/O=eScience/OU=CLRC/L=RAL/CN=tweetie pie/CN=828088554" $X509_USER_PROXY liblcmaps.so /usr/lib64 /etc/lcmaps/lcmaps.db voms
   LCMAPS has lcmaps_run
   LCMAPS has getCredentialData
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: Starting policy: get_account
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log: Cert at depth 3 is a root CA: "/C=UK/O=eScienceRoot/OU=Authority/CN=UK e-Science Root"
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    Key strength: 2048
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log: Cert at depth 2 is a CA: "/C=UK/O=eScienceCA/OU=Authority/CN=UK e-Science CA 2B"
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    signature algorithm: sha256WithRSAEncryption (=1.2.840.113549.1.1.11)
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    Key strength: 2048
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log: Cert at depth 1 is an EEC: "/C=UK/O=eScience/OU=CLRC/L=RAL/CN=tweetie pie"
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    signature algorithm: sha256WithRSAEncryption (=1.2.840.113549.1.1.11)
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    Key strength: 2048
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    CA hash: 530f7122, serial: C626
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    subjAltName rfc822Name: chris.brew@stfc.ac.uk
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    policy OID: 1.3.6.1.4.1.11439.1.1.1.2.2.0
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    policy OID: 1.2.840.113612.5.2.2.1
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    policy OID: 1.2.840.113612.5.2.3.3.3
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log: Cert at depth 0 (proxylevel 0) is a VOMS RFC3820 Proxy: "/C=UK/O=eScience/OU=CLRC/L=RAL/CN=tweetie pie/CN=828088554"
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    signature algorithm: sha256WithRSAEncryption (=1.2.840.113549.1.1.11)
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log:    Key strength: 1024
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log: The verification of the certificate has succeeded.
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: verify_log: Verification of chain without private key is OK
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: lcmaps_plugin_verify_proxy-plugin_run(): verify proxy plugin succeeded
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: lcmaps_plugin_c_pep-plugin_run(): Using endpoint https://argus.pp.rl.ac.uk:8154/authz, try #1
   lcmaps[3944231]    LOG_INFO: 2018-03-15.16:09:37Z: lcmaps_plugin_c_pep-plugin_run(): c_pep plugin succeeded
   lcmaps[3944231]  LOG_NOTICE: 2018-03-15.16:09:37Z: LCMAPS CRED FINAL: mapped uid:'<UID>',pgid:'<GID>'
   <poolaccount>:<poolgroudP>

DPM Testing
-----------------

If you have a DPM you can test manually running this command:

.. code-block:: console
		
   $ dpns-arguspoll TAG https://MY-ARGUS-SERVER:8154/authz

As described in:  https://www.gridpp.ac.uk/wiki/DPM_Argus_Integration





Log files
#########

There is a log directory: /var/log/argus.  This has three directories: pap, pepd, and pdp.   Each of these contains access.log, audit.log, and process.log

When a request comes in the following happens:

#. A request comes in and goes to pepd.  This is logged in the pepd/access.log
#. This is passed to pdp for a decision and an access request is logged in pdp/access.log
#. A log is made in pdp/process.log stating how decision was made.
#. A log is then made in pdp/audit.log recording what decision was made.
#. Passed back to pepd (no log).
#. Pepd runs stuff and records this in pepd/process.log.
#. Decision is recorded in pepd/audit.log

The decisions in pepd/audit.log are:
Allow, all was fine.
Deny, passed through the chain and a positive decision was made to deny.
Non-applicable, allowed, but no idea how to map.
Forth option.
