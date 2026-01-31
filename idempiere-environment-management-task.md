# iDempiere Environment Management Task

The purpose of this task is to manage iDempiere environments using Incus virtualization.

## NixOS Containers

All iDempiere Incus containers run NixOS. Each environment is self-documented via its `/etc/nixos/configuration.nix` file.

For host-level Incus infrastructure (storage pools, networks, profiles), see incus-environment-management-task.

## Environments

**Pristine**
- Source of truth for tested configurations
- System-admin access only
- Receives configurations after testing is complete

**Test**
- Cloned from pristine
- Ephemeral - exists only to complete a configuration feature
- Destroyed after configuration is migrated to pristine

## Workflow

1. Clone pristine to create test environment
1. Develop and test configuration in test environment
1. Migrate tested configuration to pristine
1. Destroy test environment

Tags: #role-system-admin
