# Домашнее задание к занятию «Хранение в K8s. Часть 2»

### Цель задания

В тестовой среде Kubernetes нужно создать PV и продемострировать запись и хранение файлов.

------


### Задание 1

**Что нужно сделать**

Создать Deployment приложения, использующего локальный PV, созданный вручную.

Создал манифест deploiment.yaml включающий в том чсле конфиги pv и pvc:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storage-deployment
  namespace: homework
spec:
  replicas: 1
  selector:
    matchLabels:
      app: storage
  template:
    metadata:
      labels:
        app: storage
    spec:
      containers:
      - name: busybox
        image: busybox:latest
        command: ['sh', '-c', 'while true; do date >> /data/output.txt; sleep 5; done']
        volumeMounts:
        - name: safedata
          mountPath: /data
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 8080
        env:
        - name: HTTP_PORT
          value: "8080"
        volumeMounts:
        - name: safedata
          mountPath: /data
      volumes:
      - name: safedata
        persistentVolumeClaim:
          claimName: pvc1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
  namespace: homework
spec:
  storageClassName: ""
  persistentVolumeReclaimPolicy: Retain
  capacity:
    storage: 1Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /tmp/pv1/
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
  namespace: homework
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Mi


```
Проверяю:
```bash
ser@microk8s:~/kuber-homeworks-2.2$ kubectl apply -f deployment.yaml 
deployment.apps/storage-deployment created
persistentvolume/pv1 created
persistentvolumeclaim/pvc1 created
user@microk8s:~/kuber-homeworks-2.2$ kubectl get deployments.apps storage-deployment 
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
storage-deployment   0/1     1            0           10s
user@microk8s:~/kuber-homeworks-2.2$ kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM           STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pv1    1Mi        RWO            Retain           Bound    homework/pvc1                  <unset>                          18s
user@microk8s:~/kuber-homeworks-2.2$ kubectl get pvc
NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
pvc1   Bound    pv1      1Mi        RWO                           <unset>                 23s
user@microk8s:~/kuber-homeworks-2.2$ 
```
Поочереди подключаюсь к контейнерам пода и демонстрирую, что multitool может читать файл, в который busybox пишет каждые пять секунд в общей директории.
```bash
user@microk8s:~/kuber-homeworks-2.2$ kubectl exec -it pods/storage-deployment-5d57b5494d-hcd6g -c busybox -- sh
/ # tail -f /data/output.txt 
Sat Nov  9 06:20:23 UTC 2024
Sat Nov  9 06:20:28 UTC 2024
Sat Nov  9 06:20:33 UTC 2024
Sat Nov  9 06:20:38 UTC 2024
Sat Nov  9 06:20:43 UTC 2024
Sat Nov  9 06:20:48 UTC 2024
Sat Nov  9 06:20:53 UTC 2024
Sat Nov  9 06:20:58 UTC 2024
Sat Nov  9 06:21:03 UTC 2024
Sat Nov  9 06:21:08 UTC 2024
Sat Nov  9 06:21:13 UTC 2024
^C
/ # 
command terminated with exit code 130
user@microk8s:~/kuber-homeworks-2.2$ kubectl exec -it pods/storage-deployment-5d57b5494d-hcd6g -c multitool--- sh
/ # tail -f /data/output.txt 
Sat Nov  9 06:20:43 UTC 2024
Sat Nov  9 06:20:48 UTC 2024
Sat Nov  9 06:20:53 UTC 2024
Sat Nov  9 06:20:58 UTC 2024
Sat Nov  9 06:21:03 UTC 2024
Sat Nov  9 06:21:08 UTC 2024
Sat Nov  9 06:21:13 UTC 2024
Sat Nov  9 06:21:18 UTC 2024
Sat Nov  9 06:21:23 UTC 2024
Sat Nov  9 06:21:28 UTC 2024
Sat Nov  9 06:21:33 UTC 2024
^C
/ # 
command terminated with exit code 130
```
Удаляю deployment и pvc :
```bash
user@microk8s:~/kuber-homeworks-2.2$ kubectl delete deployments.apps storage-deployment 
deployment.apps "storage-deployment" deleted
user@microk8s:~/kuber-homeworks-2.2$ kubectl delete pvc pvc1 
persistentvolumeclaim "pvc1" deleted
user@microk8s:~/kuber-homeworks-2.2$ kubectl get pv pv1 
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM           STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
pv1    1Mi        RWO            Retain           Released   homework/pvc1                  <unset>                          4m2s
user@microk8s:~/kuber-homeworks-2.2$ 
```
После удаления PVC, PV сменил статус с Bound на Released, так как больше не свзан с PVC.

Проверяя, что файл сохранился на локальном диске ноды:
```bash
user@microk8s:~/kuber-homeworks-2.2$ cat /tmp/pv1/output.txt 
Sat Nov  9 06:19:58 UTC 2024
Sat Nov  9 06:20:03 UTC 2024
Sat Nov  9 06:20:08 UTC 2024
Sat Nov  9 06:20:13 UTC 2024
Sat Nov  9 06:20:18 UTC 2024
Sat Nov  9 06:20:23 UTC 2024
Sat Nov  9 06:20:28 UTC 2024
Sat Nov  9 06:20:33 UTC 2024
Sat Nov  9 06:20:38 UTC 2024
Sat Nov  9 06:20:43 UTC 2024
Sat Nov  9 06:20:48 UTC 2024
Sat Nov  9 06:20:53 UTC 2024
Sat Nov  9 06:20:58 UTC 2024
Sat Nov  9 06:21:03 UTC 2024
Sat Nov  9 06:21:08 UTC 2024
Sat Nov  9 06:21:13 UTC 2024
Sat Nov  9 06:21:18 UTC 2024
Sat Nov  9 06:21:23 UTC 2024
Sat Nov  9 06:21:28 UTC 2024
Sat Nov  9 06:21:33 UTC 2024
Sat Nov  9 06:21:38 UTC 2024
Sat Nov  9 06:21:43 UTC 2024
Sat Nov  9 06:21:48 UTC 2024
Sat Nov  9 06:21:53 UTC 2024
Sat Nov  9 06:21:58 UTC 2024
Sat Nov  9 06:22:03 UTC 2024
Sat Nov  9 06:22:08 UTC 2024
```
Удаляю PV и проверяю что с файломна ноде:
```bash 
user@microk8s:~/kuber-homeworks-2.2$ kubectl delete pv pv1 
persistentvolume "pv1" deleted
user@microk8s:~/kuber-homeworks-2.2$ cat /tmp/pv1/output.txt 
Sat Nov  9 06:19:58 UTC 2024
Sat Nov  9 06:20:03 UTC 2024
Sat Nov  9 06:20:08 UTC 2024
Sat Nov  9 06:20:13 UTC 2024
Sat Nov  9 06:20:18 UTC 2024
Sat Nov  9 06:20:23 UTC 2024
Sat Nov  9 06:20:28 UTC 2024
Sat Nov  9 06:20:33 UTC 2024
Sat Nov  9 06:20:38 UTC 2024
Sat Nov  9 06:20:43 UTC 2024
Sat Nov  9 06:20:48 UTC 2024
Sat Nov  9 06:20:53 UTC 2024
Sat Nov  9 06:20:58 UTC 2024
Sat Nov  9 06:21:03 UTC 2024
Sat Nov  9 06:21:08 UTC 2024
Sat Nov  9 06:21:13 UTC 2024
Sat Nov  9 06:21:18 UTC 2024
Sat Nov  9 06:21:23 UTC 2024
Sat Nov  9 06:21:28 UTC 2024
Sat Nov  9 06:21:33 UTC 2024
Sat Nov  9 06:21:38 UTC 2024
Sat Nov  9 06:21:43 UTC 2024
```
Файл не удалился т.к. для PV установлен параметр   _persistentVolumeReclaimPolicy: Retain_


------

### Задание 2


Создать Deployment приложения, которое может хранить файлы на NFS с динамическим созданием PV.

Включитл и настроил NFS-сервер на MicroK8S:
```bash
user@microk8s:~/kuber-homeworks-2.2$ microk8s enable storage
Infer repository core for addon storage
Addon core/storage is already enabled
user@microk8s:~/kuber-homeworks-2.2$ kubectl get storageclasses.storage.k8s.io 
NAME                          PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
microk8s-hostpath (default)   microk8s.io/hostpath   Delete          WaitForFirstConsumer   false                  16d
```

Создал Deployment приложения состоящего из multitool с подключением к встроенному nfs:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-nfs
  labels:
    app: nfs-app
  namespace: homework
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-app
  template:
    metadata:
      labels:
        app: nfs-app
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        env:
          - name: HTTP_PORT
            value: "8080"
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: vol-nfs
          mountPath: /nfs-share
      volumes:
      - name: vol-nfs
        persistentVolumeClaim:
          claimName: nfs

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spe
  storageClassName: "microk8s-hostpath"
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi

```
Применяю и смотрю какие PV и PVC создались:
```bash
user@microk8s:~/kuber-homeworks-2.2$ kubectl get pvc
NAME   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS        VOLUMEATTRIBUTESCLASS   AGE
nfs    Bound    pvc-e824edce-5c3e-43ee-9c6c-b79d9f7d5a0c   10Mi       RWO            microk8s-hostpath   <unset>                 18s
user@microk8s:~/kuber-homeworks-2.2$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS        VOLUMEATTRIBUTESCLASS   REASON   AGE
pvc-e824edce-5c3e-43ee-9c6c-b79d9f7d5a0c   10Mi       RWO            Delete           Bound    homework/nfs   microk8s-hostpath   <unset>                          10s
user@microk8s:~/kuber-homeworks-2.2$ 
```

Проверяю возможность чтения и записи файла изнутри пода. 
```bash
user@microk8s:~/kuber-homeworks-2.2$ kubectl exec -ti pods/deployment-nfs-96f8c65c9-klftw -- sh
/ # cd /nfs-share/
/nfs-share # echo > test.txt Hi!!!
/nfs-share # cat test.txt 
Hi!!!
/nfs-share # rm test.txt 
/nfs-share # ls
/nfs-share # 
```


------


