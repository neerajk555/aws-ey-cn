# Exercise 9: EBS-Backed Storage

## What This Exercise Does

A **PersistentVolumeClaim (PVC)** is a request for durable storage that outlives any individual Pod. On the shared EKS cluster, a PVC provisions a REAL AWS **EBS volume** - unlike everything else in these exercises, an EBS volume keeps billing every month **even with zero Pods using it**, until the volume itself is explicitly deleted. This exercise creates one, proves data survives Pod deletion (the actual point of using a PVC at all), and then walks through verifying the underlying volume is truly gone by its exact ID - not just trusting that deleting the PVC object was enough.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.01-0.02 for a few minutes of a 1Gi volume - BUT this is the ONE exercise where 'nothing running' does not mean 'nothing billing'
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 5 complete
- Your instructor has confirmed the EBS CSI driver and a tagged default StorageClass are installed on the shared cluster

## Steps

## Step 1: Create a PVC

**In VS Code:** Create a new file named `~/course/pvc.yaml` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/pvc.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
kubectl apply -f ~/course/pvc.yaml
kubectl get pvc
```


This alone doesn't create the EBS volume yet - a PVC in isolation just sits there requesting storage until something actually uses it, which happens in the next step.

## Step 2: Mount it in a Pod and write data

**In VS Code:** Create a new file named `~/course/pod-with-pvc.yaml` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
spec:
  containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-pvc
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/pod-with-pvc.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
spec:
  containers:
    - name: app
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-pvc
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
kubectl apply -f ~/course/pod-with-pvc.yaml
kubectl exec pvc-demo -- sh -c 'echo persisted-data > /data/file.txt'
```


Two separate things connect the Pod to your storage: `volumes` (bottom) declares that `data` refers to the PVC named `my-pvc` from the previous step, and `volumeMounts` (inside the container spec) says WHERE inside the container's filesystem that volume shows up - here, at `/data`. The final `kubectl exec` command writes a test file into that mounted path, which is what you'll check survives the Pod's deletion in the next step.

## Step 3: Confirm a REAL EBS volume now exists, and capture its exact ID

```
VOLUME_ID=$(kubectl get pv $(kubectl get pvc my-pvc -o jsonpath='{.spec.volumeName}') -o jsonpath='{.spec.csi.volumeHandle}')
echo $VOLUME_ID
aws ec2 describe-volumes --volume-ids $VOLUME_ID --region us-east-1 --query 'Volumes[0].{State:State,Size:Size,VolumeType:VolumeType}'
```

Keep this `$VOLUME_ID` value - you'll use it again in Cleanup below to prove the volume is truly gone, not just guess based on the PVC disappearing.

## Step 4: Delete and recreate the Pod - the data should survive

```
kubectl delete pod pvc-demo
kubectl apply -f ~/course/pod-with-pvc.yaml
kubectl exec pvc-demo -- cat /data/file.txt
```

This is the entire point of using a PVC instead of the Pod's own local filesystem: your data outlives any single Pod's lifecycle.

## Verify It Worked

- The AWS CLI `describe-volumes` call in Step 3 returns a real volume with `State: in-use`
- Your file's content prints correctly in Step 4, even though the Pod was fully deleted and recreated in between

## Common Mistakes

- **Deleting only the Pod and considering the exercise done** - The PVC (and its underlying EBS volume) survives Pod deletion BY DESIGN - that's the feature. You must explicitly delete the PVC too, or the EBS volume keeps billing indefinitely.
- **Verifying cleanup by tag instead of by the specific volume ID** - EBS volumes created this way aren't guaranteed to carry an Owner tag the same reliable way directly-created AWS resources do - the dependable check is the exact `$VOLUME_ID` you captured in Step 3, not a tag filter.
- **Forgetting this is the one exercise where 'nothing running' does not mean 'nothing billing'** - A volume with zero Pods attached still costs money every month until deleted - this is the single most important cleanup step in this whole set of exercises to get right.

## Cleanup (do this now, not later)

```
kubectl delete -f ~/course/pod-with-pvc.yaml
kubectl delete -f ~/course/pvc.yaml
# Verify the SPECIFIC volume from Step 3 is gone - more reliable than a tag filter,
# since dynamically-provisioned volumes aren't guaranteed to inherit an Owner tag:
sleep 15
aws ec2 describe-volumes --volume-ids $VOLUME_ID --region us-east-1
```

Then complete the mandatory checklist: **`cleanup-checklists/09-cleanup-checklist.md`**.
