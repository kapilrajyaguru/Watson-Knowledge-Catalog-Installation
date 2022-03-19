# Watson-Knowledge-Catalog-Installation
This repo provides step-by-step instructions to install WKC on top of IBM Cloud Pak for Data running on Red Hat OpenShift cluster in Azure.

# Assumptions
 - IBM Cloud Pak for data Control Plane, Foundational Services are installed and running. 
 - IBM Cloud Pak for data operator are installed in “ibm-common-services” namespace and foundational services are installed in “cpd-instance” namespace. 
 - WKC operator will be installed in “ibm-common-services” namespace and WKC service will be installed in “cpd-instance” namespace. 
 - Red Hat OpenShift cluster has access to a high-speed internet connection and can pull images directly from IBM Entitled Registry.
 - Installing for demo purposes and so, the latest version of the software will automatically install on the Red Hat OpenShift cluster.
 - User has knowledge and experience managing Red Hat OpenShift cluster

# Pre-Requisite
 - Red Hat OpenShift cluster version 4.6 or later with min 64 vCPU and 256 GB RAM
 - Bastion host with 2 vCPU and 4GB RAM with Linux OS
 - Internet access for Bastion host and Red Hat OpenShift cluster
 - OpenShift Container Storage (OCS) attached to Red Hat OpenShift cluster. This link will help you determine supported storage. In this demo, I have used OCS Storage.
 - A User with OpenShift Cluster and Project Administrator access

# Enabling services to use namespace scoping with third-party operators
    oc patch NamespaceScope common-service \
    -n ibm-common-services \
    --type=merge \
    --patch='{"spec": {"csvInjector": {"enable": true} } }'

# Create the Watson Knowledge Catalog operator subscription.
 - The following command assumes that you plan to install WKC operator in ibm-common-services namespace.
      
        oc apply -f wkc-operator.yaml
 - Run the following command to confirm that the subscription was triggered:
    
        oc get sub -n ibm-common-services ibm-cpd-wkc-operator-catalog-subscription \
        -o jsonpath='{.status.installedCSV} {"\n"}'
    Verify that the command returns ibm-cpd-wkc.v1.0.6.
 - Run the following command to confirm that the cluster service version (CSV) is ready:
    
        oc get csv -n ibm-common-services ibm-cpd-wkc.v1.0.6 \
        -o jsonpath='{ .status.phase } : { .status.message} {"\n"}'
    Verify that the command returns Succeeded : install strategy completed with no errors.
 - Run the following command to confirm that the operator is ready:
        
        oc get deployments -n ibm-common-services -l olm.owner="ibm-cpd-wkc.v1.0.6" \
        -o jsonpath="{.items[0].status.availableReplicas} {'\n'}"
    Verify that the command returns an integer greater than or equal to 1. If the command returns 0, wait for the deployment to become available.
 # Run the following command to create an SCC named wkc-iis-scc.
   The command assumes that you plan to install Watson Knowledge Catalog in the cpd-instance project.
        
        oc apply -f wkc-iis-scc.yaml
 - Run the following command to verify that the SCC was created:

        oc get scc wkc-iis-scc
 - Create the SCC cluster role for wkc-iis-scc:
        
        
        oc create clusterrole system:openshift:scc:wkc-iis-scc \
        --verb=use \
        --resource=scc \
        --resource-name=wkc-iis-scc
 - Assign the wkc-iis-sa service account to the SCC cluster role.
    The command assumes that you plan to install Watson Knowledge Catalog in the cpd-instance project. If you plan to install Watson Knowledge Catalog in a different       project, replace cpd-instance in the serviceaccount parameter with the appropriate project for your environment.


        oc create rolebinding wkc-iis-scc-rb \
        --namespace cpd-instance \
        --clusterrole=system:openshift:scc:wkc-iis-scc \
        --serviceaccount=cpd-instance:wkc-iis-sa
    Replace cpd-instance with the name of the Red Hat OpenShift project where you plan to install Watson Knowledge Catalog.
 - Confirm that the wkc-iis-sa service account can use the wkc-iis-scc SCC:


         oc adm policy who-can use scc wkc-iis-scc \
         --namespace cpd-instance | grep "wkc-iis-sa"

