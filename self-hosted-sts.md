# AWS Security Token Service - Steps to Migrate the OIDC issuer from public S3 Bucket to a self-hosted NGINX host

Steps to migrate the OIDC identity provider URL from a public S3 Bucket to a private using NGINX running in OpenShift to provide the public OpenID Connect documents required in an OpenShift cluster deployed in manual mode with STS on AWS.

> NOTE: the steps described in this document are not supported or recommended to be done in a production environment.

- [Prerequisites](#prerequisites)
- [Validate and Backup](#validate-backup)
    - [Validate](#validate-tokens)
    - [Backup](#backup)
- [Migrate](#migrate)
    - [Setup new OIDC with NGINX](#setup)
        - [Create the NGINX Host](#nginx)
        - [Create and patch the new OIDC discovery documents and JWKS](#setup-oidc-documents)
        - [Create the new OIDC using CloudFront Distribution](#setup-oidc-idp)
    - [Patch the cluster to use the new OIDC](#patch-cluster)
    - [Revoke public access to S3 Bucket](#revoke-s3-public-access)
- [Rollback to OIDC with S3 Public URL](#rollback)
- [Delete the Old OIDC identity provider](#delete)

## Prerequisites <a name="prerequisites"></a>

1) An OCP Cluster created on AWS with manual mode with STS using S3 Bucket as OIDC URL (issuerURL)

> **NOTE**: The steps described in this document was tested and is validated to patch IAM Roles created by **cluster components**. **You must patch the IAM Role Trusted Policies [created to user-workload by pod identity webhook steps](https://docs.openshift.com/rosa/authentication/assuming-an-aws-iam-role-for-a-service-account.html).

2) Export the environment variables below - required to be logged on OCP:
```bash
# Discovery the `--name` provided when creating resources by CCO
export OIDC_BUCKET_HOST=$(basename $(oc get authentication cluster -o jsonpath={'.spec.serviceAccountIssuer'} ))
export OIDC_ARN_S3=$(aws iam list-open-id-connect-providers | jq -r ".OpenIDConnectProviderList[] | select(.Arn | endswith(\"$OIDC_BUCKET_HOST\")).Arn")
export CLUSTER_NAME=$(aws iam list-open-id-connect-provider-tags --open-id-connect-provider-arn $OIDC_ARN_S3 | jq -r '.Tags[] | select(.Key=="Name").Value')
export OIDC_BUCKET_NAME=$(echo $OIDC_BUCKET_HOST | awk -F'.' '{print$1}')
export CLUSTER_REGION=$(echo $OIDC_BUCKET_HOST | awk -F'.' '{print$3}')
```

Check the discovered var values

> All the values must be discovered.

```bash
cat <<EOF
OIDC_BUCKET_HOST=$OIDC_BUCKET_HOST
OIDC_ARN_S3=$OIDC_ARN_S3
CLUSTER_NAME=$CLUSTER_NAME
OIDC_BUCKET_NAME=$OIDC_BUCKET_NAME
CLUSTER_REGION=$CLUSTER_REGION
EOF
```

3) Make sure you can reach (read) the S3 Bucket created with the default name

```bash
aws s3 ls --region $CLUSTER_REGION s3://$OIDC_BUCKET_NAME
```

4) A clean work directory: a lot of files will be created, make sure you switched to a new work directory to save the files properly (it can be used in the future for rollback)

5) An OpenShift user with cluster-admin permissions

## Validate and Backup <a name="validate-backup"></a>

### Validate tokens <a name="validate-tokens"></a>

This section is to make sure everything is working correctly in your existing environment.

The steps described below will use the credentials provided to the `ebs-csi driver` component, trying to assume the role using `aws-cli`.

It's expected that the existing token will be able to authenticate in AWS. Otherwise, you must abort the operations and do not try to run any step described in the next sections

```bash
## test existing token
# Get Token path from AWS credentials mounted to pod
TOKEN_PATH=$(oc get secrets ebs-cloud-credentials \
    -n openshift-cluster-csi-drivers \
    -o jsonpath='{.data.credentials}' |\
    base64 -d |\
    grep "^web_identity_token_file" |\
    awk '{print$3}')

# Get Controler's pod
CAPI_POD=$(oc get pods -n openshift-cluster-csi-drivers \
    -l app=aws-ebs-csi-driver-controller \
    -o jsonpath='{.items[*].metadata.name}' | awk '{print $1}')

# Extract tokens from the pod
TOKEN=$(oc exec -n openshift-cluster-csi-drivers ${CAPI_POD} \
    -c csi-driver -- cat ${TOKEN_PATH})

echo $TOKEN | awk -F. '{ print $2 }' | base64 -d 2>/dev/null | jq .iss

IAM_ROLE=$(oc get secrets ebs-cloud-credentials \
    -n openshift-cluster-csi-drivers \
    -o jsonpath='{.data.credentials}' |\
    base64 -d |\
    grep "^role_arn" |\
    awk '{print$3}')

echo $IAM_ROLE

aws sts assume-role-with-web-identity \
    --role-arn "${IAM_ROLE}" \
    --role-session-name "my-session" \
    --web-identity-token "${TOKEN}"
```

### Backup existing state <a name="backup"></a>

- Get Objects and existing authentication response

```bash
export BACKUP_PATH=$PWD/current-cluster
mkdir $BACKUP_PATH

oc get authentication -o yaml |tee -a $BACKUP_PATH/authentication.yaml

aws sts assume-role-with-web-identity \
    --role-arn "${IAM_ROLE}" \
    --role-session-name "my-session" \
    --web-identity-token "${TOKEN}" \
    | jq -r '.Credentials=""' \
    | tee ${BACKUP_PATH}/identities.json
```

- Save the current IAM Roles

```bash
aws iam list-roles \
    | jq -r  ".Roles[] | select(.RoleName | startswith(\"${CLUSTER_NAME}-openshift\"))" \
    | tee ${BACKUP_PATH}/iam-roles.json
```

## Migrate

### Setup new OIDC in OpenShift <a name="setup"></a>

#### Create the NGINX hosting server <a name="setup-nginx"></a>

- Create project

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: aws-sts-oidc
  labels:
    name: aws-sts-oidc
    kubernetes.io/metadata.name: aws-sts-oidc
EOF
```

- Create service account

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-sts-oidc-sa
  namespace: aws-sts-oidc
EOF
```

- Create RBAC for service account

```bash
oc apply -f - <<EOF
###SecurityContextConstraints
allowHostDirVolumePlugin: true
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities:
- DAC_READ_SEARCH
- SYS_RESOURCE
apiVersion: security.openshift.io/v1
defaultAddCapabilities: null
fsGroup:
  type: RunAsAny
groups: []
kind: SecurityContextConstraints
metadata:
  annotation: null
  name: aws-sts-oidc
priority: null
readOnlyRootFilesystem: false
requiredDropCapabilities:
- KILL
- MKNOD
- SYS_CHROOT
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
users:
- system:serviceaccount:aws-sts-oidc:aws-sts-oidc-sa
volumes:
- configMap
- downwardAPI
- emptyDir
- hostPath
- persistentVolumeClaim
- secret
EOF
```

- Create service 

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: token
  namespace: aws-sts-oidc
  labels:
    name: aws-sts-oidc
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
      name: http
  selector:
    app: aws-sts-oidc
EOF
```

- Create route 

```bash
oc apply -f - <<EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: token
  namespace: aws-sts-oidc
spec:
  host: aws-oidc.apps.openshift.example.com
  to:
    kind: Service
    name: token
    weight: 100
  wildcardPolicy: None
  tls:
    termination: edge
EOF
```

- Get the NGINX Host
```bash

#token.aws-sts-oidc.svc.cluster.local
export AWS_OIDC_HOST=$(oc get route -n aws-sts-oidc token -o jsonpath='{.status.ingress[*].host}')

echo ${AWS_OIDC_HOST}
```

#### Create and patch the new OIDC discovery documents and JWKS<a name="setup-oidc-documents"></a>

- Download the current OIDC files (discovery document and JWKS) to the local disk:

```bash
aws s3 sync s3://${OIDC_BUCKET_NAME} ./bucket
```

OIDC documents:

```bash
$ ls -a bucket
.  ..  keys.json  .well-known
```

- Patch the new documents with the NGINX Host Domain name:
```bash
sed -i "s/${OIDC_BUCKET_HOST}/${AWS_OIDC_HOST}/g" bucket/.well-known/openid-configuration
```

- Store the patched files in a ConfigMap

```bash
oc create configmap -n aws-sts-oidc aws-sts-oidc --from-file=keys.json=bucket/keys.json --from-file=openid-configuration=bucket/.well-known/openid-configuration
```

- Deploy the NGINX Host

```bash
oc apply -f - <<EOF
kind: Deployment
apiVersion: apps/v1
metadata:
  name: aws-sts-oidc
  namespace: aws-sts-oidc
spec:
  replicas: 2
  selector:
    matchLabels:
      app: aws-sts-oidc
  template:
    metadata:
      labels:
        app: aws-sts-oidc
        deployment: aws-sts-oidc
    spec:
      nodeSelector:
        kubernetes.io/os: linux
        node-role.kubernetes.io/worker: ''
      serviceAccount: aws-sts-oidc-sa 
      securityContext:
        runAsUser: 0
        fsGroup: 0
      volumes:
        - name: aws-sts-oidc
          configMap:
            name: aws-sts-oidc
            items:
              - key: keys.json
                path: keys.json
              - key: openid-configuration
                path: .well-known/openid-configuration
            defaultMode: 420
      containers:
        - name: aws-sts-oidc
          image: nginx:latest
          ports:
            - containerPort: 80
              protocol: TCP
          resources:
            requests:
              cpu: 100m
              memory: 256M
          readinessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 5
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
          livenessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 15
            timeoutSeconds: 1
            periodSeconds: 20
            successThreshold: 1
            failureThreshold: 3
          volumeMounts:
            - name: aws-sts-oidc
              readOnly: true
              mountPath: /usr/share/nginx/html
          imagePullPolicy: IfNotPresent
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - aws-sts-oidc
      restartPolicy: Always
EOF
```

Now the NGINX Host is serving the OIDC, test it:

```
$ curl https://${AWS_OIDC_HOST}/keys.json
$ curl https://${AWS_OIDC_HOST}/.well-known/openid-configuration
```

#### Create the new OIDC using the NGINX Host
<a name="setup-oidc-idp"></a>

- Extract the service account signer public key, to generate the IdP by `ccoctl`:

```bash
oc get configmap bound-sa-token-signing-certs \
    --namespace openshift-kube-apiserver \
    --output json \
    | jq --raw-output '.data["service-account-001.pub"]' \
    > serviceaccount-signer.public
```

- Generate the IdP files into the local directory `new-oidc`:

```bash
ccoctl aws create-identity-provider \
    --name=${CLUSTER_NAME} \
    --region=${CLUSTER_REGION} \
    --public-key-file=${PWD}/serviceaccount-signer.public \
    --output-dir=new-oidc/ \
    --dry-run
```

- Patch the IdP OIDC to the new Domain name

```bash
sed -i "s/${OIDC_BUCKET_HOST}/${AWS_OIDC_HOST}/g" new-oidc/04-iam-identity-provider.json
```

- Discover the thumbprint for the keys from the NGINX Host URL:

> AWS Docs - Getting the Thumbprint: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc_verify-thumbprint.html

```bash
openssl s_client -servername ${AWS_OIDC_HOST} \
    -showcerts -connect ${AWS_OIDC_HOST}:443 </dev/null \
    | openssl x509 -outform pem > certificate.crt

export CERT_THUMBPRINT=$(openssl x509 -in certificate.crt -fingerprint -sha1 -noout \
    | awk -F'=' '{print$2}' | tr -d ':')

jq -r ".ThumbprintList=[\"$CERT_THUMBPRINT\"]" ${PWD}/new-oidc/04-iam-identity-provider.json \
    > ${PWD}/new-oidc/04-iam-identity-provider-new.json
```

- Create the identity provider AWS OIDC:

```bash
aws iam create-open-id-connect-provider \
    --cli-input-json file://${PWD}/new-oidc/04-iam-identity-provider-new.json \
    > ${PWD}/new-oidc/04-iam-identity-provider-object.json 

export OIDC_ARN=$(jq -r .OpenIDConnectProviderArn ${PWD}/new-oidc//04-iam-identity-provider-object.json)

echo ${OIDC_ARN}
```

### Patch the cluster to use the new OIDC<a name="patch-cluster"></a>

- Patch the trusted policy documents with the new OIDC URL

```bash
sed "s/${OIDC_BUCKET_HOST}/${AWS_OIDC_HOST}/g" ${BACKUP_PATH}/iam-roles.json \
    | tee iam-roles-new.json
```

- Patch the IAM Roles Trusted Policy documents

> NOTE 1: from here, the cluster will lose access to the integrated components (machine-api, image registry, CSI, ...)

> NOTE 2: The script below should be run carefully, it was created to show the current and desired policies. If you find anything that does not match the expected changes, abort it immediately.

> Helper [`aws iam get-role`](https://docs.aws.amazon.com/cli/latest/reference/iam/get-role.html)

> Helper [`aws iam update-assume-role-policy`](https://docs.aws.amazon.com/cli/latest/reference/iam/update-assume-role-policy.html)

```bash
for ROLE_NAME in $(jq -r .RoleName iam-roles-new.json);
do
    echo -e ">>>>>\n#> (1) CURRENT IAM Role \"$ROLE_NAME\":";
    aws iam get-role --role-name $ROLE_NAME | jq .Role;

    echo -e "\n#> (2) NEW IAM Role \"$ROLE_NAME\" AssumeRolePolicyDocument:";
    jq -r ". | select(.RoleName == \"$ROLE_NAME\").AssumeRolePolicyDocument" iam-roles-new.json \
        | tee ${PWD}/iam-roles-new-$ROLE_NAME.json
    
    read -p "ATTENTION: The AssumeRolePolicyDocument for IAM Role(1) will be patched to the value of (2). Do you want to continue? [y/n]: " answer
    if [ -z "$answer" ] || [ "$answer" != "y" ]
    then
        echo "answer[$answer]. Canceling the operation.";
        break
    fi
    echo "Patching..."
    aws iam update-assume-role-policy \
        --role-name $ROLE_NAME \
        --policy-document file:///${PWD}/iam-roles-new-$ROLE_NAME.json
    echo "Done! Return code=$?"
done
```

- Patch the issuer URL to the new OIDC URL on the Authentication object:

```bash
oc patch authentication cluster \
    --type=merge \
    -p "{\"spec\":{\"serviceAccountIssuer\":\"https://${AWS_OIDC_HOST}\"}}"
```

- Wait for the kube-apiserver rollout

> Wait to clean the `PROGRESSING=TRUE`. It could take some minutes to start and complete.

```bash
$ oc get co kube-apiserver -w
$ oc get pods -n openshift-kube-apiserver -l apiserver=true -w
```

- Restart all pods:

```bash
oc -n openshift-cluster-csi-drivers delete po --all
```

- Test the new token

> Repeat the steps in the section ["Validate tokens" section](#validate-tokens)

> Make sure the JWT token has the NGINX host name as the Issuer URL, field `.iss`

> Make sure you can assume the role correctly and the signer will be the NGINX host: `.Provider` in the answer from `assume-role-with-web-identity`

If you have completed these steps successfully, the cluster is using the new identity provider with NGINX host.

### Revoke public access to the S3 Bucket<a name="revoke-s3-public-access"></a>

- Change the default policy blocking public access to the bucket:

```bash
aws s3api put-public-access-block \
    --bucket ${OIDC_BUCKET_NAME} \
    --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

- Test it (expected to fail [HTTP 403])

```bash
curl -vvv https://${OIDC_BUCKET_HOST}/keys.json
```

## Rollback to OIDC with S3 Public URL<a name="rollback"></a>

When the process to migrate to a private bucket has failed, and you want to roll back to the OIDC issuer URL pointing to the public S3 Bucket, you must follow the steps below.

- Reopen the bucket policy

```bash
aws s3api put-public-access-block \
    --bucket ${OIDC_BUCKET_NAME} \
    --public-access-block-configuration \
    BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
```

- Replace the `serviceAccountIssuer` with the S3 Bucket's URL

```bash
oc patch authentication cluster \
    --type=merge \
    -p "{\"spec\":{\"serviceAccountIssuer\":\"https://${OIDC_BUCKET_HOST}\"}}"
```

- Patch the Assume Role policy (Trusted Policy)

```bash
# Patch the roles back to S3
for ROLE_NAME in $(jq -r .RoleName ${BACKUP_PATH}/iam-roles.json);
do
    echo -e ">>>>>\n#> (1) CURRENT IAM Role \"$ROLE_NAME\":";
    aws iam get-role --role-name $ROLE_NAME | jq .Role;

    echo -e "\n#> (2) NEW IAM Role \"$ROLE_NAME\" AssumeRolePolicyDocument:";
    jq -r ". | select(.RoleName == \"$ROLE_NAME\").AssumeRolePolicyDocument" ${BACKUP_PATH}/iam-roles.json \
        | tee ${BACKUP_PATH}/iam-roles-rollback-$ROLE_NAME.json
    
    read -p "ATTENTION: The AssumeRolePolicyDocument for IAM Role(1) will be patched to the value of (2). Do you want to continue? [y/n]: " answer
    if [ -z "$answer" ] || [ "$answer" != "y" ]
    then
        echo "answer[$answer]. Canceling the operation.";
        break
    fi
    echo "Patching..."
    aws iam update-assume-role-policy \
        --role-name $ROLE_NAME \
        --policy-document file:///${BACKUP_PATH}/iam-roles-rollback-$ROLE_NAME.json
    echo "Done! Return code=$?"
done
```

- Wait for the kube-apiserver to apply the configuration (PROGRESSING=FALSE)

```bash
oc get co kube-apiserver -w
oc get pods -n openshift-kube-apiserver -l apiserver=true -w
```

- Restart all pods

```bash
oc -n openshift-cluster-csi-drivers delete po --all
```

## Delete the Old OIDC identity provider<a name="delete"></a>

If you run successfully the steps and tested them, you can remove the old OIDC pointing to the S3.

```bash
aws iam delete-open-id-connect-provider --open-id-connect-provider-arn $OIDC_ARN_S3
```
