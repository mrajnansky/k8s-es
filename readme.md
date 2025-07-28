# Create token for kibana
elasticsearch-service-tokens create elastic/kibana kibana-token

curl -u elastic:testingPassword -X POST "http://localhost:9200/_security/service/elastic/kibana/credential/token/kibana-token"


curl -u elastic:testingPassword -X POST "http://localhost:9200/_security/user/kibana_system/_password" -H "Content-Type: application/json" -d '{"password":"kibana_password_123"}'