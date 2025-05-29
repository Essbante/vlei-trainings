# KLI Operations: Creating and Managing Delegated AIDs 

<div class="alert alert-primary">
<b>🎯 OBJECTIVE</b><hr>
Understand the concept of delegated AIDs, where one Autonomic Identifier (AID), the delegator, grants specific authority to another AID, the delegate. This notebook demonstrates how to create and manage delegated AIDs using the KERI Command Line Interface (KLI), covering:
<ul>
<li>The two-step cooperative process of delegated inception.</li>
<li>Performing delegated key rotation.</li>
<li>Examining the Key Event Logs (KELs) of both the delegator and the delegate to understand how the delegation is anchored and verified.</li>

## Introduction to Delegated AIDs

In KERI, delegation is a powerful mechanism that allows one AID (the delegator) to authorize another AID (the delegate) to perform certain actions. This is achieved through a cooperative cryptographic process where both parties participate in establishing the relationship. 

Key aspects of KERI delegation include:

- Cooperative Establishment: The creation (inception) and subsequent management (e.g., rotation) of a delegated AID requires actions from both the delegate (initiating the request) and the delegator (confirming and anchoring the event). 
- Cryptographic Binding: The delegated AID's prefix is a self-addressing identifier (SAID) derived from its own delegated inception event.  This inception event, in turn, includes the delegator's AID, creating a cryptographic link.
- Anchoring: The delegator anchors the delegation by including a "delegated event seal" in one of its own key events.  This seal contains the delegate's AID, the sequence number of the delegated event, and a digest of that delegated event. 
- Delegated Authority: The delegator typically retains ultimate establishment control authority, while the delegate might be authorized for specific non-establishment events or further, limited delegations. 
- Hierarchical Structures: Delegation can be applied recursively, enabling the creation of complex hierarchical key management structures. 

This notebook will walk through the KLI commands to perform delegated inception and delegated rotation, illustrating how these concepts are put into practice.

## Initial Setup

First, we'll set up the necessary keystores and a primary AID for the delegator. We will also initialize a keystore for the delegate. For simplicity in this notebook, passcodes for keystores are omitted using the `--nopasscode` flag.

The `keystore_init_config.json` file is used to pre-configure the keystores with witness information.


```python
# Imports and Utility functions
from scripts.utils import exec, exec_bg, clear_keri
from scripts.utils import pr_continue, pr_title, pr_message
clear_keri()

pr_title("Initializing keystores")

# Delegate Keystore
delegate_keystore="delegate_keystore"
delegate_salt= exec("kli salt")

!kli init --name {delegate_keystore} \
    --nopasscode \
    --salt {delegate_salt} \
    --config-dir ./config \
    --config-file keystore_init_config.json

# Delegator Keystore
delegator_keystore="delegator_keystore"
delegator_salt=exec("kli salt")

!kli init --name {delegator_keystore} \
    --nopasscode \
    --salt {delegator_salt} \
    --config-dir ./config \
    --config-file keystore_init_config.json

pr_title("Incepting delegator AID")

# Delegator AID
delegator_alias = "delegator_alias"

!kli incept --name {delegator_keystore} \
    --alias {delegator_alias} \
    --icount 1 \
    --isith 1 \
    --ncount 1 \
    --nsith 1 \
    --wits BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha \
    --wits BLskRTInXnMxWaGqcpSyMgo0nYbalW99cGZESrz3zapM \
    --wits BIKKuvBwpmDVA4Ds-EpL5bt9OqPzWPja2LigFYZN2YfX \
    --toad 2 \
    --transferable

pr_title("Generating OOBIs")

# OOBI Exchange
# Delegator generates an OOBI for its AID
delegator_oobi = exec(f"kli oobi generate --name {delegator_keystore} --alias {delegator_alias} --role witness")
print("Delegator OOBI: " + delegator_oobi)

pr_title("Resolving OOBIs")
# Delegate's keystore resolves the Delegator's OOBI
!kli oobi resolve --name {delegate_keystore} \
    --oobi-alias {delegator_alias} \
    --oobi {delegator_oobi}

pr_continue()
```

    Proceeding with deletion of '/usr/local/var/keri/' without confirmation.
    ✅ Successfully removed: /usr/local/var/keri/



