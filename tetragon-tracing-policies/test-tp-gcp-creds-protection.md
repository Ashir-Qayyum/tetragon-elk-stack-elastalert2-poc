Step 0: Check that Tetragon and the policy are running<br><br>

kubectl get pods -n kube-system -l app.kubernetes.io/name=tetragon<br>
kubectl get tracingpolicy<br>

The Tetragon pod should be Running, and kill-gcp-credentials-search should be listed. If Tetragon isn't installed in minikube, the policy does nothing.<br><br>

---<br>
---<br>
---<br><br>

Step 1: Watch events in one terminal<br>

kubectl exec -n kube-system ds/tetragon -c tetragon -- tetra getevents -o compact<br>

Leave this running. Add --pods <nginx-pod-name> to filter to the pod. You should see a line with security_file_open and SIGKILL whenever the policy fires.<br><br>

---
---
---<br><br>

Step 2: Create fake credential files in the nginx pod<br>

Use a second terminal. Creating the files is safe because sh and echo are not in the binary list.<br>


kubectl exec -it nginx -- bash<br>

mkdir -p /root/.config/gcloud/legacy_credentials<br>
echo "fake" > /root/.config/gcloud/credentials.db<br>
echo "fake" > /root/.config/gcloud/legacy_credentials/adc.json<br>
echo "fake" > /tmp/application_default_credentials.json<br>
echo "fake" > /tmp/service_account.json<br>
echo "hello" > /tmp/notes.txt<br>

---
---
---<br><br>

Step 3: Tests that should be killed<br><br>

Run these inside the pod. Each should print Killed, and echo $? should show 137.<br>


grep fake /root/.config/gcloud/credentials.db<br>
grep -r fake /root/.config/gcloud<br>
find /root/.config/gcloud<br>
grep fake /tmp/application_default_credentials.json<br>
grep fake /tmp/service_account.json<br>
grep x /proc/self/environ<br>

Running grep from the shell, as above, is what triggers it. A quick one-liner also works from the Mac:<br><br>

exit<br>
then,<br>
kubectl exec nginx -- grep fake /root/.config/gcloud/credentials.db<br>

---
---
---<br><br>

Step 4: Tests that should NOT be killed<br>

These show the limits described in the comments.<br>


grep hello /tmp/notes.txt                       # allowed: not a protected path<br>
cat /root/.config/gcloud/credentials.db         # allowed: cat is not in the binary list<br>
env | grep GOOGLE_APPLICATION_CREDENTIALS       # allowed: grep opens no file<br>
echo "fake" > /tmp/service_account_key.json<br>
grep fake /tmp/service_account_key.json         # allowed: name is not an exact match<br><br>

---
---
---<br><br>

Things that can go wrong<br>
Nothing gets killed: Run tetra getevents -o json and check process.binary for the grep event. If it shows something other than the paths in the list (for example /bin/grep vs /usr/bin/grep), add that exact path to matchBinaries. Also make sure Tetragon is running on the same node as the nginx pod.
Policy fails to load: Run kubectl describe tracingpolicy kill-gcp-credentials-search and read the status or error message.
Home directory: The nginx image runs as root, so /root/.config/gcloud matches. If the test user has a different home, that path isn't covered.
/environ false positives: Any grep or find reading /proc/<pid>/environ will be killed, including legitimate use. Keep that in mind on real workloads.

---
---
---<br><br>

Cleanup<br><br>


rm -rf /root/.config/gcloud /tmp/application_default_credentials.json /tmp/service_account*.json /tmp/notes.txt<br>
kubectl delete tracingpolicy kill-gcp-credentials-search    # only if you want to remove the policy<br>