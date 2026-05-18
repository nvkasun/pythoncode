{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowUseApprovedRDSKMSKeysViaRDS",
      "Effect": "Allow",
      "Action": [
        "kms:DescribeKey",
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncryptFrom",
        "kms:ReEncryptTo",
        "kms:GenerateDataKey",
        "kms:GenerateDataKeyWithoutPlaintext",
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "rds.eu-west-1.amazonaws.com",
          "aws:ResourceTag/ApplicationName": "CloudFactory",
          "aws:ResourceTag/Environment": "dev"
        }
      }
    }
  ]
}


Allow-RDS-PerformanceInsights-KMS



{
  "Sid": "AllowLZSelfServiceRunnerUseOfRDSPerformanceInsightsKey",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::668311713531:role/LZ-SelfService-Runner"
  },
  "Action": [
    "kms:DescribeKey",
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncryptFrom",
    "kms:ReEncryptTo",
    "kms:GenerateDataKey",
    "kms:GenerateDataKeyWithoutPlaintext",
    "kms:CreateGrant",
    "kms:ListGrants",
    "kms:RevokeGrant"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "kms:ViaService": "rds.eu-west-1.amazonaws.com"
    }
  }
}



-------------------------------------------------------------------new


{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowUseRDSPerformanceInsightsKMSKey",
      "Effect": "Allow",
      "Action": [
        "kms:DescribeKey",
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncryptFrom",
        "kms:ReEncryptTo",
        "kms:GenerateDataKey",
        "kms:GenerateDataKeyWithoutPlaintext",
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant"
      ],
      "Resource": "arn:aws:kms:eu-west-1:668311713531:key/d8c93bbf-5676-4da4-911a-30ae6a48782f",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "rds.eu-west-1.amazonaws.com"
        }
      }
    }
  ]
}



{
  "Sid": "AllowLZSelfServiceRunnerUseOfRDSPerformanceInsightsKey",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::668311713531:role/LZ-SelfService-Runner"
  },
  "Action": [
    "kms:DescribeKey",
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncryptFrom",
    "kms:ReEncryptTo",
    "kms:GenerateDataKey",
    "kms:GenerateDataKeyWithoutPlaintext",
    "kms:CreateGrant",
    "kms:ListGrants",
    "kms:RevokeGrant"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "kms:ViaService": "rds.eu-west-1.amazonaws.com"
    }
  }
}
----------------------------------------------------------------

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowUseAuroraKMSViaRDS",
      "Effect": "Allow",
      "Action": [
        "kms:DescribeKey",
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncryptFrom",
        "kms:ReEncryptTo",
        "kms:GenerateDataKey",
        "kms:GenerateDataKeyWithoutPlaintext"
      ],
      "Resource": "arn:aws:kms:eu-west-1:668311713531:key/d8c93bbf-5676-4da4-911a-30ae6a48782f",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "rds.eu-west-1.amazonaws.com"
        }
      }
    },
    {
      "Sid": "AllowCreateGrantForAuroraKMS",
      "Effect": "Allow",
      "Action": [
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant"
      ],
      "Resource": "arn:aws:kms:eu-west-1:668311713531:key/d8c93bbf-5676-4da4-911a-30ae6a48782f",
      "Condition": {
        "Bool": {
          "kms:GrantIsForAWSResource": "true"
        }
      }
    }
  ]
}



{
  "Sid": "AllowLZSelfServiceRunnerUseAuroraKMSViaRDS",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::668311713531:role/LZ-SelfService-Runner"
  },
  "Action": [
    "kms:DescribeKey",
    "kms:Encrypt",
    "kms:Decrypt",
    "kms:ReEncryptFrom",
    "kms:ReEncryptTo",
    "kms:GenerateDataKey",
    "kms:GenerateDataKeyWithoutPlaintext"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "kms:ViaService": "rds.eu-west-1.amazonaws.com"
    }
  }
},
{
  "Sid": "AllowLZSelfServiceRunnerCreateGrantForAuroraKMS",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::668311713531:role/LZ-SelfService-Runner"
  },
  "Action": [
    "kms:CreateGrant",
    "kms:ListGrants",
    "kms:RevokeGrant"
  ],
  "Resource": "*",
  "Condition": {
    "Bool": {
      "kms:GrantIsForAWSResource": "true"
    }
  }
}
