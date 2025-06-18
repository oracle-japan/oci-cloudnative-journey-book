# Terraform の基礎知識

## Terrafrom による OCI Object Storage Bucket の作成

1. OCI Cloud Shell の起動

OCI Console Dashboard の上部メニューにあるアイコンをクリックして、「Cloud Shell」を選択します。OCI Cloud Shell が起動します。（図-03）

![図-03 OCI Cloud Shell の起動](images/terraform-figure-03.png)

図-03 OCI Cloud Shell の起動

2. 公開鍵と秘密鍵の作成

Cloud Shell 起動後、認証に必要となる公開鍵と秘密鍵を作成します。最初に専用のディレクトリを作成します。

```sh
$ mkdir $HOME/.oci
```

openssl コマンドを実行して秘密鍵を作成します。

```sh
$ openssl genrsa -out $HOME/.oci/terraform.pem 2048
```

作成した秘密鍵に権限を設定します。

```sh
$ chmod 600 $HOME/.oci/terraform.pem
```

公開鍵を作成します。

```sh
$ openssl rsa -pubout -in $HOME/.oci/terraform.pem -out $HOME/.oci/terraform_public.pem
```

公開鍵の内容を表示します。

```sh
$ cat $HOME/.oci/terraform_public.pem
```

4. tf ファイルの作成

専用のディレクトリを作成します。

```sh
$ mkdir tf-os
```

Cloud Service Provider (OCI) に API リクエストを送るために必要な設定ファイルを「terraform.tf」として作成します。

```sh
$ vim tf-os/terraform.tf
```
```sh
provider "oci" {
  region = var.region
}

terraform {
  required_providers {
    oci = {
      source  = "oracle/oci"
      version = "< 6.0.0"
    }
  }
}
```

事前に取得した情報を変数として利用するため、「variables.tf」として作成します。

```sh
$ vim tf-os/variables.tf
```
```sh
# Provider
variable "region" {
  description = "OCI Region Identifier. (e.g. ap-tokyo-1, ...)"
  default     = "プロビジョニングするリージョンコード"
}

variable "tenancy_ocid" {
  description = "OCID of tenancy."
  default     = "プロビジョニングするテナントのOCID"
}

variable "user_ocid" {
  description = "OCID of user."
  default     = "プロビジョニングするテナントユーザーのOCID"
}

variable "private_key_path" {
  default     = "$HOME/.oci/terraform.pem" #秘密鍵のパス
  description = "File path of private key."
}

variable "private_key" {
  default     = null
  description = "Content of private key."
}

variable "fingerprint" {
  description = "Fingerprint."
  default     = "API キーのフィンガープリント"
}

variable "compartment_ocid" {
  description = "OCID of compartment."
  default     = "プロビジョニングするコンパートメントのOCID"
}

# Object Storage
variable "objectstorage_namespace" {
  description = "Namespace of Object Storage."
  default     = "xxxxx" #任意値
}

variable "objectstorage_bucket_name" {
  description = "Name of Object Storage Bucket."
  default     = "terraform" #任意値
}
```

「main.tf」に、オブジェクトストレージサービスのバケットをプロビジョニングする定義を書きます。

```sh
$ vim tf-os/main.tf
```
```sh
resource "oci_objectstorage_bucket" "objectstorage_bucket" {
  compartment_id = var.compartment_ocid
  name           = var.objectstorage_bucket_name
  namespace      = var.objectstorage_namespace
}
```

5. init plan apply destory の実行

専用のディレクトリに移動します。

```sh
$ cd tf-os
```

初期化を実行します。

```sh
$ terraform init
```
```sh
Initializing the backend...

Initializing provider plugins...
- Reusing previous version of oracle/oci from the dependency lock file
- Using previously-installed oracle/oci v5.47.0

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

計画を実行して、プロビジョニングされるリソースをプレビューします。  
※出力の一部をマスクしています。

```sh
$ terraform plan
```
```sh
Initializing the backend...

