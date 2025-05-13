# ACDC Issuance

<div class="alert alert-primary">
<b>🎯 OBJECTIVE</b><hr>
Demonstrate the process of issuing an Authentic Chained Data Container (ACDC), also known as a Verifiable Credential (VC), from an Issuer to a Holder using the Issuance and Presentation Exchange (IPEX) protocol. This involves setting up identities, defining a credential schema, creating the credential, and securely transferring it.
</div> 

## Setting up Holder and Issuer

Authentic Chained Data Containers (ACDCs) are KERI's implementation of verifiable credentials. They allow entities (controllers) to issue cryptographically verifiable statements (credentials) about subjects (often other controllers) according to predefined structures (schemas). These credentials can then be presented to verifiers to prove claims.

This notebook focuses on the issuance part of the workflow: how an Issuer creates an ACDC and securely delivers it to a Holder. We will use the Issuance and Presentation Exchange (IPEX) protocol, which defines a standard way to handle the offering and acceptance of ACDCs between KERI controllers.

Before any credential issuance can happen, both the entity that will issue the credential (the Issuer) and the entity that will receive it (the Holder) need to have their own KERI Autonomic Identifiers established. This involves initializing their respective keystores and creating their AIDs.

### Holder setup

This should be familiar by now; you initialize a keystore and incept the AID for the holder of the credential. 


```python
from scripts.utils import exec

holder_keystore_name = "holder_ks"

holder_keystore_passcode = exec("kli passcode generate")

holder_keystore_salt = exec("kli salt")
# Alias for our non-transferable AID
holder_aid = "holder_aid"

# Initialize the keystore
!kli init --name {holder_keystore_name} --passcode {holder_keystore_passcode} --salt {holder_keystore_salt}\
    --config-dir ./config \
    --config-file keystore_init_config.json

!kli incept --name {holder_keystore_name} --alias {holder_aid} --passcode {holder_keystore_passcode} \
    --file ./config/aid_inception_config.json
```

    KERI Keystore created at: /usr/local/var/keri/ks/holder_ks
    KERI Database created at: /usr/local/var/keri/db/holder_ks
    KERI Credential Store created at: /usr/local/var/keri/reg/holder_ks
    	aeid: BC1B1oOCFjC6tPSI5CQEZEzqhBNF-hLBrZ5WgCkRVtbG
    
    Loading 3 OOBIs...
    http://witness-demo:5642/oobi/BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha/controller?name=Wan&tag=witness succeeded
    http://witness-demo:5643/oobi/BLskRTInXnMxWaGqcpSyMgo0nYbalW99cGZESrz3zapM/controller?name=Wes&tag=witness succeeded
    http://witness-demo:5644/oobi/BIKKuvBwpmDVA4Ds-EpL5bt9OqPzWPja2LigFYZN2YfX/controller?name=Wil&tag=witness succeeded
    Waiting for witness receipts...
    Prefix  ELRWbQXb4YPUgzRFSDVQBVRrPiKV_pIbbpCxBmSGVvee
    	Public key 1:  DJ2V7WpVs9fHQ98bQk3QwtOk9jye9vjNGAbj2ZMB9ddN
    


### Issuer setup

Perform the same initialization and inception process for the Issuer. The Issuer is the entity that will create and sign the credential.


```python
# Issuer setup
issuer_keystore_name = "issuer_ks"

issuer_keystore_passcode = exec("kli passcode generate")

issuer_keystore_salt = exec("kli salt")
# Alias for our non-transferable AID
issuer_aid = "issuer_aid"

# Initialize the keystore
!kli init --name {issuer_keystore_name} --passcode {issuer_keystore_passcode} --salt {issuer_keystore_salt}\
    --config-dir ./config \
    --config-file keystore_init_config.json

!kli incept --name {issuer_keystore_name} --passcode {issuer_keystore_passcode} --alias {issuer_aid}  \
    --file ./config/aid_inception_config.json
```

    KERI Keystore created at: /usr/local/var/keri/ks/issuer_ks
    KERI Database created at: /usr/local/var/keri/db/issuer_ks
    KERI Credential Store created at: /usr/local/var/keri/reg/issuer_ks
    	aeid: BHOPHa-zLH4vVduI-t1mF8CeFXvGBnfYfwoA_lOAkuDo
    
    Loading 3 OOBIs...
    http://witness-demo:5642/oobi/BBilc4-L3tFUnfM_wJr4S4OJanAv_VmF_dJNN6vkf2Ha/controller?name=Wan&tag=witness succeeded
    http://witness-demo:5643/oobi/BLskRTInXnMxWaGqcpSyMgo0nYbalW99cGZESrz3zapM/controller?name=Wes&tag=witness succeeded
    http://witness-demo:5644/oobi/BIKKuvBwpmDVA4Ds-EpL5bt9OqPzWPja2LigFYZN2YfX/controller?name=Wil&tag=witness succeeded
    Waiting for witness receipts...
    Prefix  ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby
    	Public key 1:  DHOOp9J_PKVqhxqbjtKwnmqbIUybGYKCuD-cEIjv9hNl
    


