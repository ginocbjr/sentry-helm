# Helm Chart for Sentry
This is a helm chart for sentry. This is deployed and tested on Azure Kubernetes Service. To deploy this to other Kubernetes instance, you will need to make some changes to `init/templates/storage-class-azure-file.yml` because that is for azure only.

## Installation Instructions
1. Install sentry-init
```bash
helm upgrade sentry-init ./sentry/init --install --debug
```
2. Generate secret key and convert to base64
```bash
docker run --rm sentry config generate-secret-key
echo 'generated-secret-key' | base64
```
Make sure to save the secret key to a secure location. If you lose this, you will have to create a totally new sentry instance. Verify if you converted to base64 properly by running this command and comparing
```bash
echo 'base64-secret-key' | base64 --decode
```

3. Upgrade the database
```bash
kubectl run sentry-upgrade --image=sentry:9.0-onbuild --rm -i --tty \
--env="SENTRY_MEMCACHED_HOST=service-sentry-memcached" \
--env="SENTRY_REDIS_HOST=service-sentry-redis" \
--env="SENTRY_POSTGRES_HOST=service-sentry-pg" \
--env="SENTRY_SECRET_KEY=generated-secret-key-raw" \
-nsentry \
-- upgrade
```
Please note that value for SENTRY_SECRET_KEY should be the raw string and NOT in base64 format. Also this will ask for admin email and password. Make sure to set it now.

4. Install sentry-main
```bash
helm upgrade sentry-main ./sentry/main --debug --install  \
--set mailUsername="mail-server-username-in-base64" \
--set mailPassword="mail-server-password-in-base64" \
--set secretKey="generated-secret-key-in-base64"
```

5. Wait for few moments and check pods and services if they exist and are running
```bash
# pods
NAME                                READY   STATUS    RESTARTS   AGE
sentry-cron-xxxxxxxxxx-xxxxx        1/1     Running   0          2d17h
sentry-memcached-xxxxxxxxxx-xxxxx   1/1     Running   0          2d17h
sentry-postgres-xxxxxxxxxx-xxxxx    1/1     Running   0          2d17h
sentry-redis-xxxxxxxxxx-xxxxx       1/1     Running   0          2d17h
sentry-web-xxxxxxxxxx-xxxxx         1/1     Running   0          2d17h
sentry-worker-xxxxxxxxxx-xxxxx      1/1     Running   0          2d17h
# services
NAME                       TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)           AGE
service-sentry-memcached   NodePort       xxx.xx.xx.x      <none>         11211:32254/TCP   2d17h
service-sentry-pg          LoadBalancer   xxx.xx.xxx.xxx   xxx.xx.x.xxx   5432:30685/TCP    2d17h
service-sentry-redis       NodePort       xxx.xx.xxx.xxx   <none>         6379:31621/TCP    2d17h
service-sentry-web         LoadBalancer   xxx.xx.xxx.xx    xxx.xx.x.xxx   80:32618/TCP      2d17h
```