# Configuring MinIO with KES and Vault as KMS A Comprehensive Guide (Windows) 
Unlock the power of secure object storage by seamlessly integrating MinIO with KES (Key Management Service) and HashiCorp Vault as the Key Management Service (KMS). This step-by-step guide provides a comprehensive walkthrough, enabling you to fortify your MinIO deployment with advanced encryption capabilities.

# QuickStart: Installation Steps

## Step 1 :  Vault Installation

### 1. Generate Vault Private Key & Certificate
KES and Vault will exchange sensitive information. so need to create a private key and certificate for this ip address : 127.0.0.
#### Note:
  Follow the Step 2 for kes installation and then follow this cammand
  
	  kes identity new --key vault.key --cert vault.crt --ip "127.0.0.1" localhost
### 2. Configure Vault Server
Create the vault-config.json file to start a single vault server instance on port 8200:

    	{
    	  "api_addr": "https://127.0.0.1:8200",
    	  "backend": {
    	    "file": {
    	      "path": "vault/file" # this path will change your case 
    	    }
    	  },
    
    	  "default_lease_ttl": "168h",
    	  "max_lease_ttl": "720h",
    
    	  "listener": {
    	    "tcp": {
    	      "address": "0.0.0.0:8200",
    	      "tls_cert_file": "vault.crt",
    	      "tls_key_file": "vault.key",
    	      "tls_min_version": "tls12"
    	    }
    	  }
    	}

### 3. Start Vault Server 
follow this command to start vault server

	  vault server -config vault-config.json

### 4. Set VAULT_ADDR endpoint
The vault CLI need to know the vault endpoint:

    	set VAULT_ADDR=https://127.0.0.1:8200
    
    	set VAULT_SKIP_VERIFY=true

### 5. Initialize Vault Server 
	
    	vault operator init
    
    	Unseal Key 1: eyW/+8ZtsgT81Cb0e8OVxzJAQP5lY7Dcamnze+JnWEDT
    	Unseal Key 2: 0tZn+7QQCxphpHwTm6/dC3LpP5JGIbYl6PK8Sy79R+P2
    	Unseal Key 3: cmhs+AUMXUuB6Lzsvgcbp3bRT6VDGQjgCBwB2xm0ANeF
    	Unseal Key 4: /fTPpec5fWpGqWHK+uhnnTNMQyAbl5alUi4iq2yNgyqj
    	Unseal Key 5: UPdDVPto+H6ko+20NKmagK40MOskqOBw4y/S51WpgVy/
    	 
    	Initial Root Token: s.zaU4Gbcu0Wh46uj2V3VuUde0
    
    	Vault is initialized with 5 key shares and a key threshold of 3. Please securely
    	distribute the key shares printed above. When the Vault is re-sealed,
    	restarted, or stopped, you must supply at least 3 of these keys to unseal it
    	before it can start servicing requests.

### 6. Set the VAULT_TOKEN 
Replace the Root Token value.

    set VAULT_TOKEN=s.zaU4Gbcu0Wh46uj2V3VuUde0

### 7. Unseal Vault Server
	
  	$ vault operator unseal eyW/+8ZtsgT81Cb0e8OVxzJAQP5lY7Dcamnze+JnWEDT
  
  	$ vault operator unseal 0tZn+7QQCxphpHwTm6/dC3LpP5JGIbYl6PK8Sy79R+P2
  
  	$ vault operator unseal cmhs+AUMXUuB6Lzsvgcbp3bRT6VDGQjgCBwB2xm0ANeF

Once enough valid unseal key shares have been submitted, Vault will become unsealed and able to process requests.

### 8. Enable K/V Backend
    vault secrets enable -version=1 kv
In general, we recommend the K/V v1 engine.

### 9. Create Vault Policy
The Vault policy defines the API paths the KES server will be able to access.
create the kes-policy.hcl file and paste this lines in kes-policy.hcl.

  	path "kv/*" {
  	   capabilities = [ "create", "read", "delete" ]
  	}


The following command creates the policy at Vault:

	  vault policy write kes-policy kes-policy.hcl

### 10. Enable AppRole Authentication
The KES server will later need to authenticate to Vault.

	  vault auth enable approle

### 11. Create KES Role
The following command adds a new role kes-server at Vault:

	  vault write auth/approle/role/kes-server token_num_uses=0  secret_id_num_uses=0  period=5m

### 12. Bind Policy to Role

	  vault write auth/approle/role/kes-server policies=kes-policy

### 13. Generate AppRole ID

	  vault read auth/approle/role/kes-server/role-id 