### Issuer/Holder OOBI 

For the Issuer to send a credential to the Holder, they first need to discover each other's key state (KEL - Key Event Log). This is done using Out-of-Band Introductions (OOBIs). Each controller generates an OOBI URL pointing to one of its witnesses. They then exchange these OOBIs (typically through a non-KERI channel, hence "out-of-band") and resolve them. Resolving an OOBI allows a controller to securely fetch and verify the KEL of the other controller via the specified witness.   

(This time we skip the optional challenge/response step for convenience, which would normally be used to further authenticate that the controllers possess the private keys associated with their AIDs.)


```python
holder_oobi_gen = f"kli oobi generate --name {holder_keystore_name} --alias {holder_aid} --passcode {holder_keystore_passcode} --role witness"
holder_oobi = exec(holder_oobi_gen)

issuer_oobi_gen = f"kli oobi generate --name {issuer_keystore_name} --alias {issuer_aid} --passcode {issuer_keystore_passcode} --role witness"
issuer_oobi = exec(issuer_oobi_gen)

!kli oobi resolve --name {holder_keystore_name} --passcode {holder_keystore_passcode} --oobi-alias {issuer_aid} \
    --oobi {issuer_oobi}

!kli oobi resolve --name {issuer_keystore_name} --passcode {issuer_keystore_passcode} --oobi-alias {holder_aid}\
    --oobi {holder_oobi}
```

    http://witness-demo:5642/oobi/ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby/witness resolved
    http://witness-demo:5642/oobi/ELRWbQXb4YPUgzRFSDVQBVRrPiKV_pIbbpCxBmSGVvee/witness resolved


### Issuer Registry

To issue ACDCs, the Issuer needs a Credential Registry. Think of this as a dedicated log, managed by the Issuer's AID, specifically for tracking the status (like issuance and revocation) of the credentials it manages. The registry itself has an identifier (a SAID, derived from its inception event) and its history is maintained in a Transaction Event Log (TEL). Anchoring events from the TEL into the Issuer's main KEL ensures the registry's state changes are secured by the Issuer's controlling keys.   

Use `kli vc registry incept` to create a new registry named `issuer_registry` controlled by the Issuer's AID (issuer_aid).


```python
issuer_registry_name="issuer_registry"

!kli vc registry incept --name {issuer_keystore_name} \
    --passcode {issuer_keystore_passcode} \
    --alias {issuer_aid} \
    --registry-name {issuer_registry_name}
```

    Waiting for TEL event witness receipts
    Sending TEL events to witnesses
    Registry:  issuer_registry(ENnRx9nRnQj99FU2NJ1TRZ9YhGUGjuTacByEHvH9fcWM) 
    	created for Identifier Prefix:  ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby


To query the status of a registry, use the command `kli vc registry status`. This shows the registry's SAID, its current sequence number (how many events have occurred in its TEL), and the controlling AID.


```python
!kli vc registry status --name {issuer_keystore_name} \
    --passcode {issuer_keystore_passcode} \
    --registry-name {issuer_registry_name} 
```

    Registry:  ENnRx9nRnQj99FU2NJ1TRZ9YhGUGjuTacByEHvH9fcWM
    Seq No.  0
        Controlling Identifier:  ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby
    
    Backers: Not supported
    Events:	Interaction Allowed
    


## Schema preparation

As seen before, you need a schema to issue credentials. A schema defines the structure and data types for a specific kind of credential, ensuring consistency and enabling automated validation. 

For this example, we have prepared a schema to simulate an access pass for an event. It defines the expected attributes for such a pass

```json
"eventName": {
  "description": "The event name",
  "type": "string"
},
"accessLevel": {
  "description": "staff/speaker/attendee",
  "type": "string",
  "enum": [
    "staff",
    "speaker",
    "attendee"
  ]
},
"validDate": {
  "description": "Valid date yyyy-mm-dd",
  "type": "string"
}
```