# CRI-O Container Settings
 - Copy machineconfig.yaml to /tmp directory

        cp crio.conf /tmp/

 - Login to Red Hat openshift in command line. Use cloned machineconfig object YAML file, as follows, and apply it.
    Note: If you are using Cloud Pak for Data on OpenShift Container Platform version 4.6, the ignition version is 3.1.0. If you are using Cloud Pak for Data on           OpenShift Container Platform version 4.8, change the ignition version to 3.2.0.

        oc apply -f machineconfig.yaml
    The above action will reboot your cluster nodes one-by-one. Monitor all of the nodes to ensure that the changes are applied, by using the following command:
  
        watch oc get nodes
    You can also use the following command to confirm that the MachineConfig sync is complete:

        watch oc get mcp

# Kernel parameter settings
    Enabling unsafe sysctls
    Configure kubelet to allow Db2U to make unsafe sysctl calls for Db2 to manage required memory settings. 
 - Update all of the nodes to use a custom KubletConfig:

        oc apply -f kubeletconfig.yaml
 - Update the label on the machineconfigpool:

        oc label machineconfigpool worker db2u-kubelet=sysctl
 - Wait for the cluster to restart and then run the following command to verify that the machineconfigpool is updated:
        oc get machineconfigpool
    The command should return output with the following format:
    
        NAME     CONFIG   UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
        master   master   True      False      False      3              3                   3                     0                      139m
        worker   worker   False     True       False      5              1                   1                     0                      139m
    Wait until all of the worker nodes are updated and ready.

# Create the Db2U catalog source if you plan to install one of the following services:
 - Data Virtualization
 - Db2®
 - Db2 Big SQL
 - Db2 Warehouse
 - OpenPages® (required only if you want OpenPages to automatically provision a Db2 database)

 - Check whether the IBM Db2U Catalog already exists on your cluster:

        oc get catalogsource -n openshift-marketplace
    Review the output to determine whether there is an entry called ibm-db2uoperator-catalog.
 - If the IBM Db2U Catalog does not exist, create it:
  
        oc apply -f ibm-db2u-catalog.yaml
 - Verify that the IBM Db2U Catalog was successfully created:

        oc get catalogsource -n openshift-marketplace
    Review the output to ensure that there is an entry called ibm-db2uoperator-catalog.
 - Verify that ibm-db2uoperator-catalog is READY:

        oc get catalogsource -n openshift-marketplace ibm-db2uoperator-catalog \
        -o jsonpath='{.status.connectionState.lastObservedState} {"\n"}'

# Create a WKC custom resource to install Watson Knowledge Catalog. Follow the appropriate guidance for your environment.

        oc apply -f wkc.yaml
             
 - Get the status of Watson Knowledge Catalog. It might take two to three hours to install Watson Knowledge Catalog. You can check the status of Watson Knowledge Catalog by running the following command:

        oc get WKC wkc-cr -o jsonpath='{.status.wkcStatus} {"\n"}'
    While Watson Knowledge Catalog is installing, the status shows In-Progress.
    You can check the status of the modules by running the following commands. The modules are installed in the order shown, and you must wait for each module to show     Completed before the next module is installed.
 - To get the status of the common core services module, run the following command:

        oc get CCS ccs-cr -o jsonpath='{.status.ccsStatus} {"\n"}'
 - To get the status of the Data Refinery module, run the following command:

        oc get DataRefinery datarefinery-sample -o jsonpath='{.status.datarefineryStatus} {"\n"}'
 - To get the status of the Db2 as a service module, run the following command:

        oc get Db2aaserviceService db2aaservice-cr -o jsonpath='{.status.db2aaserviceStatus} {"\n"}'
 - To get the status of the InfoSphere® Information Server module, run the following command. If you are installing the core version of the service, this check does not apply.

        oc get IIS iis-cr -o jsonpath='{.status.iisStatus} {"\n"}'
 - To get the status of the Unified Governance module, run the following command. If you are installing the core version of the service, this check does not apply.

        oc get UG ug-cr -o jsonpath='{.status.ugStatus} {"\n"}'

