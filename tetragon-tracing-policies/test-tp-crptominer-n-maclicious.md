Step 0: Test whether the policy is loaded or not<br><br>

kubectl get pods -n kube-system -l app.kubernetes.io/name=tetragon<br>
kubectl get tracingpolicy<br>
kubectl describe tracingpolicy block-cryptominer-n-malicious-exec<br><br>

There should be no error in the describe. If Tetragon rejects any field, it will be visible here.<br><br>
---
---
---<br><br>

Step 1: See events in one terminal<br>

kubectl exec -n kube-system ds/tetragon -c tetragon -- tetra getevents -o compact<br>

Keep it running, SIGKILL line will be returned upon each kill<br><br>
---
---
---<br><br>

Step 2: Testing with Nginx pod (from Second Terminal)<br>

kubectl exec -it nginx -- bash<br>

Layer 1 (writable folder), there should be killed returned:<br><br>


cp /bin/sleep /tmp/t1<br>
/tmp/t1 1              # Killed<br>
echo $?                # 137<br>
cd /tmp && ./t1 1      # killed from relative path as well<br>
cp /bin/sleep /dev/shm/t2 && /dev/shm/t2 1     # Killed<br><br>

Layer 2 (miner names), there should be killed returned:<br><br>


cp /bin/sleep /root/xmrig<br>
/root/xmrig 1          # Killed<br><br>

Layer 4 (mining pool port), Killed should be returned:<br><br>


timeout 3 bash -c 'echo > /dev/tcp/1.1.1.1/3333'<br>

Here would be bash killed. Even if connect is failed, the hook is triggered even on connect try. If it is not killed, try curl --max-time 3 http://1.1.1.1:4444 as well.<br>

This should not be killed (policy should not interrupt the normal work):<br><br>


ls /tmp<br>
cp /bin/sleep /root/normal && /root/normal 1     # it should run<br>
nginx -v<br>
timeout 3 bash -c 'echo > /dev/tcp/1.1.1.1/443'  # normal port, allowed<br><br>

---
---
---<br><br>

Step 3: Layer 3 (test from Python shell)<br>

Nginx doesnot have python, so create a separate pod for testing:<br>


kubectl run pytest --image=python:3.12-slim --restart=Never -- sleep 3600<br>
kubectl exec -it pytest -- bash<br>

In this image Python is in /usr/local/bin/python3, which is in our list. From inside the Pod, run:<br>


python3 -c "import os; os.system('id')"      # This should not execute<br>
python3 -c "import os; os.system('sh -c id')"<br>
python3 -c "import subprocess; subprocess.run(['/bin/sh','-c','id'])"<br>

os.system internally launches /bin/sh, which should match /sh (or /dash on Debian-based systems). Therefore, the Python process should be terminated. However, python3 -c "print('hi')" should execute normally.<br><br>

---
---
---<br><br>

Cleanup<br><br>

kubectl delete pod pytest<br>
kubectl exec nginx -- rm -f /tmp/t1 /dev/shm/t2 /root/xmrig /root/normal<br>