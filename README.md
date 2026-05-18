{
  "Sid": "AllowTerraformRunnerUseOfRDSPerformanceInsightsKey",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::668311713531:role/codebuild-aws-cloud-factory-infra_dev-service-role"
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
    "AWS": "arn:aws:iam::668311713531:role/codebuild-aws-cloud-factory-infra_dev-service-role"
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
