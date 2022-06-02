# Deployment To Cloud Foundry with App Metrics
To deploy run the below and update your username, password and jdbc host, port, and database:
```sh
cf create-service -c '{ "git": { "uri": "https://github.com/btrieger/spring-petclinic-cloud-config.git", "periodic": true }, "count": 3 }' p.config-server standard config
cf create-service p.service-registry standard registry 
cf create-service credhub default customers-db -c "{\"jdbcUrl\":\"jdbc:mysql://<HOST>:<PORT>/<DB>\",\"username\":\"<USERNAME>\",\"password\":\"<PASSWORD>\" }"
cf create-service credhub default visits-db -c "{\"jdbcUrl\":\"jdbc:mysql://<HOST>:<PORT>/<DB>\",\"username\":\"<USERNAME>\",\"password\":\"<PASSWORD>\" }"
cf create-service credhub default vets-db -c "{\"jdbcUrl\":\"jdbc:mysql://<HOST>:<PORT>/<DB>\",\"username\":\"<USERNAME>\",\"password\":\"<PASSWORD>\" }"
mvn clean package -Pcloud
cf push --no-start
cf add-network-policy api-gateway --destination-app vets-service --protocol tcp --port 8080
cf add-network-policy api-gateway --destination-app customers-service --protocol tcp --port 8080
cf add-network-policy api-gateway --destination-app visits-service --protocol tcp --port 8080
cf start vets-service & cf start visits-service & cf start customers-service & cf start api-gateway &
cf install-plugin metric-registrar
cf install-plugin -r CF-Community "log-cache"
cf register-metrics-endpoint vets-service /actuator/prometheus --internal-port 8080
cf register-metrics-endpoint visits-service /actuator/prometheus --internal-port 8080
cf register-metrics-endpoint customers-service /actuator/prometheus --internal-port 8080
cf register-metrics-endpoint api-gateway /actuator/prometheus --internal-port 8080
```


## Add Indicator

Create indicators.yml and update the spec.product.name if needed to point at your product. (format org,space,app)
```yaml
---
apiVersion: indicatorprotocol.io/v1
kind: IndicatorDocument

spec:
  product:
   name: system,system,customers-service
   version: 0.0.1

  indicators:
   - name: "PetclinicOwnerTimer"
     promql: "sum(petclinic_owner_seconds_count{method=\"POST\", status=\"201\", source_id=\"$sourceId\"})"
     documentation:
      title: "Petclinic Owner Timer"
     presentation:
      units: "counts"
```

Run below command to apply:
```sh
curl -vvv https://metrics.sys.latest-pas.pci.cf-app.com/indicator-documents -H "Authorization: $(cf oauth-token)" --data-binary "@indicators.yml"
```