<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Initializing keystores</span>
</pre>



    KERI Keystore created at: /usr/local/var/keri/ks/delegate_keystore
    KERI Database created at: /usr/local/var/keri/db/delegate_keystore
    KERI Credential Store created at: /usr/local/var/keri/reg/delegate_keystore
    
    Loading 3 OOBIs...
    http://witness-demo:5642/oobi/BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha/controller?name=Wan&tag=witness succeeded
    http://witness-demo:5643/oobi/BLskRTInXnMxWaGqcpSyMgo0nYbalW99cGZESrz3zapM/controller?name=Wes&tag=witness succeeded
    http://witness-demo:5644/oobi/BIKKuvBwpmDVA4Ds-EpL5bt9OqPzWPja2LigFYZN2YfX/controller?name=Wil&tag=witness succeeded
    KERI Keystore created at: /usr/local/var/keri/ks/delegator_keystore
    KERI Database created at: /usr/local/var/keri/db/delegator_keystore
    KERI Credential Store created at: /usr/local/var/keri/reg/delegator_keystore
    
    Loading 3 OOBIs...
    http://witness-demo:5642/oobi/BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha/controller?name=Wan&tag=witness succeeded
    http://witness-demo:5643/oobi/BLskRTInXnMxWaGqcpSyMgo0nYbalW99cGZESrz3zapM/controller?name=Wes&tag=witness succeeded
    http://witness-demo:5644/oobi/BIKKuvBwpmDVA4Ds-EpL5bt9OqPzWPja2LigFYZN2YfX/controller?name=Wil&tag=witness succeeded



<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Incepting delegator AID</span>
</pre>



    Waiting for witness receipts...
    Prefix  EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1
    	Public key 1:  DGpJPebl-NON52cCAK7FwknNbCtkDkxl1rdBlSCqAJub
    



<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Generating OOBIs</span>
</pre>



    Delegator OOBI: http://witness-demo:5642/oobi/EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1/witness



<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Resolving OOBIs</span>
</pre>



    http://witness-demo:5642/oobi/EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1/witness resolved



<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #008000; text-decoration-color: #008000">You can continue ✅</span>

</pre>



    Delegator OOBI: http://witness-demo:5642/oobi/ELHViCL-OJjigPoy3fVkLzFjRkjo3ZuERTveluafMdRW/witness



<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Resolving OOBIs</span>
</pre>



    http://witness-demo:5642/oobi/ELHViCL-OJjigPoy3fVkLzFjRkjo3ZuERTveluafMdRW/witness resolved



<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #008000; text-decoration-color: #008000">You can continue ✅</span>

</pre>



## Creating Delegated Identifiers
Delegation is a multi-step process involving both the entity wishing to become a delegate and the entity granting the delegation (the delegator).

### Step 1: Delegate Incepts Proxy
The delegate first needs an AID that can initiate the delegation request. This "proxy" AID is a regular AID within the delegate's keystore. It will be used to manage the unpublished keys of the new delegated AID until the delegator confirms the delegation.


```python
# Delegate proxy
pr_title("Incepting delegate proxy AID")

# This AID is in the delegate's keystore and is used to initiate the delegation request.
delegate_proxy_alias = "delegate_proxy_alias"
!kli incept --name delegate_keystore \
    --alias delegate_proxy_alias \
    --icount 1 \
    --isith 1 \
    --ncount 1 \
    --nsith 1 \
    --wits BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha \
    --wits BLskRTInXnMxWaGqcpSyMgo0nYbalW99cGZESrz3zapM \
    --wits BIKKuvBwpmDVA4Ds-EpL5bt9OqPzWPja2LigFYZN2YfX \
    --toad 2 \
    --transferable

pr_continue()
```