To see the full schema, click **[here](config/schemas/event_pass_schema.bak.json)**.  

### SAIDifying the schema

You might notice the  **[schema file](config/schemas/event_pass_schema.bak.json)** doesn't have SAIDs+ embedded within it yet. As done before, use the helper script to perform this process, taking the `event_pass_schema.bak.json` file as input and outputting the SAIDified version to `event_pass_schema.json`. 

Additionally, capture the top-level SAID of the schema using the function `get_schema_said`


```python
# Imports and Utility functions
from scripts.saidify import process_schema_file, get_schema_said

process_schema_file("config/schemas/event_pass_schema.bak.json", "config/schemas/event_pass_schema.json") 

schema_said = get_schema_said("config/schemas/event_pass_schema.json")

print(schema_said)

```

    Successfully wrote processed data to config/schemas/event_pass_schema.json
    EGUPiCVO73M9worPwR3PfThAtC0AJnH5ZgwsXf6TzbVK


You can view the complete, SAIDified schema definition **[here](config/schemas/event_pass_schema.json)**. Notice the `$id` fields are now populated with SAIDs.


### Making the Schema Discoverable

After the schema has been SAIDified, it needs to be made available so that any party needing it (like the Issuer and Holder) can retrieve and verify it. In KERI, discovery often happens via OOBIs.

For this tutorial, we use a simple service called **vLEI-server**, which acts as a basic schema cache. It's essentially an HTTP file server pre-loaded with the SAIDified schema. It exposes an OOBI URL endpoint for each schema it holds, allowing controllers to resolve the schema using its SAID.

This has already been prepared, so you can query the schema directly from the vLEI-server using its SAID via a simple HTTP request.


```python
!curl -s http://vlei-server:7723/oobi/{schema_said}
```

    {"$id":"EGUPiCVO73M9worPwR3PfThAtC0AJnH5ZgwsXf6TzbVK","$schema":"http://json-schema.org/draft-07/schema#","title":"EventPass","description":"Event Pass Schema","type":"object","credentialType":"EventPassCred","version":"1.0.0","properties":{"v":{"description":"Credential Version String","type":"string"},"d":{"description":"Credential SAID","type":"string"},"u":{"description":"One time use nonce","type":"string"},"i":{"description":"Issuer AID","type":"string"},"ri":{"description":"Registry SAID","type":"string"},"s":{"description":"Schema SAID","type":"string"},"a":{"oneOf":[{"description":"Attributes block SAID","type":"string"},{"$id":"ELppbffpWEM-uufl6qpVTcN6LoZS2A69UN4Ddrtr_JqE","description":"Attributes block","type":"object","properties":{"d":{"description":"Attributes data SAID","type":"string"},"i":{"description":"Issuee AID","type":"string"},"dt":{"description":"Issuance date time","type":"string","format":"date-time"},"eventName":{"description":"The event name","type":"string"},"accessLevel":{"description":"staff/speaker/attendee","type":"string","enum":["staff","speaker","attendee"]},"validDate":{"description":"Valid date yyyy-mm-dd","type":"string"}},"additionalProperties":false,"required":["d","i","dt","eventName","accessLevel","validDate"]}]}},"additionalProperties":false,"required":["v","d","i","ri","s","a"]}

### Resolve Schema oobi

Both the Issuer and the Holder need to know the schema definition to understand the structure of the credential being issued/received. They achieve this by resolving the schema's OOBI URL provided by the `vLEI-server`


```python
schema_oobi_alias = "schema_oobi"
schema_oobi = f"http://vlei-server:7723/oobi/{schema_said}"

!kli oobi resolve --name {holder_keystore_name} --passcode {holder_keystore_passcode} --oobi-alias {schema_oobi_alias} \
    --oobi {schema_oobi}

!kli oobi resolve --name {issuer_keystore_name} --passcode {issuer_keystore_passcode} --oobi-alias {schema_oobi_alias}\
    --oobi {schema_oobi}
```

    http://vlei-server:7723/oobi/EGUPiCVO73M9worPwR3PfThAtC0AJnH5ZgwsXf6TzbVK resolved
    http://vlei-server:7723/oobi/EGUPiCVO73M9worPwR3PfThAtC0AJnH5ZgwsXf6TzbVK resolved


## Prepare and Create Credential

With the Issuer and Holder identities established, connected via OOBI, the Issuer registry created, and the schema SAIDified and resolved, we are now ready to actually create the credential.

### Credential data

