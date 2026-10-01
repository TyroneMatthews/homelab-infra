# create PVC + pod
kubectl apply -f pvc-test.yaml
kubectl wait --for=condition=Ready pod/pvc-test --timeout=120s

# write data
kubectl exec pvc-test -- sh -c 'echo "hello $(date)" > /data/test.txt'
kubectl exec pvc-test -- cat /data/test.txt

# delete the pod (NOT the PVC)
kubectl delete pod pvc-test

# new pod, same PVC
kubectl apply -f pvc-test.yaml   # recreates only the pod; PVC is unchanged
kubectl wait --for=condition=Ready pod/pvc-test --timeout=120s

# verify
kubectl exec pvc-test -- cat /data/test.txt