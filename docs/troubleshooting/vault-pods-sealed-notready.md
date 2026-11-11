# Vault Pods NotReady: Sealed State

**Technology:** HashiCorp Vault, Kubernetes

**Date:** 2026-08-04

## Problem

The Vault pods `vault-0`, `vault-1`, and `vault-2` were `Running` but not ready (`0/1`).

The readiness probes failed because Vault was initialized but sealed.

Vault was using Raft storage and Shamir seal.

## Fix

- Check the Vault pods:

   ```bash
   kubectl get pods -n vault
   ```

- Check a pod to find the problem:

   ```bash
   kubectl describe pod -n vault vault-0
   ```

- Check the Vault status:

   ```bash
   kubectl exec -n vault vault-0 -- vault status
   ```

   The status showed:

   ```text
   Initialized: true
   Sealed: true
   Seal Type: shamir
   Threshold: 1
   ```

- Unseal the Vault pod using the original unseal key:

   ```bash
   kubectl exec -it -n vault <vault-pod> -- vault operator unseal
   ```

   Enter the unseal key when requested.

- Repeat the unseal operation for each sealed Vault pod.

## Result

Vault was successfully unsealed.

All Vault pods became ready (`1/1`).

The readiness probes no longer failed because of the sealed state.
