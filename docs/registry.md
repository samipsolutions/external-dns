### TXT Registry migration to a new format ###

In order to support more record types and be able to track ownership without TXT record name clash, a new TXT record is introduced.
It contains record type it manages, e.g.:
* A record foo.example.com will be tracked with classic foo.example.com TXT record
* At the same time a new TXT record will be created a-foo.example.com

Prefix and suffix are extended with %{record_type} template where the user can control how prefixed/suffixed records should look like.

In order to maintain compatibility, both records will be maintained for some time, in order to have downgrade possibility.  
The controller will try to create the "new format" TXT records if they are not present to ease the migration from the versions < 0.12.0.

Later on, the old format will be dropped and only the new format will be kept (<record_type>-<endpoint_name>).

Cleanup will be done by controller itself.

### Encryption of TXT Records
TXT records may contain sensitive information, such as the internal ingress name or namespace, which attackers could exploit to gather information about your infrastructure. 
By encrypting TXT records, you can protect this information from unauthorized access. It is strongly recommended to encrypt all TXT records to prevent potential security breaches.

To enable encryption of TXT records, you can use the following parameters:
- `--txt-encrypt-enabled=true`
- `--txt-encrypt-aes-key=32bytesKey` (used for AES-256-GCM encryption and should be exactly 32 bytes)

Note that the key used for encryption should be a secure key and properly managed to ensure the security of your TXT records.

### Generating TXT encryption AES key
Python
```python
python -c 'import os,base64; print(base64.urlsafe_b64encode(os.urandom(32)).decode())'
```

Bash
```shell
dd if=/dev/urandom bs=32 count=1 2>/dev/null | base64 | tr -d -- '\n' | tr -- '+/' '-_'; echo
```

OpenSSL
```shell
openssl rand -base64 32 | tr -- '+/' '-_'
```

PowerShell
```powershell
# Add System.Web assembly to session, just in case
Add-Type -AssemblyName System.Web
[Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes([System.Web.Security.Membership]::GeneratePassword(32,4))).Replace("+","-").Replace("/","_")
```

Terraform
```hcl
resource "random_password" "txt_key" {
  length           = 32
  override_special = "-_"
}
```

### Manually Encrypt/Decrypt TXT Records

In some cases, you may need to edit labels generated by External-DNS, and in such cases, you can use simple Golang code to do that.

```go
package main

import (
	"fmt"
	"sigs.k8s.io/external-dns/endpoint"
)

func main() {
	key := []byte("testtesttesttesttesttesttesttest")
	encrypted, _ := endpoint.EncryptText(
		"heritage=external-dns,external-dns/owner=example,external-dns/resource=ingress/default/example",
		key,
		nil,
	)
	decrypted, _, _ := endpoint.DecryptText(encrypted, key)
	fmt.Println(decrypted)
}
```