### 14. Generate AppRole Secret

	  vault write -f auth/approle/role/kes-server/secret-id 





## Step 2 :  Kes Server Installation

### 1. Download the kes.exe file follow this url
In this repository you get the all version related packages.
	
	  https://github.com/minio/kes/releases

### Direct Download

	  https://github.com/minio/kes/releases/download/2023-11-10T10-44-28Z/kes-windows-amd64.exe

### 2. Create files for localhost ip : 127.0.0.1 

	  kes identity new --ip "127.0.0.1" --cert=public.cert --key=private.key

### 3. Create the private key, certificate for Minio.

	  kes identity new --key=client.key --cert=client.crt MinIO

### 4. This command you get the minio client.cert.

	  kes identity of client.crt

### 5. Configure the kes-config.yml file
	
     notepad D:\minio-kes-setup\kes\kes-config.yml
copy & paste in kes-config.yml file and change the values as per your according your set.

	
 	    address: 0.0.0.0:7373 # Listen on all network interfaces on port 7373

      admin:
        identity: disabled  # We disable the admin identity since we don't need it in this guide 
         
      tls:
        key: private.key    # The KES server TLS private key
        cert: public.crt    # The KES server TLS certificate
         
      policy:
        my-app: 
          allow:
          - /v1/key/create/my-key*
          - /v1/key/generate/my-key*
          - /v1/key/decrypt/my-key*
          identities:
          - 02ef5321ca409dbc7b10e7e8ee44d1c3b91e4bf6e2198befdebee6312745267b # Use the identity of your client.crt
         
      keystore:
         vault:
           endpoint: https://127.0.0.1:8200
           version:  v1 # The K/V engine version - either "v1" or "v2".
           approle:
             id:     "" # Your AppRole ID
             secret: "" # Your AppRole Secret
             retry:  15s
           status:
             ping: 10s
           tls:
             ca: vault.crt # Manually trust the vault certificate since we use self-signed certificates

### 6. Start KES Server
Now, we can start a KES server

	  kes server --config config.yml --auth off




## Step 3 : Minio Server Installation 

### 1. Install the MinIO Server 

	  https://dl.min.io/server/minio/release/windows-amd64/minio.exe

### 2. Lunch the minio server
Open command prompt or powershell and change the directory where your minio.exe is located and you can check the minio latest verions
      
      minio.exe --version
	
    	minio.exe version RELEASE.2024-01-01T16-36-33Z (commit-id=8f13c8c3bfee304e11a56076039f13e343937124)
    	Runtime: go1.21.5 windows/amd64
    	License: GNU AGPLv3 <https://www.gnu.org/licenses/agpl-3.0.html>
    	Copyright: 2015-2024 MinIO, Inc.

Note : D:/minio this path is minio can store the configuration files in this path folder. In your  case you can change the your path 
Run this command to starting minio server

	     minio server D:/minio

The minio server process prints its output to the system console, similar to the following:

    	API: http://192.0.2.10:9000  http://127.0.0.1:9000
    	RootUser: minioadmin
    	RootPass: minioadmin
    
    	Console: http://192.0.2.10:9001 http://127.0.0.1:9001
    	RootUser: minioadmin
    	RootPass: minioadmin
    
    	Command-line: https://min.io/docs/minio/linux/reference/minio-mc.html
    	   $ mc alias set myminio http://192.0.2.10:9000 minioadmin minioadmin
    
    	Documentation: https://min.io/docs/minio/linux/index.html
    
    	WARNING: Detected default credentials 'minioadmin:minioadmin', we recommend that you change these values with 'MINIO_ROOT_USER' and 'MINIO_ROOT_PASSWORD' environment variables.


### 3. In pervious kes installation, you created the four files so, two files are kes files (client.key, client.cert) and for localhost path ="127.0.0.1" (public.cert, private.key)
Now, we need to set this below commands for minio, so first close the running minio server using Ctrl+C. set this one by one in terminal.

    set MINIO_KMS_KES_ENDPOINT=https://127.0.0.1:7373
    set MINIO_KMS_KES_CERT_FILE=client.crt
    set MINIO_KMS_KES_KEY_FILE=client.key
    set MINIO_KMS_KES_KEY_NAME=minio-default-key
    set MINIO_KMS_KES_CAPATH=D:\minio-kes-setup\kes\public.cert
    set MINIO_ROOT_USER=minio
    set MINIO_ROOT_PASSWORD=minio123

Run this command, Now your MINIO_ROOT_USER, MINIO_ROOT_PASSWORD value is change as per you set the value.

	  minio server D:/data

