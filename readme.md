# Create token for kibana
elasticsearch-service-tokens create elastic/kibana kibana-token

curl -u elastic:testingPassword -X POST "http://localhost:9200/_security/service/elastic/kibana/credential/token/kibana-token"