<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Incepting delegate proxy AID</span>
</pre>



    Waiting for witness receipts...
    Prefix  EKjPT6AlddiVmu3-Tlr987MlLxAhFhSc2IVaWk8zlD8K
    	Public key 1:  DPgaVau8J1-ORM6eWbYbPkxP_dm1BL45mJO3HEuB1FEG
    



<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #008000; text-decoration-color: #008000">You can continue ✅</span>

</pre>




<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #008000; text-decoration-color: #008000">You can continue ✅</span>

</pre>



### Step 2: Delegate request delegated AID Inception

Now, the delegate uses its proxy AID to request the inception of a new, delegated AID.

- `--name` and `--alias`: Define the keystore and the alias for the new delegated AID being created.
- `--delpre`: Specifies the prefix of the AID that will be delegating authority.
- `--proxy`: Specifies the alias of the AID within the `delegate_keystore` that is making the request and will temporarily manage the keys for `delegate_alias`.

The `kli incept --delpre` command will initiate the process and then wait for the delegator to confirm. We run this in the background (`exec_bg`) because it will pause.


```python
pr_title("Incepting delegated AID")

delegator_pre = exec(f"kli aid --name {delegator_keystore} --alias {delegator_alias}")
pr_message("Delegator prefix: " + delegator_pre)

delegate_alias = "delegate_alias"
# Incept delegate. Note --delpre and --proxy parameters
# The command runs in the background since it waits for the delegator's confirmation
# exec_bg (execute in background) does that. Output is sent to a log file. 
# This pattern of exec_bg is repeated throughout the notebook 
command = f"""
kli incept --name {delegate_keystore} \
    --alias {delegate_alias} \
    --icount 1 \
    --isith 1 \
    --ncount 1 \
    --nsith 1 \
    --wits BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha \
    --toad 1 \
    --transferable \
    --delpre {delegator_pre} \
    --proxy {delegate_proxy_alias} > ./logs/delegate_incept.log
"""

exec_bg(command)

pr_continue()
```


<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Incepting delegated AID</span>
</pre>




<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">Delegator prefix: EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1</span>
</pre>



    Command 
    kli incept --name delegate_keystore     --alias delegate_alias     --icount 1     --isith 1     --ncount 1     --nsith 1     --wits BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha     --toad 1     --transferable     --delpre EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1     --proxy delegate_proxy_alias > ./logs/delegate_incept.log
     started with PID: 14304



<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #008000; text-decoration-color: #008000">You can continue ✅</span>

</pre>



### Step 3: Delegator confirms delegation 
The delegator now needs to confirm the delegation request. The `kli delegate confirm` command checks for pending delegation requests for the specified delegator AID and, if `--auto` is used, automatically approves them. This action creates an event in the delegator's KEL that anchors the delegate's inception event. 


```python
# Delegator confirmation
pr_title("Confirming delegation")

command = f"""
kli delegate confirm --name {delegator_keystore} \
    --alias {delegator_alias} \
    --auto
"""

output = exec(command, True)

pr_message(output)

pr_continue()
```


<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Confirming delegation</span>
</pre>




<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">[</span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">'Delegagtor Prefix  EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1'</span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">, </span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">'Delegate </span>
<span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a inception Anchored at Seq. No.  1'</span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">, </span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">'Delegate </span>
<span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a inception event committed.'</span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">]</span>
</pre>




<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #008000; text-decoration-color: #008000">You can continue ✅</span>

</pre>



Now, let's examine the status of the newly created delegated AID.


```python
pr_title(f"Delegated AID status")
!kli status --name delegate_keystore --alias delegate_alias --verbose
```


