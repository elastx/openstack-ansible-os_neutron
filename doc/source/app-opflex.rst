======================================
Scenario - Using OpFlex Neutron plugin
======================================

TODO
~~~~~~~~~~~~

Rewrite for Ocata

Introduction
~~~~~~~~~~~~

This document describes the steps required to deploy OpFlex Neutron plugin
with OpenStack-Ansible (OSA). These steps include:

- Configure OSA environment overrides.

- Configure OSA user variables.

- Execute the playbooks.

For additional documentation and configuration about OpFlex and its
architecture, please reference the `OpFlex <https://www.cisco.com/c/en/us/
solutions/collateral/data-center-virtualization/application-centric-
infrastructure/white-paper-c11-731302.html>`_ and `Cisco ACI with OpenStack
OpFlex Deployment Guide <https://www.cisco.com/c/en/us/td/docs/switches/
datacenter/aci/apic/sw/1-x/openstack/b_ACI_with_OpenStack_OpFlex_Deployment_
Guide_for_Ubuntu.html>`_ documentation.

Prerequisites
~~~~~~~~~~~~~

#. Cisco ACI fabric installed and initialized.

#. An existing repository server hosting Cisco OpFlex software.

#. All neutron and compute nodes must have the following bridges configured:

- ``br-mgmt``
- ``br-apic`` (used for communication with Cisco APIC)

Configure OSA Environment for OpFlex
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

OpFlex Neutron plugin utilize and depend on Open vSwitch Neutron plugin.
Copy the neutron environment overrides to
``/etc/openstack_deploy/env.d/neutron.yml`` to disable the creation of the
neutron l3 and metadata container(s), and implement the openvswitch and
opflex-ovs-agent hosts group containing all compute hosts.

.. code-block:: yaml

  component_skel:
    neutron_agent:
      belongs_to:
        - neutron_all
    neutron_metering_agent:
      belongs_to:
        - neutron_all
    neutron_server:
      belongs_to:
        - neutron_all
    neutron_opflex_agent:
      belongs_to:
        - neutron_all
    neutron_opflex_aim_agent:
      belongs_to:
        - neutron_all
    neutron_opflex_ovs_agent:
      belongs_to:
        - neutron_all

  container_skel:
    neutron_opflex_ovs_container:
      belongs_to:
        - compute_containers
      contains:
        - neutron_opflex_agent
      properties:
        is_metal: true
        service_name: neutron
    neutron_agents_container:
      belongs_to:
        - network_containers
      contains:
        - neutron_agent
        - neutron_metering_agent
      properties:
        service_name: neutron
    neutron_server_container:
      belongs_to:
        - network_containers
      contains:
        - neutron_server
        - neutron_opflex_aim_agent
      properties:
        service_name: neutron

  physical_skel:
    network_containers:
      belongs_to:
        - all_containers
    network_hosts:
      belongs_to:
        - hosts

For Unified mode assign neutron_opflex_aim_agent host group to
neutron_server and neutron_agent containers.

Configure OpFlex Neutron Plugin
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Set the following in ``/etc/openstack_deploy/openstack_user_config.yml``.

.. code-block:: yaml

  global_overrides:
    tunnel_bridge: "br-apic"
    provider_networks:
      - network:
          container_bridge: "br-apic"
          container_type: "veth"
          container_interface: "eth15"
          container_mtu: "1600"
          dhcp: true
          type: "raw"
          group_binds:
            - neutron_agent
            - neutron_opflex_agent
            - neutron_server
          static_routes:
            - cidr: 224.0.0.0/4
              gateway: 0.0.0.0

Set the following in ``/etc/openstack_deploy/user_variables.yml``.

.. code-block:: yaml

  openstack_host_specific_kernel_modules:
    - name: "openvswitch"
      pattern: "CONFIG_OPENVSWITCH="
      group: "network_hosts"

  neutron_plugin_type: ml2.opflex
  neutron_plugin_types:
    - ml2.ovs
  neutron_ml2_drivers_type: "opflex,local,flat,vlan,gre,vxlan"

  # Add cisco_apic_l3 to plugin base
  neutron_plugin_base:
    - apic_aim_l3
    - group_policy
    - metering
    - ncp

  # Override openvswitch config variables
  neutron_openvswitch_agent_ini_overrides:
    ovs:
      enable_tunneling: False
      integration_bridge: br-int
      tunnel_bridge:
      vxlan_udp_port:
      tunnel_types:

  neutron_provider_networks:
    network_types: "unified"

  neutron_ml2_conf_ini_overrides:
    ml2:
      extension_drivers: apic_aim
      mechanism_drivers: apic_aim


  opflex_apic_hosts:
    - 100.100.0.1
  opflex_apic_peer_svi: 100.100.0.30
  opflex_apic_remote_ip: 100.100.0.32
  opflex_apic_system_id: openstack
  opflex_apic_user: opflex
  opflex_apic_password: opflex

  # Repository server hosting Cisco OpFlex software
  opflex_apt_repo_url: https://repo.opflex.private
  opflex_repo:
    repo: "deb {{ opflex_apt_repo_url }} /ubuntu/"
    state: present