Initializing provider plugins...
- Reusing previous version of oracle/oci from the dependency lock file
- Using previously-installed oracle/oci v5.47.0

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
xxxxxx@cloudshell:tf-os (xx-xxxxx-x)$ vim terraform.tf
xxxxxx@cloudshell:tf-os (xx-xxxxx-x)$ terraform plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # oci_objectstorage_bucket.objectstorage_bucket will be created
  + resource "oci_objectstorage_bucket" "objectstorage_bucket" {
      + access_type                  = "NoPublicAccess"
      + approximate_count            = (known after apply)
      + approximate_size             = (known after apply)
      + auto_tiering                 = (known after apply)
      + bucket_id                    = (known after apply)
      + compartment_id               = "ocid1.compartment.oc1.."
      + created_by                   = (known after apply)
      + defined_tags                 = (known after apply)
      + etag                         = (known after apply)
      + freeform_tags                = (known after apply)
      + id                           = (known after apply)
      + is_read_only                 = (known after apply)
      + kms_key_id                   = (known after apply)
      + name                         = "terraform"
      + namespace                    = "xxxxx"
      + object_events_enabled        = (known after apply)
      + object_lifecycle_policy_etag = (known after apply)
      + replication_enabled          = (known after apply)
      + storage_tier                 = (known after apply)
      + time_created                 = (known after apply)
      + versioning                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run
"terraform apply" now.
```

初期化、計画が完了したので、適用します。
※出力の一部をマスクしています。

```sh
$ terraform apply
```
```sh
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # oci_objectstorage_bucket.objectstorage_bucket will be created
  + resource "oci_objectstorage_bucket" "objectstorage_bucket" {
      + access_type                  = "NoPublicAccess"
      + approximate_count            = (known after apply)
      + approximate_size             = (known after apply)
      + auto_tiering                 = (known after apply)
      + bucket_id                    = (known after apply)
      + compartment_id               = "ocid1.compartment.oc1.."
      + created_by                   = (known after apply)
      + defined_tags                 = (known after apply)
      + etag                         = (known after apply)
      + freeform_tags                = (known after apply)
      + id                           = (known after apply)
      + is_read_only                 = (known after apply)
      + kms_key_id                   = (known after apply)
      + name                         = "terraform"
      + namespace                    = "xxxxx"
      + object_events_enabled        = (known after apply)
      + object_lifecycle_policy_etag = (known after apply)
      + replication_enabled          = (known after apply)
      + storage_tier                 = (known after apply)
      + time_created                 = (known after apply)
      + versioning                   = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.
```

「yes」と入力して実行します。

```sh
Enter a value: yes
```
```sh
oci_objectstorage_bucket.objectstorage_bucket: Creating...
oci_objectstorage_bucket.objectstorage_bucket: Creation complete after 1s [id=n/orasejapan/b/terraform]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

リソースが作成されたことを確認します。  
※出力の一部をマスクしています。

```sh
$ terraform show
```
```sh
# oci_objectstorage_bucket.objectstorage_bucket:
resource "oci_objectstorage_bucket" "objectstorage_bucket" {
    access_type           = "NoPublicAccess"
    approximate_count     = "0"
    approximate_size      = "0"
    auto_tiering          = "Disabled"
    bucket_id             = "ocid1.bucket.oc1.xx-xxxxx-xx."
    compartment_id        = "ocid1.compartment.oc1.."
    created_by            = "ocid1.user.oc1.."
    defined_tags          = {}
    etag                  = "5c39bd10-f705-48f6-a9ca-00ed480ac661"
    freeform_tags         = {}
    id                    = "n/xxxxx/b/terraform"
    is_read_only          = false
    name                  = "terraform"
    namespace             = "xxxxx"
    object_events_enabled = false
    replication_enabled   = false
    storage_tier          = "Standard"
    time_created          = "2025-01-08 10:21:50.068 +0000 UTC"
    versioning            = "Disabled"
}
```

terraform apply を実行したディレクトリには、 terraform.tfstate というファイルが作成されています。terraform.tfstate は、 Terraform が最後に apply した状態が記録されています。Terraform のコードと実際にプロビジョニングされたリソースの状態をマッピングできます。  
※出力の一部をマスクしています。

```sh
$ cat terraform.tfstate
```
```sh
{
  "version": 4,
  "terraform_version": "1.5.7",
  "serial": 6,
  "lineage": "477718e5-e06e-7336-e338-b7a009a635dd",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "oci_objectstorage_bucket",
      "name": "objectstorage_bucket",
      "provider": "provider[\"registry.terraform.io/oracle/oci\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "access_type": "NoPublicAccess",
            "approximate_count": "0",
            "approximate_size": "0",
            "auto_tiering": "Disabled",
            "bucket_id": "ocid1.bucket.oc1.xx-xxxxx-xx.",
            "compartment_id": "ocid1.compartment.oc1..",
            "created_by": "ocid1.user.oc1..",
            "defined_tags": {},
            "etag": "5c39bd10-f705-48f6-a9ca-00ed480ac661",
            "freeform_tags": {},
            "id": "n/xxxxx/b/terraform",
            "is_read_only": false,
            "kms_key_id": null,
            "metadata": {},
            "name": "terraform",
            "namespace": "xxxxx",
            "object_events_enabled": false,
            "object_lifecycle_policy_etag": null,
            "replication_enabled": false,
            "retention_rules": [],
            "storage_tier": "Standard",
            "time_created": "2025-01-08 10:21:50.068 +0000 UTC",
            "timeouts": null,
            "versioning": "Disabled"
          },
          "sensitive_attributes": [],
          "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjoxMjAwMDAwMDAwMDAwLCJkZWxldGUiOjEyMDAwMDAwMDAwMDAsInVwZGF0ZSI6MTIwMDAwMDAwMDAwMH19"
        }
      ]
    }
  ],
  "check_results": null
}
```

ここで、再度適用を試みると、既に存在しているので作成されないことが分かります。  
※ -auto-approve オプションは、apply 時に yes の入力を省きます。

```sh
$ terraform apply -auto-approve
```
```sh
oci_objectstorage_bucket.objectstorage_bucket: Refreshing state... [id=n/orasejapan/b/terraform]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.
```

プロビジョニングしたリソースを削除します。  
※出力の一部をマスクしています。

```sh
$ terraform destroy
```
```sh
oci_objectstorage_bucket.objectstorage_bucket: Refreshing state... [id=n/orasejapan/b/terraform]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  - destroy

Terraform will perform the following actions:

  # oci_objectstorage_bucket.objectstorage_bucket will be destroyed
  - resource "oci_objectstorage_bucket" "objectstorage_bucket" {
      - access_type           = "NoPublicAccess" -> null
      - approximate_count     = "0" -> null
      - approximate_size      = "0" -> null
      - auto_tiering          = "Disabled" -> null
      - bucket_id             = "ocid1.bucket.oc1.xx-xxxxx-xx." -> null
      - compartment_id        = "ocid1.compartment.oc1.." -> null
      - created_by            = "ocid1.user.oc1.." -> null
      - defined_tags          = {} -> null
      - etag                  = "5c39bd10-f705-48f6-a9ca-00ed480ac661" -> null
      - freeform_tags         = {} -> null
      - id                    = "n/xxxxx/b/terraform" -> null
      - is_read_only          = false -> null
      - metadata              = {} -> null
      - name                  = "terraform" -> null
      - namespace             = "xxxxx" -> null
      - object_events_enabled = false -> null
      - replication_enabled   = false -> null
      - storage_tier          = "Standard" -> null
      - time_created          = "2025-01-08 10:21:50.068 +0000 UTC" -> null
      - versioning            = "Disabled" -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.
```

「yes」と入力して実行します。

```sh
Enter a value: yes
```
```sh
oci_objectstorage_bucket.objectstorage_bucket: Destroying... [id=n/orasejapan/b/terraform]
oci_objectstorage_bucket.objectstorage_bucket: Destruction complete after 2s
```

削除されたかを確認します。

```sh
$ terraform show
```
```sh
The state file is empty. No resources are represented.
```