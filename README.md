# amq-streams-testing

## Installation
- Openshift GitOps
- AMQ Streams

```
oc apply -f ./cluster-configuration
```
Wait till both operators are installed


Get Argo CD admin password:
```
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
```

helm upgrade amq ./chart/ --install 
helm upgrade amq ./chart1/ --install 












## Installation

- Install AMQ Streams Operator

```$bash
oc apply -f cluster-configuration/amq-streams-subscription.yaml
```

- Deploy AMQ Streams Architecture (*kafka)

```$bash
oc apply -f cluster-configuration/amq-streams-configure.yaml
```

- Review the AMQ Streams installations

```$bash
oc get pod -n amq-streams-testing
NAME                      READY   STATUS    RESTARTS   AGE
amq-cluster-kafka-0       1/1     Running   0          11s
amq-cluster-kafka-1       1/1     Running   0          11s
amq-cluster-kafka-2       1/1     Running   0          11s
amq-cluster-zookeeper-0   1/1     Running   0          34s
amq-cluster-zookeeper-1   1/1     Running   0          34s
amq-cluster-zookeeper-2   1/1     Running   0          34s

```


- Create the Kafka topic in order to consume events automatically

```$bash
oc apply -f cluster-configuration/amq-kafkatopic.yaml
```

## Manual testing
- Run the following command to receive records from the kafka-topic topic.
```
oc -n amq-streams-testing  exec -it amq-cluster-kafka-0 -- bin/kafka-console-consumer.sh --topic kafka-topic --bootstrap-server amq-cluster-kafka-bootstrap:9092
```
Because the topic is empty, the output should be empty. Leave the terminal window open and the console consumer working.

- Open a new terminal window and start a producer for the call-detail-records topic. Enter a text after the input indicator >. Then, press Enter to send each line as a separate record.
```
oc -n amq-streams-testing  exec -it amq-cluster-kafka-0 -- bin/kafka-console-producer.sh --topic kafka-topic --bootstrap-server amq-cluster-kafka-bootstrap:9092
```
