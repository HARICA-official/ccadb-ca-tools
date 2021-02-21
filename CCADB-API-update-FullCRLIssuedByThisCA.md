# Procedure to get CCADB salesforceIDs for HARICA Intermediate CA Certificates and update "FullCRLIssuedByThisCA"

## Introduction
This is a collection of scripts in order to update the FullCRLIssuedByThisCA field in CCADB for Intermediate CA Certificates.

The procedure assumes that the operator: 
* doesn't know exactly which CA Certificates are disclosed to CCADB
* doesn't know the salesforceID for each CA Certificate. 

The procedure starts with creating a list of ALL intermediate CA Certificates operated by the CA, then it matches the ones that are disclosed in CCADB and then updates the FullCRLIssuedByThisCA value only for the disclosed CAs.

## Preparation steps
1. Create a file `activeCAs` which includes the "CA shortname" in each row. Most CA software uses some sort of "CA shortname" to identify the CA.
2. Create a folder to store CA Certificates in PEM format 

```
mkdir -p IntermediateCA-Certificates
```

3. Download all CA Certificates in PEM format, remove data before the encapsulation boundaries (RFC 7468).

```
CA_REPOSITORY="https://repo.harica.gr/certs"

for i in `cat activeCAs`; do
  curl -# "$CA_REPOSITORY/$i.pem" \
    | sed -n '/-----BEGIN CERTIFICATE-----/,$p' > "IntermediateCA-Certificates/$i.pem"
done
```

4. Create a file which includes each CA Certificate in one-line

```
cd IntermediateCA-Certificates
for i in `cat ../activeCAs`; do
  awk '{printf("%s", $0)} END {printf "\n"}' "$i.pem" > "$i.flat.pem"
done
cd ..
```

5. Create a file `activeCAs-flat-PEM` which includes each CA Certificate in PEM format without EOL in each row 

```
for i in `cat activeCAs`; do
  cat "IntermediateCA-Certificates/$i.flat.pem" >> activeCAs-flat-PEM
done
```

6. Create a file with the CA's SHA256 fingerprints

```
true > activeCAs-sha256-fingerprints
for i in `cat activeCAs`; do
  openssl x509 -noout -fingerprint -sha256 -in "IntermediateCA-Certificates/$i.pem" \
    | sed 's/.*=//; s/://g' >> activeCAs-sha256-fingerprints
done
```


7. Produce a json file for each Intermediate CA Certificate in folder that will be used to retrieve the corresponding salesforce recordId

```
mkdir -p json-files

paste activeCAs activeCAs-sha256-fingerprints activeCAs-flat-PEM | \
while IFS=$'\t' read -r ca fp pem; do
  cat <<JSON > "json-files/$ca.json"
{
  "PEM": "$pem",
  "SHA256": "$fp",
  "RecordType": "Intermediate Certificate"
}
JSON
done
```

8. Retrieve each CA's salesforce recordId in folder, after setting the proper value for the OAuth2 authorization token. Replace "mozilla-ccadb.csXX.force.com" with the proper sandbox FQDN. For production CCADB, use "ccadb.force.com". Make sure the "jq" package is installed.

```
TOKEN='aaaaaaaaaaaaaaaaaaaaaaaaaaaaa'

mkdir -p salesforce

for ca in `cat activeCAs`; do
  curl -H "Accept: application/json"       \
       -H "Content-Type: application/json" \
       -H "Authorization: Bearer ${TOKEN}" \
       -d @"json-files/$ca.json"           \
       https://mozilla-ccadb.csXX.force.com/services/apexrest/get/recordid \
    | jq -r .RecordId > "salesforce/$ca.id"
done
```

## Process to update "FullCRLIssuedByThisCA" value in Intermediate CA record
Mandatory fields: 
* SalesforceRecordId
* IntermediateCertificateName (usually subject:commonName)
* IntermediateCertPEM
* FullCRLIssuedByThisCA

1. Create a file `CCADB-Disclosed-Intermediates` which includes only the "CA shortname" of the disclosed CAs in CCADB we want to update, in each row
2. Get the commonName from the Intermediate CA Certificates

```
true > CCADB-Disclosed-Intermediates.CN
for i in `cat CCADB-Disclosed-Intermediates`; do
  openssl x509 -subject -noout -in "IntermediateCA-Certificates/$i.pem" \
    | awk -F'CN = ' '{print $2}' >> CCADB-Disclosed-Intermediates.CN
done
```
3. Create a file with the FullCRL URL for each disclosed Intermediate CA

```
true > CCADB-Disclosed-Intermediates.crl
for i in `cat CCADB-Disclosed-Intermediates`; do
  echo "http://crl.harica.gr/$i.crl" >> CCADB-Disclosed-Intermediates.crl
done
```

4. Create a file with the salesforceId for each disclosed Intermediate CA

```
true > CCADB-Disclosed-Intermediates.id
for i in `cat CCADB-Disclosed-Intermediates`; do
  cat salesforce/$i.id >> CCADB-Disclosed-Intermediates.id
done
```

5. Create a file with the PEM (flat) for each disclosed Intermediate CA
```
true > CCADB-Disclosed-Intermediates.pem
for i in `cat CCADB-Disclosed-Intermediates`; do
  cat IntermediateCA-Certificates/$i.flat.pem >> CCADB-Disclosed-Intermediates.pem
done
```

6. Paste the values of the 5 files into one consolidated file (mandatory values and Full CRL URL)

```
paste CCADB-Disclosed-Intermediates     \
      CCADB-Disclosed-Intermediates.id  \
      CCADB-Disclosed-Intermediates.CN  \
      CCADB-Disclosed-Intermediates.pem \
      CCADB-Disclosed-Intermediates.crl \
> update-pasted
```

7. Prepare the json files for the update

```
cat update-pasted | \
while IFS=$'\t' read -r ca id cn pem crl; do
cat <<JSON > json-files/$ca.update.json
{
    "CertificateInformation": {
        "SalesforceRecordId": "$id",
        "IntermediateCertificateName": "$cn",
        "IntermediateCertPEM": "$pem"
    },
    "PertainingToCertificatesIssued": {
       "FullCRLIssuedByThisCA": "$crl"
    }
}
JSON
done
```

8. REVIEW each json file to confirm correct values
9. Execute the update. Replace "mozilla-ccadb.csXX.force.com" with the proper sandbox FQDN. For production CCADB, use "ccadb.force.com". The command results will be stored in folder "salesforce".

```
for i in `cat CCADB-Disclosed-Intermediates`; do
  curl -H "Accept: application/json"       \
       -H "Content-Type: application/json" \
       -H "Authorization: Bearer ${TOKEN}" \
       -d @json-files/$i.update.json       \
       -o salesforce/$i.update.log         \
       https://mozilla-ccadb.csXX.force.com/services/apexrest/create/intermediatecert
done
```

10. Check the log output for possible errors

```
grep -v Success salesforce/*update.log
```

11. Verify the updated values in CCADB by connecting to CCADB Web site and confirm the updated values.
