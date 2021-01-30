It is also possible to create a RHCOS image from a cloudformation template. The following instructions are for creating a new machine leveraging a cloudformation template.


Start by copying the worker-template.ign file to worker.ign

```
cp worker-template.ign worker.ign
```

Update the worker.ign file, replacing the example sshkey with your public key and save the file.

After updating the worker.ign file with YOUR ssh public key, encode the file in base64 for use in the cloudformation template.

`cat workerpayload.ign | base64`

Copy the rhcos-template.json file to rhcos.json and edit rhcos.json. 

```
cp rhcos-template.json rhcos.json
```

Put the base64 output in the ignitionImage key section and save the file.

Run the command to create your machine from the cloudformation template file:

```
aws cloudformation create-stack --stack-name rhcos-scan \
     --template-body file://rhcos.yaml \
     --parameters file://rhcos.json
aws cloudformation describe-stacks --stack-name rhcos-scan
```

record the Private IP address from the describe-stacks command and use that to ssh to the host

ssh core@\<Priavate IP\>

