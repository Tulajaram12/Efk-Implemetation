# Efk-Implemetation  
kubectl create namespace elastic-system  
helm repo add elastic https://helm.elastic.co  
helm repo update  
helm show values elastic/elasticsearch   --version 8.5.1 > elasticsearchvalues.yaml

if you are planning for the separate Nodegroup for the elasticsearch
then add tolerations for the values.yaml

first label all the nodes by using the 
kubectl label nodes <node-name> workload=elasticsearch
  OR 
While launching the nodegroups itself add the labels for it
kubectl taint nodes nodename workload=elasticsearch:NoSchedule

You can design for HA by launching 3 different Nodegroups
in 3 different az i.e one nodegroup you launch in 1az and other nodegroup you launch in another az 

# 1) Add the tolerations
tolerations:
  # - key: "workload"
  #  operator: "Equal"
  #  value: "elasticsearch"
  #  effect: "NoSchedule"

# 2) Add the nodeSelector
nodeSelector:
  workload: elasticsearch

# 3)Change the VolumeSize in volumeClaimTemplate 
volumeClaimTemplate:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 30Gi

# Add the secret value here
secret:
  enabled: true
  password: "IGYoV1ftcEgOgtW3" # generated randomly if not defined

# Even Change the resources as needed 
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "1000m"
    memory: "2Gi"


# Change the initial Delay Second
readinessProbe:
  failureThreshold: 3
  initialDelaySeconds: 60
  periodSeconds: 10
  successThreshold: 3
  timeoutSeconds: 5
helm install elasticsearch elastic/elasticsearch --version 8.5.1 -n elastic-system -f values.yaml
