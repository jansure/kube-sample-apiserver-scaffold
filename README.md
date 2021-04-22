# sample-apiserver

演示如何使用 kubernetes apiserver 



```sh
go get -d k8s.io/sample-apiserver
cd $GOPATH/src/k8s.io/sample-apiserver  # assuming your GOPATH has just one entry
godep restore
```

### go 1.11 之后的使用方式，如果使用的是go 1.11之前的版本请联系 bjzhangshuai@gmail.com


```sh
git clone https://github.com/kubernetes/sample-apiserver.git
cd sample-apiserver
go mod vendor
```

### 编译

```
linux:
    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -o artifacts/simple-image/kube-sample-apiserver
macos:
    CGO_ENABLED=0 GOOS=darwin GOARCH=amd64 go build -a -o artifacts/simple-image/kube-sample-apiserver-darwin
```

### 打包镜像

```
docker build -t 172.19.217.41:5000/kube-sample-apiserver:lastest ./artifacts/simple-image
docker push 172.19.217.41:5000/kube-sample-apiserver:lastest

我在本地搭建了docker registry 所以就使用本地的了。
```

### 部署到集群

```
kubectl apply -f artifacts/example
```

## 本地启动并且生成本地证书证书

1. First we need a CA to later sign the client certificate:

   ``` shell
   openssl req -nodes -new -x509 -keyout ca.key -out ca.crt
   ```

2. Then we create a client cert signed by this CA for the user `development` in the superuser group
   `system:masters`:

   ``` shell
   openssl req -out client.csr -new -newkey rsa:4096 -nodes -keyout client.key -subj "/CN=development/O=system:masters"
   openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -out client.crt
   ```

3. As curl requires client certificates in p12 format with password, do the conversion:

   ``` shell
   openssl pkcs12 -export -in ./client.crt -inkey ./client.key -out client.p12 -passout pass:password
   ```
   
4. Install etcd:
    ``` shell
        brew install etcd
    ```
   Only for MacOs,Other os please referring etcd installation guide

4. With these keys and certs in-place, we start the server:

   ``` shell
   etcd &
   
   cd $GOPATH/src/sample-apiserver/artifacts/simple-image  && ./kube-sample-apiserver-darwin --secure-port 8443 --etcd-servers http://127.0.0.1:2379 --v=7 \
      --client-ca-file ca.crt \
      --kubeconfig ~/.kube/config \
      --authentication-kubeconfig ~/.kube/config \
      --authorization-kubeconfig ~/.kube/config
   ```


5. Use curl to access the server using the client certificate in p12 format for authentication:

   ``` shell
   curl -fv -k --cert-type P12 --cert client.p12:password \
      https://localhost:8443/apis/wardle.example.com/v1alpha1/namespaces/default/flunders
   ```

   Or use wget:
   ``` shell
   wget -O- --no-check-certificate \
      --certificate client.crt --private-key client.key \
      https://localhost:8443/apis/wardle.example.com/v1alpha1/namespaces/default/flunders
   ```

   Note: Recent MacOs versions broke client certs with curl. On Mac try `brew install httpie` and then:

   ``` shell
   http --verify=no --cert client.crt --cert-key client.key \
      https://localhost:8443/apis/wardle.example.com/v1alpha1/namespaces/default/flunders
   ```
