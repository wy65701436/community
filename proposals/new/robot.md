Proposal: Support Robot account in Harbor

Author: Yan Wang

## Abstract

Garbage Collection is a automaticlly way to delete unused image layers, then saving the disk usage.

## Motivation

In the current release, Harbor uses online garbage collection, which needs Harbor to run with setting to readonly mode.
During this time, any pushes are prohibited. 

## Solution

This proposal wants to try to introduce a way to enable non-blocking GC without setting Harbor to readonly.

### OCI Database

To facilitate non-blocking GC, Harbor builds up a OCI DataBase to track all the uploaded assets,

Before digging into details,

The components of each OCI artifact:

1, Configuration file
2, Layers
3, Manifest

#### OCI Database Data

Each artifact stored in Harbor is made up of several DB records, like library/hello-world:latest:

1, Repository Table -- library/hello-world
2, Artifact Table -- sha256:
3, Tag Table
4, Artifact&Blobs Table
5, Blobs Table
6, Project_Blobs Table

For OCI client starts to upload a artifact to finish, the above items are recorded in Harbor DB.

Base on the above data, what we knows:

    Indetifier of each blob/manifest.
    Reference count of each blob/manifest.

## Non-Blocking

As a system admin, you configure Harbor to run a garbage collection job on a fixed schedule. At the scheduled time, Harbor:

    Identifies and marks unused image layers.
    Deletes the marked image layers.

### Mark
Bases on the Harbor DB, we can count each blob/manifest's reference count, and selece the reference count 0 as the candidate.

####Question 1, hwo to deal with the uploading blobs at the phase of marking.
We do have a table to record the uploading blobs info, that's project & blob.
The delete candidate excludes all of blobs that in the project & blob.


### Sweep
The registry controller will grant the capability of deleting blob & manifest.
####Question 1, hwo to deal with the uploading blobs at the phase of sweeping.
Docker client will send a head request to ask the existence of the blob, we will intercept that request.


####Question 2, how to deal with the uploading "untagged manifest" of a index at the phase of sweeping.
We need to introduce the cutoff time, any to be deleted blob or manifest, the update time must not be later than the cutoff time.

### Delete Blob & Manifest
We'd like to enable the registry control to have the capability to delete blob & mainfest via digest by leverage the distribution code.

#### API

