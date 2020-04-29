# Asynchronous job pattern

## Introduction

If triggering an external job (eg an Amazon EMR job) from Argo that does not run to completion in a container, there are two options:

- create a container that polls the external job completion status
- combine a trigger step that starts the job with a `Suspend` step that is unsuspended by an API call to Argo when the external job is complete.

This document describes the second option in more detail.

## The pattern

The pattern involves two steps, where the first step is a short running step that triggers a long-running job outside Argo (eg an HTTP submission), and the next step is a `Suspend` step that is either stopped or resumed via a call to the Argo API when the job outside Argo fails or succeeds.

When implemented as a `WorkflowTemplate` it can look something like this:

```
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: external-job-template
spec:
  templates:
  - name: run-external-job
    inputs:
      parameters:
        - name: "job-cmd"
    steps:
      - - name: trigger-job
          template: trigger-job
          arguments:
            parameters:
              - name: "job-cmd"
                value: "{{inputs.parameters.job-cmd}}"
      - - name: wait-completion
          template: wait-completion
          arguments:
            parameters:
              - name: uuid
                value: "{{steps.trigger-job.outputs.result}}"

  - name: trigger-job
    inputs:
      parameters:
        - name: "job-cmd"
          value: "{{inputs.parameters.job-cmd}}"
      image: appropriate/curl:latest
      command: ["/bin/sh", "-c"]
      args: ["{{inputs.parameters.cmd}}"]

  - name: wait-completion
    inputs:
      parameters:
        - name: uuid
    suspend: {}
```

In this case the ```job-cmd``` parameter can be a command that makes an http call via curl to an endpoint that returns a job uuid. More sophisticated submission and parsing of submission output could be done with something like a Python script step.

On job completion the external job would need to call either resume if successful:

```
curl --request PUT \
  --url http://localhost:2746/api/v1/workflows/<NAMESPACE>/<WORKFLOWNAME>/resume
  --header 'content-type: application/json' \
  --data '{
      "namespace": "<NAMESPACE>",
      "name": "<WORKFLOWNAME>",
      "nodeFieldSelector": "inputs.parameters.uuid.value=<UUID>"
    }'  
  ```

or stop if unsuccessful:

```
curl --request PUT \
  --url http://localhost:2746/api/v1/workflows/<NAMESPACE>/<WORKFLOWNAME>/stop
  --header 'content-type: application/json' \
  --data '{
      "namespace": "<NAMESPACE>",
      "name": "<WORKFLOWNAME>",
      "nodeFieldSelector": "inputs.parameters.uuid.value=<UUID>",
      "message": "<FAILURE-MESSAGE>"
    }'  
  ```