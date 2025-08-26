---
title: "Provisioning Machines without PXE or Compute Resources on OpenStack via Satellite"
slug: "Provisioning Machines without PXE or Compute Resources on OpenStack via Satellite"
date: "2024-01-01 00:00:00+0000"
weight: 1
tags: 
   - provisioning
   - openstack
   - satellite
categories: 
   - Red Hat
---

# Creating Machines without PXE or Compute Resources on OpenStack via Satellite

## Introduction

In my daily life as a sysadmin, I spend a lot of time in my lab, which runs exclusively on OpenStack, and which I can rebuild at will thanks to Ansible (I feel an article about this coming soon). Recently, I faced an interesting challenge: testing PXE-less provisioning with Satellite without using compute resources. Like any self-respecting system engineer, I preferred to test this on my existing lab rather than create an environment more suited to this need, even though, let's be honest, it would probably have saved me time in the end.

I thought it would be interesting to share this adventure, not only because it piqued my curiosity and took up a lot of my time, but also because this provisioning method is quite underused with Satellite.

## Environment Description

As I like to be up to date in my articles, I specify the versions of the technologies used, even if some things are beyond my control (thanks to the endless updates).

- **Red Hat OpenStack 17.1**
- **Red Hat Satellite 6.15** (on RHEL 8 latest)
- **Creation of a RHEL 9.4 server**

## Procedure

### Satellite Preparation

If you're starting with a fresh Satellite installation, there are some essential steps you shouldn't miss. If your environment is already configured, some elements might already be in place.

- **Enable HTTPBoot on Satellite**

  ```bash
  satellite-installer --foreman-proxy-http true --foreman-proxy-httpboot true
  ```

  A little reminder: unspecified options here remain unchanged, so there's no need to list everything that has been modified since the cluster was created.

- **Products**

  You must have the BaseOS and AppStream repositories, as well as the corresponding kickstart repositories.

  In *Content -> Red Hat Repositories*, activate them, then in *Content -> Product*, synchronize them (a sync plan will make things even smoother).

- **Content Views (CV)**

  For each OS, you will need a CV or a CCV (Composite Content View) containing the 4 repos and published on the appropriate lifecycles.

- **Domain and Subnet**

  This step is crucial: for everything to work, you need to associate an IP address and a domain with your future host. Make sure these elements are well configured.

  For the domain: *Infrastructure -> Domains*, then *Create Domain*. Specify the DNS domain name and don’t forget to assign it to the necessary "locations" and "organizations".

  For the subnet: *Infrastructure -> Subnets*, then *Create Subnet*. Fill in the required fields and don't forget to add a Primary DNS server (it will be used to resolve the name of the Satellite and/or capsule), Boot Mode "Static".

- **Provisioning Template**

  This is also an important step. You can use your own provisioning templates, or start with the default template (kickstart default), but make sure the "reboot" command is not present.

- **Host Group**

  To create or modify a Host Group: *Configure -> Host Groups*.

  In the "Operating System" and "Activation Keys" tabs, I'll go into detail; for the rest, follow usual practices.

  In *Operating System*, choose the OS corresponding to the synchronized kickstart repo version, keep the media on Synced Content, and check that it is using the correct BaseOS kickstart repo for the target version. Also, consider specifying the root password if you're using a generic one, otherwise, don’t forget to create it in a subsequent step.

  In *Activation Keys*, make sure to specify the one that will allow the association to the CV/CCV.

- **Operating System (OS)**

  Ensure that the OS you want to use is correctly associated with the right provisioning template.

  To do this, go to *Hosts -> Provisioning Setup -> Operating System*, choose the desired OS, and in the "Templates" tab, select the correct template at the "Provisioning Template" line (the one specified earlier).

### OpenStack (OSP) Preparation

Rest assured, the hardest part is done. On the OSP side, there are also some preparations to make.

I assume you already have what you need in terms of project (network, subnet, flavor, etc.).

- **Preparing a Network Port**

  Since Satellite requires specifying a MAC address when creating a host, we will prepare a port and assign it an IP address.

  ```bash
  openstack port create --network <network name> --fixed-ip subnet=<subnet name>,ip-address=<chosen ipv4> <interface name>
  ```

  Be sure to retrieve the MAC address of the newly created network card:

  ```bash
  openstack port show <interface name> | grep "mac_address"
  ```

### Host Provisioning

Finally, the long-awaited step!

- **Creating a Host on Satellite**

  Create a new host on Satellite:

  *Hosts -> Create Host*

  Specify the following elements:

  - **Name**: the hostname of the future server
  - **Organization & Location**
  - **Host Group**: the one specified earlier (this will automatically fill in the rest of the fields in this tab and the "Operating System" tab)
  - In *Interface*, click *Edit* on the existing interface, then:
    - Enter the MAC address retrieved earlier
    - Specify the domain and subnet
    - Enter the IPv4 address used to create the port on OpenStack

  The host is now created on the Satellite side, and it should be in "Pending Installation" status. This status is what allows us to retrieve the key step of this provisioning: the Full Host Image.

  To do this, on the host's page (if you haven’t succumbed to the temptation to switch tabs), click on the three little dots in the top right, then on "Full host '<hostname>' image".

  You will then download an ISO image containing all the necessary information to build this host. Don’t worry, this is actually just a "discover foreman" image with overloaded information such as the name, IP address, and a kickstart file to retrieve. The weight of this image shouldn’t exceed a hundred MB.

- **Importing the Image on OpenStack**

  Now that you have this ISO, on OpenStack, you can create the associated image:

  ```bash
  openstack image create --disk-format iso --file <image name>
  ```

- **Starting the Provisioning**

  Everything is ready, all that's left is to start the installation of the instance. Since we are going a bit against OSP's basic principles, here’s how to proceed:

  1. Create an instance that will boot from the image, to which you will associate a non-temporary volume.
  2. Once the installation is complete (waiting for reboot), delete this instance (the volume will not be deleted).
  3. Create a new instance that will boot from the freshly installed volume.

And that's it! You now have an instance provisioned by your Satellite.

## Conclusion

In a reproducibility approach, I was able to use Satellite's PXE-less provisioning system by leveraging "Full Host Images". Once the preparation part is done, we can even consider automating the host creation via an Ansible playbook, taking in the key information such as OS, hostname, IPv4, etc.