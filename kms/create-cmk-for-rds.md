# Create Custom Managed Key (CMK) for RDS Clusters

To manage your encrypted RDS Clusters within a Well-Architected Framework you should create a new KMS key for RDS.
Depending on your business requirements you may wish to separate certain data stores with different KMS keys.


    # Required Variables
    RDS_KMS_ALIAS="alias/rds-cmk"

    OUTPUT=$(aws kms create-key --tags TagKey=Purpose,TagValue=RDS --description "RDS CMK")
    KMS_KEY_ID=$(jq .KeyMetaData.KeyId <<< ${OUTPUT})
    aws kms create-alias --alias-name ${RDS_KMS_ALIAS} --target-key-id ${KMS_KEY_ID}

# Example Output

## kms create-key    
    {
        "KeyMetadata": {
            "AWSAccountId": "999999999999",
            "KeyId": "4aa1e0f0-5c8a-4b67-aaad-e30200bf193a",
            "Arn": "arn:aws:kms:us-east-2:999999999999:key/4aa1e0f0-5c8a-4b67-aaad-e30200bf193a",
            "CreationDate": "2021-12-12T20:10:26.339000-05:00",
            "Enabled": true,
            "Description": "RDS CMK",
            "KeyUsage": "ENCRYPT_DECRYPT",
            "KeyState": "Enabled",
            "Origin": "AWS_KMS",
            "KeyManager": "CUSTOMER",
            "CustomerMasterKeySpec": "SYMMETRIC_DEFAULT",
            "EncryptionAlgorithms": [
                "SYMMETRIC_DEFAULT"
            ]
        }
    }

# References
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/kms/create-key.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/kms/create-alias.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/kms/enable-key-rotation.html
