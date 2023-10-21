

Disclaimer - this is not official Red Hat documentation


# Introduction

The objective of this repository is to demonstrate the usage of a Helm chart in ArgoCD and to automate the secrets management in ArgoCD using the Secret Storage CSI Driver (SSCSI) in ARO.

## This repo is using the following tecnologies:
- GitOps/ArgoCD
- Helm
- SSCSI
- Azure KeyVault
-K8s


# Requirements
- Azure KeyVault (azkv) service will be used as the container of the secrets to be mounted in the pods. 
- SSCSI will retrieve the secrets from azkv and mount it in the pods. 
- SSCSI will have to be authenticated against the azkv service, for that a service principal will be used. A SP will have a key that must be protected, therefore this repo uses Bitnamy Sealed Secrets to encript the SP key, which will allow the encrypted key to be added to a Git repository.
- SSCSI by default mounts the retrived azkv secrets retrived as a volume in the pods, nonetheless we have an additional requirement to mount as a K8s secret as well (details bellow). 


# Procedure:

## Overview Steps

0.Clone the Git repository

1.Install GitOps

2.Install SSCSI in ARO

3.Install Bitnami Sealed Secrets in ARO

4.Exercise: Developer creates a new application in GitOps:
4.1.Create the Azure Service Principal (SP)
4.2.Encrypt the SP key
4.3.The encripted SP Key is uploaded to Git
4.4.The developer creates a new Application in GitOps feeding the Git link of the application Helm chart  


## Detailed Procedure

0.Clone the Git repository into your laptop
git clone https://github.com/CSA-RH/helm-gitops-sscsi_secrets.git

1.Install GitOps in your ARO cluster
https://docs.openshift.com/gitops/1.10/installing_gitops/installing-openshift-gitops.html

2.Install SSCSI in ARO

2.1.Install procedure
I have installed using the community version.
- Red Hat Experts version
https://cloud.redhat.com/experts/misc/secrets-store-csi/
https://cloud.redhat.com/experts/misc/secrets-store-csi/azure-key-vault/

- Microsoft version
https://learn.microsoft.com/en-us/azure/openshift/howto-use-key-vault-secrets

2.2.The following additional steps are required if the SSCSI have to create an K8s secret:
-add cluster role
oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:k8s-secrets-store-csi:secrets-store-csi-driver

- create custom SCC
This custom SCC is required to create the K8s secret and is allowing the use of the CSI volume. Additionally also set the capability "SETGID" as it is required by the MySQL DDBB Image used.

oc apply -f clusterprimer/scc.yaml


3.Install Bitnami Sealed Secrets in ARO

Bitnami Sealed Secrets team has developed a Helm Chart for installing the solution automatically. This automatism is customizable with multiple variables depending on the client requirements.

