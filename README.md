# Allure TestOps Official k8S Deploy

### Values Example:
```yaml
## Override Version
version: 3.190.0

## Credentials for accessing AllureTestOps as Admin on default auth scheme
username: admin
password: admin

## Security Context
runAsUser: 65534

## your-domain.tld
host: localhost

## Registry Auth. Access to registry secrets can be obtained from QAMeta team
registry:
  # Private registry or Proxy like Nexus
  enabled: false
  # Registry Domain like quay.io / docker.io / ghcr.io / ...
  repo: docker.io
  # Prefix with registry name
  name: allure
  imagePullSecret: qameta-secret
  pullPolicy: IfNotPresent
  auth:
    username: foo
    password: bar

## Role Based Access Control
## Ref: https://kubernetes.io/docs/admin/authorization/rbac/
rbac:
  enabled: true
  rules:
    - apiGroups:
        - ''
      resources:
        - endpoints
        - services
        - pods
        - configmaps
        - secrets
      verbs:
        - get
        - watch
        - list

## AWS Injection
# Make Sure you have RBAC enabled
# Ref: https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/
aws:
  enabled: false
  accountId: 195700002069 # Your AWS Account ID
  iamRole: s3-bucket-objects # Name of your AWS ROLE allowing reading/writing s3

# Please be careful. Make sure you have no DB migrations with upcoming version. If so change type to Recreate and plan Downtime
# Also please disable health checks during DB migration
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0

network:
  # Nginx Ingress
  ingress:
    enabled: false
    className:
    annotations:
      kubernetes.io/ingress.class: nginx
      nginx.ingress.kubernetes.io/ssl-redirect: "false"
      nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  # Istio Gateway
  istio:
    enabled: false
    gatewaySelector:
      istio: ingressgateway
    domain_exceptions: "https://jira.your-domain.io https://jira.your-domain.ru" # makes Allure TestOps accessible from Jira Plugin
  # TLS Settings
  tls:
    enabled: false
    secretName: allure-tls # Secret with SSL termination secrets.
    hstsEnabled: false
  # Important! Qameta Team doesn't provide Certmanager itself, if you don't have one,
  # install from https://cert-manager.io/docs/installation/ and create ClusterIssuer
  # and provide Challenge type e.g. acme http-01 or dns-01
  certmanager:
    enabled: false
    # It is important where certificate is located depending on your Ingress/Istio Gateway
    # IstioGateway require secrets with tls certs to be located in istio-system ns
    namespace: istio-system
    # As Let's Encrypt is not the only provider, you may use your own.
    issuerName: letsencrypt-issuer
    issuerGroup: cert-manager.io

# Redis Config
redis:
  # Set Disabled if you have external Redis
  enabled: true
  # External Redis Host
  host: redis.example.com
  architecture: standalone
  auth:
    password: allure
  sentinel:
    enabled: false
    masterSet: big_master
    nodes: []

# RabbitMQ Config
rabbitmq:
  enabled: true
  external:
    enabled: false
    host: mq.example.com
  auth:
    erlangCookie: fTwP5LxRVjZ9XJkyWmJSKR5hPDWMjkQx # Set your own random string
    username: allure
    password: allure
  resources: {}

postgresql:
  enabled: true
  connectionTimeout: 60000
  postgresqlUsername: allure
  postgresqlPassword: allure
  external:
    active: false
    endpoint: db.example.com
    uaaDbName: uaa
    uaaUsername: uaa
    uaaPassword: secret
    reportDbName: report
    reportUsername: report
    reportPassword: secret
  initdbScripts:
    init.sql: |
      CREATE DATABASE uaa TEMPLATE template0 ENCODING 'utf8' LC_COLLATE = 'en_US.UTF-8' LC_CTYPE = 'en_US.UTF-8';
      CREATE DATABASE report TEMPLATE template0 ENCODING 'utf8' LC_COLLATE = 'en_US.UTF-8' LC_CTYPE = 'en_US.UTF-8';
  persistence:
    size: 20Gi

# Local FS is NOT disabled for migration purpose. Please bear in mind that primary FS for allure is S3
fs:
  # Supported: S3, LOCAL, NFS
  type: "S3"
  external: false
  pathstyle: true
  migration:
    enabled: false
    directory: /opt/allure/report/storage
  s3:
    endpoint: https://s3.amazonaws.com
    bucket: allure-testops
    region: eu-central-1
    # If you run Allure Testops in AWS EKS you don't need stating accessKey & secretKey
    accessKey: foo
    secretKey: bar
  # Required only if type is Local or NFS
  mountPoint: /storage
  # Create NFS Volume First as PV
  pvc:
    claimName: ""

minio:
  enabled: true
  auth:
    rootUser: WBuetMuTAMAB4M78NG3gQ4dCFJr3SSmU # Replace with your Access Key
    rootPassword: m9F4qupW4ucKBDQBWr4rwQLSAeC6FE2L # Replace with your Secret Key
  disableWebUI: true
  service:
    ports:
      api: 9000
  defaultBuckets: allure-testops
  defaultRegion: qameta-0

allure:
  timeZone: "Europe/Moscow"
  sessionLifespan: 57600
  httpOnly: "true"
  logging: warn
  management:
    # Endpoints to expose e.g. for Prometheus, HealthChecks etc.
    expose: health,info,prometheus,configprops
    cacheTTL: 20s
  uaaContextPath: "/uaa/"
  reportContextPath: "/rs/"
  auth:
    primary: system
    ldap:
      enabled: false
      auth:
        user: user
        pass: pass
      referal: follow
      url: ldap://ldap.example.com:389
      user:
        dnPatterns: sAMAccountName={0}
        searchBase: dc=example,dc=com
        searchFilter: (&((objectClass=Person))(sAMAccountName={0}))
      group:
        searchBase: ou=qa,ou=Security Groups,dc=example,dc=com
        searchFilter: (&(objectClass=Group)(member={0}))
        roleAttribute: cn
      # Syncs User Role by Group membership on any login
      syncRoles: true
      uidAttribute: sAMAccountName
      # Maps your LDAP groups on Allure Groups
      userGroupName: allure_users
      adminGroupName: allure_admins
      # If user is not in groups above, default role is assumed. Possible roles: ROLE_ADMIN,
      # ROLE_USER, ROLE_AUDITOR
      defaultRole: ROLE_AUDITOR
    oidc:
      enabled: false
      client:
        name: Google Auth
        id: foo
        secret: bar
      redirectURI: 'https://example.com/login/oauth2/code/allure'
      authMethod: client_secret_post
      issuerURI: 'https://accounts.google.com'
      authURI: 'https://accounts.google.com/o/oauth2/v2/auth'
      tokenURI: 'https://oauth2.googleapis.com/token'
      userInfoURI: 'https://openidconnect.googleapis.com/v1/userinfo'
      jwksURI: 'https://www.googleapis.com/oauth2/v1/certs'
      userNameAttribute: "email"
      scope: "{openid,email,profile}"

gateway:
  replicaCount: 1
  image: allure-gateway
  tolerations: []
  affinity: {}
  nodeSelector: {}
  service:
    port: 8080
  env:
    open:
      JAVA_TOOL_OPTIONS: >
        -XX:+UseG1GC
        -XX:+UseStringDeduplication
        -Dsun.jnu.encoding=UTF-8
        -Dfile.encoding=UTF-8
  resources:
    requests:
      memory: 1Gi
      cpu: 1
    limits:
      memory: 2Gi
      cpu: 2
  probes:
    enabled: true
    liveness:
      probe:
        periodSeconds: 40
        timeoutSeconds: 2
        successThreshold: 1
        failureThreshold: 3
        initialDelaySeconds: 60
    readiness:
      probe:
        periodSeconds: 20
        timeoutSeconds: 2
        successThreshold: 1
        failureThreshold: 3
        initialDelaySeconds: 25


uaa:
  replicaCount: 1
  image: allure-uaa
  tolerations: []
  affinity: {}
  nodeSelector: {}
  service:
    port: 8082
  env:
    open:
      JAVA_TOOL_OPTIONS: >
        -XX:+UseG1GC
        -XX:+UseStringDeduplication
        -Dsun.jnu.encoding=UTF-8
        -Dfile.encoding=UTF-8
  # Considering resource planning bear in mind that many pods is good but DB connections are expensive
  resources:
    requests:
      memory: 1Gi
      cpu: 500m
    limits:
      memory: 2Gi
      cpu: 1
  probes:
    # Disable them during updates involving DB migrations
    enabled: true
    liveness:
      probe:
        periodSeconds: 40
        timeoutSeconds: 2
        successThreshold: 1
        failureThreshold: 3
        initialDelaySeconds: 60
    readiness:
      probe:
        periodSeconds: 20
        timeoutSeconds: 5
        successThreshold: 1
        failureThreshold: 10
        initialDelaySeconds: 60

report:
  replicaCount: 1
  image: allure-report
  tolerations: []
  affinity: {}
  nodeSelector: {}
  service:
    port: 8081
  persistence:
    accessMode: ReadWriteOnce
    size: 10Gi
    annotations: {}
    finalizers:
      - kubernetes.io/pvc-protection
  env:
    open:
      JAVA_TOOL_OPTIONS: >
        -XX:+UseG1GC
        -XX:+UseStringDeduplication
        -Dsun.jnu.encoding=UTF-8
        -Dfile.encoding=UTF-8
  # Considering resource planning bear in mind that many pods is good but DB connections are expensive
  resources:
    requests:
      memory: 3Gi
      cpu: 1
    limits:
      memory: 4Gi
      cpu: 4
  probes:
    # Disable them during updates involving DB migrations
    enabled: true
    liveness:
      probe:
        periodSeconds: 40
        timeoutSeconds: 2
        successThreshold: 1
        failureThreshold: 3
        initialDelaySeconds: 300
    readiness:
      probe:
        periodSeconds: 30
        timeoutSeconds: 5
        successThreshold: 1
        failureThreshold: 10
        initialDelaySeconds: 60
```