{
  "Sid": "AllowTerraformRunnerUseOfRDSPerformanceInsightsKey",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/<RUNNER_ROLE_NAME>"
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
  "Sid": "AllowTerraformRunnerCreateGrantForRDS",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::<ACCOUNT_ID>:role/<RUNNER_ROLE_NAME>"
  },
  "Action": [
    "kms:CreateGrant",
    "kms:ListGrants",
    "kms:RevokeGrant"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "kms:ViaService": "rds.eu-west-1.amazonaws.com"
    },
    "Bool": {
      "kms:GrantIsForAWSResource": "true"
    }
  }
}