It is important to bear in mind that The kubeseal utility uses asymmetric crypto to encrypt secrets that only the controller can decrypt. Please visit the following [link](https://github.com/bitnami-labs/sealed-secrets/blob/main/docs/developer/crypto.md) for more information about security protocols and cryptographic tools used.

In the following process, a Sealed Secrets controller will be installed using a custom certificate that was generated in the respective namespace previously. This installation model is designed for multi-cluster environments where it is required to use the same certificate in multiple Kubernetes clusters in order to facilitate operations and maintainability.


- Create Namespace where Sealed Secrets Controller will be deployed

```$bash
oc new-project sealedsecrets
```

- Assign permissions to the default service account in order to be able to deploy the respective controller pod

```$bash
oc adm policy add-scc-to-user anyuid -z sealed-secrets -n sealedsecrets
```

- Generate the respective certificates and create a secret

```$bash
sh scripts/generate-cert.sh
```

- Deploy Sealed Secrets using the respective Helm Chart and the secret generated by the previous script execution

```$bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets -n sealedsecrets --set-string secretName=cert-encryption sealed-secrets/sealed-secrets
```

For further details refer to:
https://github.com/bitnami-labs/sealed-secrets#installation
https://github.com/acidonper/ocp-gitops-argocd-with-sealed-secrets/tree/master

- Install the command line tool kubeseal
https://github.com/bitnami-labs/sealed-secrets/releases

on Linux:

```$bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.2/kubeseal-0.24.2-linux-amd64.tar.gz -O kubeseal.tar.gz 
tar xzf kubeseal.tar.gz
rm kubeseal.tar.gz
chmod 755 kubeseal 
sudo mv kubeseal /usr/local/bin/
```


# Demo: Developer creates a new application in GitOps:

## Export Enviremont variables in your local laptop
export KEYVAULT_RESOURCE_GROUP=lmartinh-rgb6
export KEYVAULT_NAME=lmartinh-vault-1
export KEYVAULT_LOCATION=eastus
export NAMESPACE=petclinic3
export SECRET_NAME=demosecret4
export KEYVAULT_VALUE=petclinic

#To allow access to Azure Key Vault 
export AZK_SECRET_NAME=secrets-store-creds

## Create the Azure Service Principal (SP)

- Create an Azure Keyvault in your Resource Group that contains ARO


az keyvault create -n ${KEYVAULT_NAME} \
      -g ${KEYVAULT_RESOURCE_GROUP} \
      --location ${KEYVAULT_LOCATION}

- Create a secret in the Keyvault


az keyvault secret set \
      --vault-name ${KEYVAULT_NAME} \
      --name ${SECRET_NAME} --value ${KEYVAULT_VALUE}


- Create a Service Principal to access the keyvault

export SERVICE_PRINCIPAL_CLIENT_SECRET="$(az ad sp create-for-rbac \
      --name http://$KEYVAULT_NAME --query 'password' -otsv)"

export SERVICE_PRINCIPAL_CLIENT_ID="$(az ad sp list \
      --display-name http://$KEYVAULT_NAME --query '[0].appId' -otsv)"

echo "SERVICE_PRINCIPAL_CLIENT_SECRET"
echo $SERVICE_PRINCIPAL_CLIENT_SECRET
echo "==========="
echo "SERVICE_PRINCIPAL_CLIENT_ID"
echo $SERVICE_PRINCIPAL_CLIENT_ID


- Set an Access Policy for the Service Principal
az keyvault set-policy -n ${KEYVAULT_NAME} \
      --secret-permissions get \
      --spn ${SERVICE_PRINCIPAL_CLIENT_ID}


## Create SealedSecret Object
This object will contain the encrypted key used in the service principal to allow authorization to the Azure Key Vault.

- Locally create a secret for Kubernetes to use to access the Key Vault and label it.
#echo -n bar | oc -n ${NAMESPACE_PROJECT} create secret generic ${AKV_SECRET_NAME} --dry-run=client --from-file=foo=/dev/stdin -o yaml > sealedsecret_azv.yaml

```$bash
oc create secret generic secrets-store-creds --dry-run=client -o yaml \
      -n $NAMESPACE \
      --from-literal clientid=${SERVICE_PRINCIPAL_CLIENT_ID} \
      --from-literal clientsecret=${SERVICE_PRINCIPAL_CLIENT_SECRET} > sealedsecret_azv.yaml

???? Add a label to the secret oc -n $NAMESPACE label secret secrets-store-creds --dry-run=client -o yaml \
      secrets-store.csi.k8s.io/used=true

cat sealedsecret_azv.yaml

apiVersion: v1
data:
  clientid: "??????"
  clientsecret: "?????????"
kind: Secret
metadata:
  labels:
    secrets-store.csi.k8s.io/used: "true"
  name: secrets-store-creds
  namespace: deleteme
type: Opaque
```

- locally create the K8s manifest of the Sealed Secret wich embeds the encryption of the secret to have access to the AZV.

kubeseal -f sealedsecret_azv.yaml -n ${NAMESPACE} --name ${AZK_SECRET_NAME} \
 --controller-namespace=sealedsecrets \
 --controller-name=sealed-secrets \
 --format yaml > helm/pet-clinic/templates/sealed-secret.yaml


> **NOTE**
> controller-namespace: define the namespace where the operator is installed, 
> controller-name: is a combination of the SealedSecretController object name and the name of the namespace
?????> -n: is the namespace of the application consuming the secret  




- Create namespace

oc new-project ${NAMESPACE}


????- Is this clusterrole really needed?????? 
Regarding the steps to configure the final namespace to host the respective *SealedSecret* objects and the respective ArgoCD application that handles the creation of the secret, once the Red Hat Openshift GitOps operator is installed, are included in the following procedure:

```$bash
cat <<EOF > ClusterRole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: admin-sealedsecret
rules:
- apiGroups:
  - bitnami.com
  resources:
  - sealedsecrets
  verbs:
  - create
  - delete
  - deletecollection
  - get
  - list
  - patch
  - update
  - watch
EOF

oc apply -f ClusterRole.yaml -n {$NAMESPACE}

oc adm policy add-role-to-user admin-sealedsecret system:serviceaccount:openshift-gitops:openshift-gitops-argocd-application-controller -n {$NAMESPACE}
```

```$bash
oc label namespace {$NAMESPACE} argocd.argoproj.io/managed-by=openshift-gitops
```

- Before creating the application it is necessary to make a commit and push to the forked repository. 

```$bash
git add .
git commit -m "argocd sealed secrets"
git push
```

- Deploy the application in GitOps

oc apply -f argocd/application.yaml -n openshift-gitops


#echo "# helm-gitops-sscsi_secrets" >> README.md
#git init
#git add README.md
#git commit -m "first commit"
#git branch -M main
#git remote add origin https://github.com/CSA-RH/helm-gitops-sscsi_secrets.git
#git push -u origin main
