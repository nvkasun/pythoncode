{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowUseApprovedRDSKMSKeysViaRDS",
      "Effect": "Allow",
      "Action": [
        "kms:DescribeKey",
        "kms:Decrypt",
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



{
  "Sid": "AllowGitHubRunnerUseOfRDSKMSKeyViaRDS",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::<ENGINEERING_ACCOUNT_ID>:role/<GITHUB_ACTIONS_RUNNER_ROLE_NAME>"
  },
  "Action": [
    "kms:DescribeKey",
    "kms:Decrypt",
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