To create a specific credential instance, you must provide the actual values for the attributes defined in the schema (eventName, accessLevel, validDate). These values are typically provided in a separate data file (e.g., JSON), which is then referenced by the `kli vc create command`.

Here's the data for the specific event pass we're issuing:


```python
!cat config/credential_data/event_pass_cred_data.json
```

    {
        "eventName":"GLEIF Summit",
        "accessLevel":"staff",
        "validDate":"2026-10-01"
    }

### Create credential

The `kli vc create` command is used by the Issuer to generate the ACDC. Let's break down the parameters:   
- `--name`, `--passcode`, and `--alias`: Identify the Issuer's keystore and the specific AID within that keystore that will act as the credential issuer.
- `--registry-name`: Specifies the credential registry the Issuer will use to manage this credential's lifecycle (issuance/revocation).   
- `--schema`: Provides the SAID of the ACDC schema (event_pass_schema.json in this case) that defines the structure of this credential.   
- `--recipient`: Specifies the AID of the entity the credential is about (the Holder/subject).   
- `--data`: Points to the file containing the specific attribute values for this credential instance (@ indicates it's a file path).   
- `-time`: Provides an issuance timestamp for the credential.

Executing this command creates the credential, generates its SAID, and records an issuance event in the specified registry's TEL, anchoring it to the Issuer's KEL.


```python
time = exec("kli time")

!kli vc create --name {issuer_keystore_name} --passcode {issuer_keystore_passcode} \
    --alias {issuer_aid} \
    --registry-name {issuer_registry_name} \
    --schema {schema_said} \
    --recipient {holder_aid} \
    --data "@./config/credential_data/event_pass_cred_data.json" \
    --time {time}
```

    Waiting for TEL event witness receipts
    Sending TEL events to witnesses
    ED81zZyPPXlarZ4cSvo_78XNyXbhLHv9xpBFLS7OzJp9 has been created.


### Display issued credential

The kli vc list command allows the Issuer to see the credentials they have issued. The `--issued` flag specifies listing issued credentials, and `--verbose` shows the full credential content.


```python
!kli vc list  --name {issuer_keystore_name} \
    --passcode {issuer_keystore_passcode} \
    --issued \
    --verbose
```

    Current issued credentials for issuer_aid (ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby):
    
    Credential #1: ED81zZyPPXlarZ4cSvo_78XNyXbhLHv9xpBFLS7OzJp9
        Type: EventPass
        Status: Issued [92m✔[0m
        Issued by ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby
        Issued on 2025-05-13T20:29:45.466023+00:00
        Full Credential:
    	{
    	  "v": "ACDC10JSON0001c4_",
    	  "d": "ED81zZyPPXlarZ4cSvo_78XNyXbhLHv9xpBFLS7OzJp9",
    	  "i": "ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby",
    	  "ri": "ENnRx9nRnQj99FU2NJ1TRZ9YhGUGjuTacByEHvH9fcWM",
    	  "s": "EGUPiCVO73M9worPwR3PfThAtC0AJnH5ZgwsXf6TzbVK",
    	  "a": {
    	    "d": "ED29R-aseYbKbwSuVHn1IvNsZk8_Chvxy1iYaD9fLEKs",
    	    "i": "ELRWbQXb4YPUgzRFSDVQBVRrPiKV_pIbbpCxBmSGVvee",
    	    "dt": "2025-05-13T20:29:45.466023+00:00",
    	    "eventName": "GLEIF Summit",
    	    "accessLevel": "staff",
    	    "validDate": "2026-10-01"
    	  }
    	}


Notice the structure of the credential JSON:   
- `v`: Version string for the ACDC format and serialization.
- `d`: The SAID of this specific credential instance.
- `i`: The Issuer's AID.
- `ri`: The SAID of the credential registry used.
- `s`: The SAID of the schema used.
- `a`: The attributes block, containing:
- `d`: The SAID of the attributes block itself.
- `i`: The Issuee's (Holder's) AID.
- `dt`: The issuance date/time.
- `eventName`, `accessLevel`, and `validDate`: The specific data for this credential.

### Get credential SAID

You need to retrieve the credential's SAID (`d` field) to use it as the identifier in the subsequent IPEX steps. Use `kli vc list` again, but add the `--said` flag to return only the SAID of the matching credential(s).


```python
get_credential_said = f"kli vc list --name {issuer_keystore_name} --passcode {issuer_keystore_passcode} --alias {issuer_aid} --issued --said --schema {schema_said}"
credential_said=exec(get_credential_said)

print(credential_said)
```

    ED81zZyPPXlarZ4cSvo_78XNyXbhLHv9xpBFLS7OzJp9


## IPEX Credential Issuance Flow

At this point, the ACDC has been created and recorded in the Issuer's registry, but it still resides with the Issuer. To securely transfer the credential to the Holder, we use the **Issuance and Presentation Exchange (IPEX) protocol**. IPEX defines a message-based workflow for offering, accepting, or rejecting credentials.

### Issuer Initiates IPEX Exchange (Credential Offer)

The Issuer (acting as the "Discloser" in IPEX terms) initiates the transfer by sending a grant message to the Holder (the "Disclosee"). The `kli ipex grant` command facilitates this.   

The parameters are:

- `--name`, `--passcode`, and `--alias`: Identify the Issuer's keystore and AID.
- `--said`: The SAID of the credential being granted (which we retrieved in the previous step).
- `--recipient`: The AID of the Holder who should receive the credential offer.
- `--time`: A timestamp for the grant message.

This command sends an IPEX grant message containing the credential data to the Holder's KERI mailbox (managed via witnesses).


```python
time = exec("kli time")

!kli ipex grant \
    --name {issuer_keystore_name} \
    --passcode {issuer_keystore_passcode} \
    --alias {issuer_aid} \
    --said {credential_said} \
    --recipient {holder_aid} \
    --time {time}
```

    Sending message EHYEQcSHhM77cS9PV99sNLxKA3L9Km5Bjblo6icPdVz7 to ELRWbQXb4YPUgzRFSDVQBVRrPiKV_pIbbpCxBmSGVvee
    ... grant message sent


### Issuer Checks Sent Message (Optional)

This step isn't strictly required for the workflow but allows the Issuer to view the IPEX messages they have sent using `kli ipex list --sent`. This can be useful for debugging or tracking the state of exchanges. It shows the GRANT message SAID and details about the credential offered.


```python
!kli ipex list --name {issuer_keystore_name} \
    --passcode {issuer_keystore_passcode} \
    --alias {issuer_aid} \
    --sent
```

    
    Sent IPEX Messages:
    
    GRANT - SAID: EHYEQcSHhM77cS9PV99sNLxKA3L9Km5Bjblo6icPdVz7
    Credential ED81zZyPPXlarZ4cSvo_78XNyXbhLHv9xpBFLS7OzJp9:
        Type: EventPass
        Status: Issued [92m✔[0m
        Issued by ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby
        Issued on 2025-05-13T20:29:45.466023+00:00
        Already responded? No [91m✘[0m
    


### Holder Polls Mailbox and Gets IPEX Message SAID

The Holder needs to check their KERI mailbox for incoming messages. The `kli ipex list --poll` command checks for new IPEX messages. Use `--said` again to extract just the SAID of the received grant message, which is needed for the next step (accepting the credential).


```python
get_ipex_said=f"kli ipex list --name {holder_keystore_name} --passcode {holder_keystore_passcode} --alias {holder_aid} --poll --said"
ipex_said=exec(get_ipex_said)

print(ipex_said)

```

    EHYEQcSHhM77cS9PV99sNLxKA3L9Km5Bjblo6icPdVz7


### Holder Checks Received Message (Optional)

Similar to the Issuer checking sent messages, the Holder can use `kli ipex list` (without `--sent`) to view received IPEX messages. This confirms receipt of the GRANT offer from the Issuer.


```python
!kli ipex list --name {holder_keystore_name} \
    --passcode {holder_keystore_passcode} \
    --alias {holder_aid} 
```

    
    Received IPEX Messages:
    
    GRANT - SAID: EHYEQcSHhM77cS9PV99sNLxKA3L9Km5Bjblo6icPdVz7
    Credential ED81zZyPPXlarZ4cSvo_78XNyXbhLHv9xpBFLS7OzJp9:
        Type: EventPass
        Status: Issued [92m✔[0m
        Issued by ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby
        Issued on 2025-05-13T20:29:45.466023+00:00
        Already responded? No [91m✘[0m
    


### Holder accept credential 

Now that the Holder has received the `grant` message (identified by `ipex_said`), they can choose to accept the credential using the `kli ipex admit` command.   

- `--name`, `--passcode`, and `--alias`: Identify the Holder's keystore and AID.
- `--said`: The SAID of the grant message being admitted.
- `--time`: A timestamp for the admit message.  

This sends an `admit` message back to the Issuer, confirming acceptance. The Holder's KERI controller automatically verifies the credential against its schema and the Issuer's KEL upon admitting it, and then stores the credential securely.


```python
time = exec("kli time")

!kli ipex admit \
    --name {holder_keystore_name} \
    --passcode {holder_keystore_passcode} \
    --alias {holder_aid} \
    --said {ipex_said} \
    --time {time}
```

    Sending admit message to ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby
    ... admit message sent


### Holder Checks Sent Message (Optional)

The Holder can optionally check their sent IPEX messages to confirm the `ADMIT` message was sent.


```python
!kli ipex list --name {holder_keystore_name} \
    --passcode {holder_keystore_passcode} \
    --alias {holder_aid} \
    --sent
```

    
    Sent IPEX Messages:
    
    ADMIT - SAID: ECdVE-DYFNkMqrG95u1jCTLBP-J4OPl6xoTH1QgY0wRP
    Admitted message SAID: EHYEQcSHhM77cS9PV99sNLxKA3L9Km5Bjblo6icPdVz7
    Credential ED81zZyPPXlarZ4cSvo_78XNyXbhLHv9xpBFLS7OzJp9:
        Type: EventPass
        Status: Accepted [92m✔[0m
    


### Holder displays credential

The issuance process is complete! The Holder now possesses the verifiable credential. They can view it using kli vc list --verbose. The output will be similar to when the Issuer listed the issued credential, confirming the Holder has successfully received and stored the ACDC.


```python
!kli vc list --name {holder_keystore_name} \
    --passcode {holder_keystore_passcode} \
    --verbose
```

    Current received credentials for holder_aid (ELRWbQXb4YPUgzRFSDVQBVRrPiKV_pIbbpCxBmSGVvee):
    
    Credential #1: ED81zZyPPXlarZ4cSvo_78XNyXbhLHv9xpBFLS7OzJp9
        Type: EventPass
        Status: Issued [92m✔[0m
        Issued by ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby
        Issued on 2025-05-13T20:29:45.466023+00:00
        Full Credential:
    	{
    	  "v": "ACDC10JSON0001c4_",
    	  "d": "ED81zZyPPXlarZ4cSvo_78XNyXbhLHv9xpBFLS7OzJp9",
    	  "i": "ELBgsUufu6_S_5Qw8DElU4hrjE2D6jAdB_9glfYgTDby",
    	  "ri": "ENnRx9nRnQj99FU2NJ1TRZ9YhGUGjuTacByEHvH9fcWM",
    	  "s": "EGUPiCVO73M9worPwR3PfThAtC0AJnH5ZgwsXf6TzbVK",
    	  "a": {
    	    "d": "ED29R-aseYbKbwSuVHn1IvNsZk8_Chvxy1iYaD9fLEKs",
    	    "i": "ELRWbQXb4YPUgzRFSDVQBVRrPiKV_pIbbpCxBmSGVvee",
    	    "dt": "2025-05-13T20:29:45.466023+00:00",
    	    "eventName": "GLEIF Summit",
    	    "accessLevel": "staff",
    	    "validDate": "2026-10-01"
    	  }
    	}


<div class="alert alert-primary">
<b>📝 SUMMARY</b><hr>
The ACDC issuance process using KERI and IPEX:
<ol>
<li><b>Setup:</b> Issuer and Holder established identities (AIDs) and connected via OOBI resolution.</li>
<li><b>Registry:</b> Issuer created a credential registry (managed via a TEL) to track credential status.</li>
<li><b>Schema:</b> An ACDC schema was defined (using JSON Schema) and made verifiable through SAIDification. It was made discoverable via a simple caching server (vLEI-server) and resolved by both parties using its OOBI.</li>
<li><b>Creation:</b> Issuer created the specific ACDC instance using kli vc create, providing data conforming to the schema and linking it to the registry.</li>
<li><b>IPEX Transfer:</b>
<ul>
<li>Issuer offered the credential using kli ipex grant (sending a GRANT message).</li>
<li>Holder received the offer (polling with kli ipex list --poll).</li>
<li>Holder accepted the credential using kli ipex admit (sending an ADMIT message).</li>
</ul>
</li>
<li><b>Result:</b> Holder successfully received and stored the verifiable credential (ACDC).</li>
</ol>
This process ensures that credentials are not only structured and verifiable against a schema but are also securely issued and transferred between authenticated KERI identities.
</div>


```python
from scripts.utils import clear_keri
clear_keri()
```

    Proceeding with deletion of '/usr/local/var/keri/' without confirmation.
    ✅ Successfully removed: /usr/local/var/keri/



```python

```
