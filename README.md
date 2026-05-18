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