<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Delegated AID status</span>
</pre>



    Alias: 	delegate_alias
    Identifier: EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a
    Seq No:	0
    Delegated Identifier
        Delegator:  EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1 [92m✔ Anchored[0m
    
    
    Witnesses:
    Count:		1
    Receipts:	1
    Threshold:	1
    
    Public Keys:	
    	1. DJZO6kCWGWXvLVSyiTnkI3E1qnDRF86-QBPp-67OY2-P
    
    
    Witnesses:	
    	1. BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha
    
    {
     "v": "KERI10JSON00018d_",
     "t": "dip",
     "d": "EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a",
     "i": "EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a",
     "s": "0",
     "kt": "1",
     "k": [
      "DJZO6kCWGWXvLVSyiTnkI3E1qnDRF86-QBPp-67OY2-P"
     ],
     "nt": "1",
     "n": [
      "EHos1UUVdj-9Mu5DEvFR_c-47NtcFrnc7LH30rBVloK_"
     ],
     "bt": "1",
     "b": [
      "BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha"
     ],
     "c": [],
     "a": [],
     "di": "EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1"
    }
    


Key observations from the delegate's status:

- `Delegated Identifier`: This line confirms it's a delegated AID.
- `Delegator: <Some Prefix> ✔ Anchored`: This shows the delegator's prefix and confirms that the delegation has been successfully anchored in the delegator's KEL.
- In the JSON event data:
  - `"t": "dip"`: This signifies a Delegated Inception Event. This is the establishment event for the delegated AID.
  - `"di"`: This field contains the prefix of the Delegator AID. It cryptographically links this delegated AID back to its delegator.

The prefix of a delegated AID is a SAID of its own `dip` event, which includes the delegator's AID. This creates the strong cryptographic binding characteristic of KERI delegation. 

## Rotating Delegated Identifiers

Rotating the keys of a delegated AID also follows a cooperative, two-step process, similar to its inception. The delegate initiates the rotation, and the delegator must confirm it.

The delegate uses `kli rotate` with the`--proxy` parameter. This command is run in the background as it waits for the delegator's confirmation. The delegator confirms the delegated rotation with `kli delegate confirm`. This action creates a new anchoring event in the delegator's KEL for the delegate's rotation.


```python
pr_title(f"Rotating delegated AID")

command = f"""
kli rotate --name {delegate_keystore} \
    --alias {delegate_alias} \
    --proxy {delegate_proxy_alias}
"""
exec_bg(command)

command = f"""
kli delegate confirm --name {delegator_keystore} \
    --alias {delegator_alias} \
    --auto
"""
output = exec(command, True)

# Show the output of the background processes
pr_message(f"Rotation")
pr_message(output)

pr_continue()
```


<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Rotating delegated AID</span>
</pre>



    Command 
    kli rotate --name delegate_keystore     --alias delegate_alias     --proxy delegate_proxy_alias
     started with PID: 14313



<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">Rotation</span>
</pre>




<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">[</span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">'Delegagtor Prefix  EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1'</span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">, </span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">'Delegate </span>
<span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a rotation Anchored at Seq. No.  2'</span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">, </span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">'Delegate </span>
<span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a rotation event committed.'</span><span style="color: #5f5fff; text-decoration-color: #5f5fff; font-weight: bold">]</span>
</pre>




<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #008000; text-decoration-color: #008000">You can continue ✅</span>

</pre>



Now, let's examine the status of the delegate AID after the rotation.


```python
pr_title(f"Delegated AID status")
!kli status --name delegate_keystore --alias delegate_alias --verbose
```


<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Delegated AID status</span>
</pre>



    Alias: 	delegate_alias
    Identifier: EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a
    Seq No:	1
    Delegated Identifier
        Delegator:  EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1 [92m✔ Anchored[0m
    
    
    Witnesses:
    Count:		1
    Receipts:	1
    Threshold:	1
    
    Public Keys:	
    	1. DJjHlJKFDz0o68V63XIR88u2BDWZr5ARAs76Dop2xJko
    
    
    Witnesses:	
    	1. BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha
    
    {
     "v": "KERI10JSON00018d_",
     "t": "dip",
     "d": "EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a",
     "i": "EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a",
     "s": "0",
     "kt": "1",
     "k": [
      "DJZO6kCWGWXvLVSyiTnkI3E1qnDRF86-QBPp-67OY2-P"
     ],
     "nt": "1",
     "n": [
      "EHos1UUVdj-9Mu5DEvFR_c-47NtcFrnc7LH30rBVloK_"
     ],
     "bt": "1",
     "b": [
      "BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha"
     ],
     "c": [],
     "a": [],
     "di": "EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1"
    }
    
    {
     "v": "KERI10JSON000160_",
     "t": "drt",
     "d": "EEcfMiEyr4TdfkWK1FR6ZNtfP5-qSejoovKvzWD5cJc7",
     "i": "EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a",
     "s": "1",
     "p": "EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a",
     "kt": "1",
     "k": [
      "DJjHlJKFDz0o68V63XIR88u2BDWZr5ARAs76Dop2xJko"
     ],
     "nt": "1",
     "n": [
      "EJ1nr1WrKW8s5wJFrbL7JnmzpDC3Alwn65wVp3KCFXPM"
     ],
     "bt": "1",
     "br": [],
     "ba": [],
     "a": []
    }
    


Observations from the delegate's KEL after rotation:
- `Seq No: 1`: The sequence number has incremented.
- A new event has been added to the KEL with `"t": "drt"`. This signifies a Delegated Rotation Event. It's also an establishment event.
- The public keys `k` and next key digest `n` have changed, reflecting the rotation.
- The delegate's AID prefix `i` remains the same.

## Understanding the Delegator's KEL
Let's now examine the delegator's KEL to see how these delegation operations are recorded and anchored.


```python
pr_title(f"Delegator AID status")
!kli status --name delegator_keystore --alias delegator_alias --verbose
```


<pre style="white-space:pre;overflow-x:auto;line-height:normal;font-family:Menlo,'DejaVu Sans Mono',consolas,'Courier New',monospace"><span style="color: #0000ff; text-decoration-color: #0000ff; font-weight: bold">Delegator AID status</span>
</pre>



    Alias: 	delegator_alias
    Identifier: EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1
    Seq No:	2
    
    Witnesses:
    Count:		3
    Receipts:	3
    Threshold:	3
    
    Public Keys:	
    	1. DAdOW5ILCom8II4IqzTiXBj5hUAk9B4aC3KINOlcbGCd
    
    
    Witnesses:	
    	1. BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha
    	2. BLskRTInXnMxWaGqcpSyMgo0nYbalW99cGZESrz3zapM
    	3. BIKKuvBwpmDVA4Ds-EpL5bt9OqPzWPja2LigFYZN2YfX
    
    {
     "v": "KERI10JSON0001b7_",
     "t": "icp",
     "d": "EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1",
     "i": "EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1",
     "s": "0",
     "kt": "1",
     "k": [
      "DGpJPebl-NON52cCAK7FwknNbCtkDkxl1rdBlSCqAJub"
     ],
     "nt": "1",
     "n": [
      "EPz8nyDj7YCQ7D9i7Quqo7bMlf3PNayt1UVjizjOzh6X"
     ],
     "bt": "2",
     "b": [
      "BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha",
      "BLskRTInXnMxWaGqcpSyMgo0nYbalW99cGZESrz3zapM",
      "BIKKuvBwpmDVA4Ds-EpL5bt9OqPzWPja2LigFYZN2YfX"
     ],
     "c": [],
     "a": []
    }
    
    {
     "v": "KERI10JSON0001cf_",
     "t": "rot",
     "d": "ENRA2L1d9ocXx391UyLlLnBx9Xukw_Ck02y4JO5g9nCi",
     "i": "EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1",
     "s": "1",
     "p": "EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1",
     "kt": "1",
     "k": [
      "DN0GKt2uJ4TtQDsWlypRiT9RU890wKgcIjYU9Hw5tZRB"
     ],
     "nt": "1",
     "n": [
      "EM2fAQECUHDNU1asCTYurGDawEdH2YLTXm7E8xWIpH96"
     ],
     "bt": "3",
     "br": [],
     "ba": [],
     "a": [
      {
       "i": "EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a",
       "s": "0",
       "d": "EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a"
      }
     ]
    }
    
    {
     "v": "KERI10JSON0001cf_",
     "t": "rot",
     "d": "EKEu2mGjm_GV69-BSMOXX3pfcOms-agDZk_GK2PRbd0-",
     "i": "EPS6ruHeTeQDkhnYlPvaJ3YuT1mzWdooUO4Afl_82HH1",
     "s": "2",
     "p": "ENRA2L1d9ocXx391UyLlLnBx9Xukw_Ck02y4JO5g9nCi",
     "kt": "1",
     "k": [
      "DAdOW5ILCom8II4IqzTiXBj5hUAk9B4aC3KINOlcbGCd"
     ],
     "nt": "1",
     "n": [
      "ELSXvY0BNhEw-BBxUFihpCbtth2UnnKyzRdbGquFJ6Ld"
     ],
     "bt": "3",
     "br": [],
     "ba": [],
     "a": [
      {
       "i": "EPc6dLcAtnkmOxKzXiHL8n8dHr4JL-ZtqmQsmXPM671a",
       "s": "1",
       "d": "EEcfMiEyr4TdfkWK1FR6ZNtfP5-qSejoovKvzWD5cJc7"
      }
     ]
    }
    


Key observations from the delegator's KEL:

- Sequence Number `s: "1"` (Rotation Event):
    - This event was created when the delegator confirmed the delegated inception.
    - The `a` (anchors/payload) array contains a delegated event seal: 
      - `"i"`: The prefix of the delegate AID.
      - `"s": "0"`: The sequence number of the delegate's event being anchored (the `dip` event at sequence 0).
      - `"d"`: The SAID (digest) of the delegate's `dip` event.
- Sequence Number `s: "2"` (Rotation Event):
  - This event was created when the delegator confirmed the delegated rotation.
  - The `a` array contains another delegated event seal:
      - `"i"`: The prefix of the delegate AID.
      - `"s": "1"`: The sequence number of the delegate's event being anchored (the drt event at sequence 1).
      - `"d"`: The SAID (digest) of the delegate's drt event.

These seals in the delegator's KEL are the cryptographic proof that the delegator authorized the delegate's inception and rotation events.  Conversely, the delegated AID's `dip` event also contains a di field pointing to the delegator, and its establishment events (like `dip` and `drt`) implicitly include a delegating event location seal that refers back to the specific event in the delegator's KEL that authorized them (though not explicitly shown in the simplified `kli status` output for the delegate, this is part of the full event structure).  This creates the verifiable, cooperative link between the two AIDs.

<div class="alert alert-info">
<b>ℹ️ NOTE</b><hr>
The security of KERI's cooperative delegation model is robust. To illicitly create or rotate a delegated AID, an attacker would generally need to compromise keys from both the delegator and the delegate (specifically, the delegate's pre-rotated keys and the delegator's current signing keys for the anchoring event).  Furthermore, the delegator has mechanisms to recover or revoke a compromised delegation. 
</div>

<div class="alert alert-primary">
<b>📝 SUMMARY</b><hr>
KERI delegation allows an AID (delegator) to authorize another AID (delegate) for specific purposes. This is a cooperative process requiring actions from both parties.
<ul>
<li><b>Delegated Inception (<code>dip</code>):</b> The delegate initiates a request (e.g., via a proxy AID). The delegator confirms this by creating an anchoring event in its KEL, which contains a seal pointing to the delegate's <code>dip</code> event. The delegate's <code>dip</code> event includes the delegator's AID in its <code>di</code> field.  The delegate's AID prefix is a SAID of its <code>dip</code> event. </li>
<li><b>Delegated Rotation (<code>drt</code>):</b> Similar to inception, the delegate initiates the rotation, and the delegator confirms with another anchoring event in its KEL. The delegate's KEL will show a <code>drt</code> event. </li>
<li><b>Anchoring:</b> The delegator's KEL contains seals (AID, sequence number, and digest of the delegate's event) that provide verifiable proof of the authorized delegation.  This creates a strong, bi-directional cryptographic link.</li>
<li><b>Security:</b> The cooperative nature enhances security, as unauthorized delegation typically requires compromising keys from both entities. </li>
</ul>
</div>


```python

```
