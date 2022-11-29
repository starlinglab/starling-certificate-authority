# CA Workflow

<!-- Insert disclaimer , offline computer, etc -->

The following tutorial will walk you through creating a CA using the openssl command line tools.

- [Requirements](#requirements)
- [Steps](#steps)
  - [Creating a Root CA](#creating-a-root-ca)
  - [Creating an Intermediate CA](#creating-an-intermediate-ca)
  - [Signing the intermediate certificate with the root CA](#signing-the-intermediate-certificate-with-the-root-ca)
  - [Creating a Project Certificate](#creating-a-project-certificate)
  - [🙈 Loading Private Cert onto Yubikey](#-loading-private-cert-onto-yubikey)
- [Resources](#resources)
  - [🙈 Resetting Yubikey PIV to factory settings](#-resetting-yubikey-piv-to-factory-settings)
  - [Encrypting Private Key](#encrypting-private-key)
  - [Decrypting Private Key](#decrypting-private-key)
  - [Creating QR code](#creating-qr-code)
  - [🙈 Storing the QR code offline](#-storing-the-qr-code-offline)

## Requirements

- [OpenSSL 3.0](https://github.com/openssl/openssl)
- [QREncode](https://github.com/fukuchi/libqrencode)
- [GNU sed](https://www.gnu.org/software/sed/)

## Steps

### Creating a Root CA

[What is a root certificate?](https://en.wikipedia.org/wiki/Root_certificate)

1. Prepare the directory structure
    ```sh
    # Create a directory where the CA will reside
    mkdir CA
    
    # Create subdirectories for issued certificates, issued certificate revocation lists,
    # new certificates, private keys and certificate signing requests respectively
    mkdir CA/certs CA/crl CA/newcerts CA/private CA/csr
    
    # Limit access to your private keys
    chmod 700 CA/private
    
    # Create a file database for certificates
    touch CA/index.txt
    
    # Create serial and CRL numbers
    echo 1000 > CA/serial
    echo 1000 > CA/crlnumber
    ```

1. Configure openssl (_optionally_, you can change the values in the section labelled `DEFAULTS`)
    ```sh
    # Download the openssl.cnf from this repository
    curl --no-progress-meter --output CA/openssl.cnf https://gist.githubusercontent.com/galargh/a8e25193d66679a97b84395c07fd56de/raw/034746e8972763fa0c127d1343d85529acdd1fff/openssl.cnf

    # Update the path to the CA directory
    sed -i'' "s#/home/user#$(pwd)#g" CA/openssl.cnf
    ```

1. Generate private key
    ```sh
    openssl ecparam -genkey -name prime256v1 -out CA/private/ca.key.pem
    ```

1. Generate a self-signed certificate (**important**, the description is a required field; you can use something like `Starling Lab Root CA` for example)
    ```sh
    openssl req -new -x509 -days 7300 -config CA/openssl.cnf -key CA/private/ca.key.pem -sha256 -extensions v3_ca -out CA/certs/ca.cert.pem
    ```

1. Verify the validity of the certificate
    ```sh
    openssl x509 -in CA/certs/ca.cert.pem -text
    ```
    
### Creating an Intermediate CA

[What is an intermediate certificate?](https://en.wikipedia.org/wiki/Public_key_certificate#Types_of_certificate)

1. Prepare the directory structure
    ```sh
    # Create a directory where the CA will reside
    mkdir intermediate-stg
    
    # Create subdirectories for issued certificates, issued certificate revocation lists,
    # new certificates, private keys and certificate signing requests respectively
    mkdir intermediate-stg/certs intermediate-stg/crl intermediate-stg/newcerts intermediate-stg/private intermediate-stg/csr
    
    # Limit access to your private keys
    chmod 700 intermediate-stg/private
    
    # Create a file database for certificates
    touch intermediate-stg/index.txt
    
    # Create serial and CRL numbers
    echo 1000 > intermediate-stg/serial
    echo 1000 > intermediate-stg/crlnumber
    ```

1. Configure openssl (_optionally_, you can change the values in the section labelled `DEFAULTS`)
    ```sh
    # Download the openssl.cnf from this repository
    curl --no-progress-meter --output intermediate-stg/openssl.cnf https://gist.githubusercontent.com/galargh/a8e25193d66679a97b84395c07fd56de/raw/034746e8972763fa0c127d1343d85529acdd1fff/openssl.cnf
    
    # Update the path to the intermediate-stg directory
    sed -i'' "s#/home/user/CA#$(pwd)/intermediate-stg#g" intermediate-stg/openssl.cnf

    # Update the policy used by the certificate
    sed -i'' "s#= policy_strict#= policy_loose#g" intermediate-stg/openssl.cnf

    # Update the pem file names
    sed -i'' "s#ca.key.pem#intermediate-stg.key.pem#g" intermediate-stg/openssl.cnf
    sed -i'' "s#ca.cert.pem#intermediate-stg.cert.pem#g" intermediate-stg/openssl.cnf
    sed -i'' "s#ca.crl.pem#intermediate-stg.crl.pem#g" intermediate-stg/openssl.cnf
    ```

1. Generate private key
    ```sh
    openssl ecparam -genkey -name prime256v1 -out intermediate-stg/private/intermediate-stg.key.pem
    ```

1. Generate a signing request (**important**, the description is a required field; you can use something like `Starling Lab Intermediate CA` for example)
    ```sh
    openssl req -sha256 -new -config intermediate-stg/openssl.cnf -key intermediate-stg/private/intermediate-stg.key.pem -out intermediate-stg/csr/intermediate-stg.csr
    ```

### Signing the intermediate certificate with the root CA

You can sign the intermediate CSR multiple times with different root CA for cross singing.

CSR files do not contain any sensative information and can be exchanged easily. 
However, it is required to set up a mechanism for verification if a file was not interfered with during such exchange.

1. Sign the intermediate CSR with root CA (you'll be asked for confirmation twice)
    ```sh
    openssl ca -config CA/openssl.cnf -extensions v3_intermediate_ca -days 1825 -notext -md sha512 -create_serial -in intermediate-stg/csr/intermediate-stg.csr -out intermediate-stg/certs/intermediate-stg.cert.pem
    ```

1. Verify the validity of the certificate
    ```sh
    openssl x509 -in intermediate-stg/certs/intermediate-stg.cert.pem -text
    ```
    
1. Verify the trust in the certificate
    ```sh
    openssl verify -show_chain -CAfile CA/certs/ca.cert.pem intermediate-stg/certs/intermediate-stg.cert.pem
    ```

### Creating a Project Certificate

1. Choose a new name for the project certificate and expose it as a variable
    ```sh
    keyname=test-cert
    ```
    
1. Generate private key
    ```sh
    openssl ecparam -genkey -name prime256v1 -out intermediate-stg/private/$keyname.key.pem
    ```

1. Generate a signing request (**important**, the description is a required field; you can use something like `Starling Lab Project Certificate` for example)
    ```sh
    openssl req -sha256 -new -config intermediate-stg/openssl.cnf -key intermediate-stg/private/$keyname.key.pem -out intermediate-stg/csr/$keyname.csr
    ```

1. Sign the request with the Intermediate CA (_optionally_, you can use a fixed date, e.g. `-enddate 20230701120000Z`, instead of number of day, i.e. `-days 375`; you'll be asked for confirmation twice)
    ```sh
    openssl ca -config intermediate-stg/openssl.cnf -extensions doc_cert -days 375 -notext -md sha256 -in intermediate-stg/csr/$keyname.csr -out intermediate-stg/certs/$keyname.cert.pem
    ```

1. Verify the validity of the certificate
    ```sh
    openssl x509 -in intermediate-stg/certs/$keyname.cert.pem -text
    ```
    
1. Verify the trust in the certificate
    ```sh
    openssl verify -show_chain -CAfile CA/certs/ca.cert.pem -untrusted intermediate-stg/certs/intermediate-stg.cert.pem  intermediate-stg/certs/$keyname.cert.pem
    ```

1. Create certificate bundle
    ```sh
    cat intermediate-stg/certs/$keyname.cert.pem > intermediate-stg/certs/$keyname.cert.bundle.pem
    cat intermediate-stg/certs/intermediate-stg.cert.pem >> intermediate-stg/certs/$keyname.cert.bundle.pem
    # if any more intermediate certs were used, bundle them here
    ```
    
1. Verify the trust in the certificate bundle
    ```sh
    openssl verify -show_chain -CAfile CA/certs/ca.cert.pem -untrusted intermediate-stg/certs/$keyname.cert.bundle.pem  intermediate-stg/certs/$keyname.cert.bundle.pem
    ```

### 🙈 Loading Private Cert onto Yubikey

If you wish to have the root CA private key live on a YubiKey you will need to install some packages and libraries. You will also need to install a management key that would be used to modify the certificates in the future. If this key is lost a factory reset would need to be done.

#### Yubikey setup

You can put the private key of a CA onto a Yubikey and use the key to sign certificates. This protects the key from being leaked by accident.

Install required packages and start service
```sh
sudo apt install libengine-pkcs11-openssl opensc-pkcs11 pcscd yubico-piv-tool ykcs11 gnutls-bin
stemctl start  pcscd.socket
```
Configure management key on Yubikey. You can store key offline
```sh
export key=$(hexdump -vn24 -e'6/4 "%08X" 1 "\n"' /dev/urandom)
yubico-piv-tool -a set-mgm-key -n $key
echo $key
```

If you get `Failed authentication with the application.` this may mean the key was initalized before. If you need to reset it see factory reset instructions in resources.

#### Import CA Certificate into Yubikey

Import private key and certificate into the Yubikey (Position 9c also known as slot 02)
```sh
yubico-piv-tool -k$key -a import-key -s 9c -i private/ca.key.pem 
yubico-piv-tool -k$key -a import-certificate -s9c -i certs/ca.cert.pem
```
You can now backup and delete the secret key from `private/ca.key.pem`
```sh
pin=$(export LC_CTYPE=C; dd if=/dev/urandom 2>/dev/null | tr -cd '[:digit:]' | fold -w6 | head -1)
puk=$(export LC_CTYPE=C; dd if=/dev/urandom 2>/dev/null | tr -cd '[:digit:]' | fold -w8 | head -1)
yubico-piv-tool -achange-pin -P123456 -N${pin}
yubico-piv-tool -achange-puk -P12345678 -N${puk}
```

#### Find Yubikey URI
You will need to find the Yubikey URI. This is unique to the physical key and needs to be looked up again if a different hardware devices is used.

List all the devices
```sh
p11tool --list-all
```

Copy the one that looks like the Yubikey IE `pkcs11:model=PKCS%2315%20emulated;manufacturer=piv_II;serial=bd619d59419d430e;token=Starling%20Lab%20ROOT%20CA`

List all the keys on the YubiKey. Value is taken from previous command.

```sh
p11tool --login --list-all pkcs11:model=PKCS%2315%20emulated;manufacturer=piv_II;serial=bd619d59419d430e;token=Starling%20Lab%20ROOT%20CA
```

Find `Object 2` as this is where the key was imported into (Slot 02). Copy the URL Listed there IE `pkcs11:model=PKCS%2315%20emulated;manufacturer=piv_II;serial=bd619d59419d430e;token=Starling%20Lab%20ROOT%20CA;id=%02;object=Certificate%20for%20Digital%20Signature;type=cert`

#### Update openssl.cnf to refrence Yubikey

Edit openssl.cnf, replace value of `private_key` to the URL from above command

```sh
private_key = pkcs11:model=PKCS%2315%20emulated;manufacturer=piv_II;serial=bd619d59419d430e;token=Starling%20Lab%20ROOT%20CA;id=%02;object=Certificate%20for%20Digital%20Signature;type=cert`
```

You can now remove the private key from the `secrets` folder.

_NOTE_: When signing with Yubikey (`openssl ca` commands) you will need to tell openssl to use the pkcs11 engine by including the following `-engine pkcs11` and  `-keyform engine` flags

IE:
```sh
openssl ca -config openssl.cnf -engine pkcs11 -keyform engine -extensions v3_intermediate_ca -days 1825 -notext -md sha512 -create_serial -in  csr/intermediate-stg2022.csr -out certs/intermediate-stg2022.cert.pem
```

## Resources

### 🙈 Resetting Yubikey PIV to factory settings

```sh
yubico-piv-tool -averify-pin -P471112
yubico-piv-tool -averify-pin -P471112
yubico-piv-tool -averify-pin -P471112
yubico-piv-tool -averify-pin -P471112
yubico-piv-tool -achange-puk -P471112 -N6756789
yubico-piv-tool -achange-puk -P471112 -N6756789
yubico-piv-tool -achange-puk -P471112 -N6756789
yubico-piv-tool -achange-puk -P471112 -N6756789
yubico-piv-tool -areset
yubico-piv-tool -aset-chuid
yubico-piv-tool -aset-ccc
```

### Encrypting Private Key

```sh
openssl ec -aes-256-cbc -out intermediate-stg.key.encrypted.pem -in intermediate-stg/private/intermediate-stg.key.pem
```

### Decrypting Private Key

```sh
openssl ec -in intermediate-stg.key.encrypted.pem -out intermediate-stg/private/intermediate-stg.key.decrypted.pem
```

### Creating QR code

```sh
qrencode -r intermediate-stg.key.encrypted.pem -o intermediate-stg.key.encrypted.png
```

### 🙈 Storing the QR code offline

1. Create QR Code in PNG format.
1. Place the image onto a standard `Ledger` with a 24-word size recovery sheet.
1. Clearly indicate the certificate name, the fact it is a encrypted private key, the number of copies there are and which copy this is.
1. Store the password for the certificate in shared password manager.

#### Example

![](https://i.imgur.com/YruQeKA.png)
