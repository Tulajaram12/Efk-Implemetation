<img width="1334" height="283" alt="image" src="https://github.com/user-attachments/assets/02d1fedd-2493-44a5-b8b9-5541a370bd9d" /># Efk-Implemetation  
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

# Also do the following thing below
createCert: false

# Certificate generation manaully
mkdir certs
cd certs

openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/CN=elasticsearch-ca"
openssl genrsa -out tls.key 4096
openssl req -new -key tls.key -out tls.csr -subj "/CN=elasticsearch-master"

# Create a san.cf
cat > san.cf 
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = elasticsearch-master
DNS.3 = elasticsearch-master.elastic-system.svc.cluster.local
DNS.4 = *.elasticsearch-master
DNS.5 = *.elasticsearch-master.elastic-system.svc.cluster.local

# Creation of tls.crt
openssl x509 -req -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt -days 3650 -sha256 -extfile san.cf

# Create the secret
kubectl create secret generic elastic-certificates --from-file=tls.crt --from-file=tls.key --from-file=ca.crt -n elastic-system

# Add the secret Mounts
secretMounts: 
  - name: elastic-certificates
    secretName: elastic-certificates
    path: /usr/share/elasticsearch/config/certs
    defaultMode: 0755

# Add this Config Below Check certificate paths what folder u have created and then add
esConfig: 
  elasticsearch.yml: |
    xpack.security.enabled: true

    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.key: certs/tls.key
    xpack.security.transport.ssl.certificate: certs/tls.crt
    xpack.security.transport.ssl.certificate_authorities: certs/ca.crt

    xpack.security.http.ssl.enabled: true
    xpack.security.http.ssl.key: certs/tls.key
    xpack.security.http.ssl.certificate: certs/tls.crt
    xpack.security.http.ssl.certificate_authorities: certs/ca.crt


# Also Enable the 
helm install elasticsearch elastic/elasticsearch --version 8.5.1 -n elastic-system -f values.yaml




###  Kibana Installation
kubectl create namespace elastic-system
helm repo add elastic https://helm.elastic.co
helm repo update
helm show values elastic/kibana --version 8.5.1

# 1) Add the tolerations
tolerations:
  # - key: "workload"
  #  operator: "Equal"
  #  value: "elasticsearch"
  #  effect: "NoSchedule"

# 2) Add the nodeSelector
nodeSelector:
  workload: elasticsearch

# 3)Change the initial Delay Second
readinessProbe:
  failureThreshold: 3
  initialDelaySeconds: 60
  periodSeconds: 10
  successThreshold: 3
  timeoutSeconds: 5

# Even Change the resources as needed 
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "1000m"
    memory: "2Gi"

# if you have created manually the certificates then comment 
#elasticsearchCertificateSecret: elasticsearch-master-certs
#elasticsearchCertificateAuthoritiesFile: ca.crt

# if you want HA of kibana then change
replicas: 1

# Change protocol from http to https
protocol: https

# Add the CA Cert Mount
secretMounts:
  - name: elastic-certificates
    secretName: elastic-certificates
    path: /usr/share/kibana/config/certs


# Add this below
kibanaConfig:
  kibana.yml: |
    elasticsearch.ssl.certificateAuthorities: /usr/share/kibana/config/certs/ca.crt
    elasticsearch.ssl.verificationMode: certificate






















