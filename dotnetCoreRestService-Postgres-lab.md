# TAP Tanzu C# Restful Web App

This labs uses the Services Toolkit class-claims and dynamic provisioning approach leveraging the Bitnami Services.

A ClassClaim targets a ClusterInstanceClass in the Kubernetes cluster. To target this class, the ClassClaim only requires the name of the ClusterInstanceClass. This also uses Crossplane for provisioning of Service Instances under the hood.
Official docs [here](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/services-toolkit-concepts-service-consumption.html#level-4--class-claims-and-provisionerbased-classes-aka-dynamic-provisioning-3) and [here](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/bitnami-services-tutorials-working-with-bitnami-services.html)

You could also use the ResourceClaim approach and a separate Service Operator. A ResourceClaim targets a specific resource in the Kubernetes cluster. To target that resource, the ResourceClaim needs the name, namespace, kind, and API version of the resource. Instructions for this can be found [here](https://github.com/jaimegag/csharp-rest-service/blob/main/DATABASE.md)


Personas participating in this lab:
- Service Operator - Passive, already configured Bitnami Services with ClusterInstanceClass
- App Operator - Confirms available service Classes and creates Class Claims
- App Developer - Manages the application code and uses workload.yaml as he contract with the platform

## Create Class Claim
```bash
# Check available classes (Bitnami Services)
tanzu service class list
# This should return:
  # NAME                  DESCRIPTION
  # mysql-unmanaged       MySQL by Bitnami
  # postgresql-unmanaged  PostgreSQL by Bitnami
  # rabbitmq-unmanaged    RabbitMQ by Bitnami
  # redis-unmanaged       Redis by Bitnami
# These are ClsuterInstanceClasses resources, available cluster wide for better discovery, so that the app dev team doesn't have to know the namespace of the resource
kubectl get clusterInstanceclass
# NAME                   DESCRIPTION             READY   REASON
# mysql-unmanaged        MySQL by Bitnami        True    Ready
# postgresql-unmanaged   PostgreSQL by Bitnami   True    Ready
# rabbitmq-unmanaged     RabbitMQ by Bitnami     True    Ready
# redis-unmanaged        Redis by Bitnami        True    Ready

tanzu service class get postgresql-unmanaged
# This should return:
# NAME:           postgresql-unmanaged
# DESCRIPTION:    PostgreSQL by Bitnami
# READY:          true

# PARAMETERS:
#   KEY        DESCRIPTION                                                  TYPE     DEFAULT  REQUIRED
#   storageGB  The desired storage capacity of the database, in Gigabytes.  integer  1        false

# The only think we need to create now is a Class-Claim
# tanzu service class-claim create customer-database --class postgresql-unmanaged -n myapps
kubectl apply -f ./config/postgres-class-claim.yaml
tanzu services class-claims get customer-database --namespace myapps
# Wait for a Claimed Resource to be referenced. Should look like this:
# Name: customer-database
# Namespace: myapps
# Claim Reference: services.apps.tanzu.vmware.com/v1alpha1:ClassClaim:customer-database
# Class Reference:
#   Name: postgresql-unmanaged
# Parameters: None
# Status:
#   Ready: True
#   Claimed Resource:
#     Name: f9fefef7-aa6a-4e12-9697-a3f36b6e4a41
#     Namespace: myapps
#     Group:
#     Version: v1
#     Kind: Secret
```

## Deploy the Workload
Get Tanzu C# Restful Web ApP from Accelerator. Check `Expose OpenAPI endpoint` and keep all defaults.
Unzip and push to a public Git repository, remember that registry url to use it in the "git" configuration of the `workload.yaml` used below:

```bash
tanzu apps workload create csharp-rest-service \
  -f ./config/crwa-workload.yaml \
  -n myapps \
  --yes
```

## Import workload backstage catalog info into TAP GUI
1. Log into TAP GUI using the url defined in your tap-values.yaml file. Example: http://tap-gui.tap.zora.tkg-vsp-lab.hyrulelab.com/
2. On the software catalog page click on the Register button
3. Use https://github.com/jaimegag/csharp-rest-service/blob/main/catalog/catalog-info.yaml for the url in the form
4. Follow the rest of the instructions in the wizard


## Check Service Bindings and claims

Our SupplyChain generates a ServiceBinding together with the Knative Service (delivery) and the apiDescriptor:
```bash
kubectl get ServiceBinding csharp-rest-service-database -n myapps
# This returns:
# NAME                           READY   REASON   AGE
# csharp-rest-service-database   True    Ready    9h

# If you look inside the ClassClaim you'll find a reference to the Resource created and Secret
kubectl get classclaim customer-database -n myapps -oyaml
# This returns this info in the status at the end
  # provisionedResourceRef:
  #   apiVersion: bitnami.database.tanzu.vmware.com/v1alpha1
  #   kind: XPostgreSQLInstance
  #   name: customer-database-r69qz
  # resourceRef:
  #   apiVersion: v1
  #   kind: Secret
  #   name: f9fefef7-aa6a-4e12-9697-a3f36b6e4a41
  #   namespace: myapps
# That secret has base64 encoded values for the k8 service exposing the DB, the user and password
# DB sservice and pod are in a dedicated namespace named like: customer-database-r69qz
```

Even if we used the Class Claim approach, a ResourceClaim was created under the coves
```bash
tanzu service resource-claims list -n myapps
# Output should look like this
  # NAME                     READY  REASON
  # customer-database-gxht4  True   Ready
```

### Check the API is registered

When we deployed the App, with the `api-descriptor` params included in the `workload.yaml` an API Descriptor object was created:
```bash
kubectl get apidescriptor csharp-rest-service -n myapps
# The output should look like this:
# NAME                  STATUS
# csharp-rest-service   Ready
```

Then the API Descriptor will be read by the `apidescriptor controller` which  call the tap-gui API and register the API entitity into the TAP GUI Database.
Go to the TAP GUI and access the API Explorer. You should see a row with your API, the name should match with the FQDN of the App.
Click on the name to access the API information. Then click on `Definition`: You will see the REST API endpoints and the Open API schemas of the API.



## Test the workload

Access the customer profile endpoint
```bash
curl https://csharp-rest-service.myapps.tap.akkala.tkg-vsp-lab.hyrulelab.com/api/customer-profiles
```

Create a customer profile with a call like this:
```bash
curl -X POST -H 'Content-Type: application/json' https://csharp-rest-service.myapps.tap.akkala.tkg-vsp-lab.hyrulelab.com/api/customer-profiles -d '{"firstName": "Manjit", "lastName": "Singh", "email": "msingh67@bloomberg.net"}'
```

Get a customer profile with the ID received in the previous call
```bash
curl -X GET  https://csharp-rest-service.myapps.tap.akkala.tkg-vsp-lab.hyrulelab.com/api/customer-profiles/{id}
```

Check the data is indeed in the Postgres DB
```bash
kubectl exec -it customer-database-r69qz-0 -n customer-database-r69qz -- bash -c "psql -U postgres"
# connect to the database
\c customer-database-r69qz
# list tables
\dt
# get data from customer_profile table (don't forget the semicolon at the end)
select * from "CustomerProfiles";
# Type \q to exit
```
