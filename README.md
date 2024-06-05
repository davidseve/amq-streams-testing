# AMQ Streams Testing with Argo CD

## Overview
This repository provides a setup for testing AMQ Streams (Kafka) on OpenShift with Argo CD integration. The testing includes the configuration of Kafka topics and the deployment of two distinct Kubernetes Jobs that are instantiated within the OpenShift cluster. The first Job is responsible for producing a message to the Kafka topic, while the second Job is tasked with consuming the message from the Kafka topic.

## Usage of Argo CD hook and sync-wave 

### argocd.argoproj.io/hook
Argo CD hooks are special Kubernetes resources that are executed at specific points in the synchronization lifecycle. They can be used for tasks such as pre-sync checks, post-sync operations, and manual approval gates. In this repository, hooks may be used to prepare the environment or clean up resources after synchronization.

### argocd.argoproj.io/sync-wave
The argocd.argoproj.io/sync-wave annotation allows you to control the order in which resources are synchronized. Resources with lower wave numbers are applied first. This can be critical for dependencies, ensuring that prerequisites are met before dependent resources are deployed. For example, Kafka topics must be created before producers or consumers are deployed.

### argocd.argoproj.io/hook-delete-policy
The argocd.argoproj.io/hook-delete-policy annotation specifies when a hook resource should be deleted. Options include HookSucceeded, HookFailed, and BeforeHookCreation. This helps in cleaning up resources that are no longer needed after the hook's execution.

For further details, please refer to the Argo CD documentation on [sync waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/) and [resource hooks](https://argo-cd.readthedocs.io/en/stable/user-guide/resource_hooks).

## Prerequisites
- OpenShift cluster

## Installation

### Install Operators
1. Apply the cluster configuration to install the required operators:
    ```bash
    oc apply -f ./cluster-configuration
    ```
2. Wait until both the OpenShift GitOps and AMQ Streams operators are installed.

### Obtain Argo CD Admin Password
Retrieve the Argo CD admin password using the following command:
```bash
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
```

### Create Argo CD application

Deploy the Argo CD application by applying the configuration:

```bash
oc apply -f ./argocd -n openshift-gitops
```

This Argo CD application will create:
- Namespace
- ServiceAccount
- RoleBinding
- Kafka instance

After the synchronization for testing purpose, using the PostSync Argo CD hook will create:
- Wave 0 (first wave)
  - KafkaUser
  - KafkaTopic
- Wave 10 (second wave)
  - Job producing a message
  - Job consuming a message 

### Example Configuration
Here are examples from the repository, demonstrating how these annotations are applied:

Kafka Producer Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
    argocd.argoproj.io/sync-wave: "10"
  name: test-kafka-producer
  namespace: {{ .Values.namespace }}  
spec:
  template:
    metadata:
      name: test-kafka-producer
    spec:
      restartPolicy: Never
      serviceAccountName: kafka-tests     
      containers:
      - name: test-kafka-producer
        image: registry.redhat.io/amq-streams/kafka-36-rhel8@sha256:8b10bfb697b48ba3ed246fb846384f4cc67f05f670e9521edf0f47f829869404
        command:
          - /bin/sh
          - '-c'
          - >-
            sleep 10 && echo "${message}" | /opt/kafka/bin/kafka-console-producer.sh --topic ${topic} --bootstrap-server ${cluster}-kafka-bootstrap:${port} --producer.config=/config/kafka-producer.properties  || exit 1
        env:
          - name: message
            value: {{ .Values.kafka.message }}
          - name: namespace
            value: {{ .Values.namespace }}  
          - name: cluster
            value: {{ .Values.kafka.cluster }} 
          - name: topic
            value: {{ .Values.kafka.topic }}     
          - name: port
            value: {{ .Values.kafka.port | quote }}
        volumeMounts:
          - name: kafka-props-tests
            readOnly: true
            mountPath: /config/kafka-producer.properties
            subPath: kafka-producer.properties
      volumes:
      - name: kafka-props-tests
        secret:
          secretName: kafka-props-tests
          defaultMode: 420           
  backoffLimit: 0
```


Kafka Consumer Job
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    argocd.argoproj.io/hook: PostSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
    argocd.argoproj.io/sync-wave: "10"
  name: test-kafka-consumer
  namespace: {{ .Values.namespace }}  
spec:
  template:
    metadata:
      name: test-kafka-consumer
    spec:
      restartPolicy: Never
      serviceAccountName: kafka-tests
      containers:
      - name: test-kafka-consumer
        image: registry.redhat.io/amq-streams/kafka-36-rhel8@sha256:8b10bfb697b48ba3ed246fb846384f4cc67f05f670e9521edf0f47f829869404
        command:
          - /bin/sh
          - '-c'
          - >-
            /opt/kafka/bin/kafka-console-consumer.sh --topic ${topic} --bootstrap-server ${cluster}-kafka-bootstrap:${port} --consumer.config=/config/kafka-consumer.properties --max-messages 1 | grep "${message}" || exit 1
        env:
          - name: message
            value: {{ .Values.kafka.message }}
          - name: namespace
            value: {{ .Values.namespace }}  
          - name: cluster
            value: {{ .Values.kafka.cluster }}                      
          - name: topic
            value: {{ .Values.kafka.topic }}      
          - name: port
            value: {{ .Values.kafka.port | quote }}       
        volumeMounts:
          - name: kafka-props-tests
            readOnly: true
            mountPath: /config/kafka-consumer.properties
            subPath: kafka-consumer.properties
      volumes:
      - name: kafka-props-tests
        secret:
          secretName: kafka-props-tests
          defaultMode: 420

  backoffLimit: 0
```