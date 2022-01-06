## Configuring a Resource

[Terrajet] generates as much as it could using the available information in the
Terraform resource schema. This includes an XRM-conformant schema of the
resource, controller logic, late initialization, sensitive data handling etc.
However, there are still couple of information that requires some input
configuration which could easily be provided by checking the Terraform
documentation of the resource:

- [External name]
- [Cross Resource Referencing]
- [Additional Sensitive Fields and Custom Connection Details]
- [Late Initialization Behavior]

### External Name

Crossplane uses an annotation in managed resource CR to identify the external
resource which is managed by Crossplane. See [the external name documentation]
for more details. The format and source of the external name depends on the
cloud provider; sometimes it could simply be the name of resource
(e.g. S3 Bucket), and sometimes it is an auto-generated id by cloud API
(e.g. VPC id ). This is something specific to resource, and we need some input
configuration for terrajet to appropriately generate a resource.

Since Terraform already needs the same identifier to import a resource, most
helpful part of resource documentation is the [import section].

This is [the struct that holds the External Name configuration]:

```go
// ExternalName contains all information that is necessary for naming operations,
// such as removal of those fields from spec schema and calling Configure function
// to fill attributes with information given in external name.
type ExternalName struct {
    // SetIdentifierArgumentFn sets the name of the resource in Terraform argument
    // map.
    SetIdentifierArgumentFn SetIdentifierArgumentFn
    
    // OmittedFields are the ones you'd like to be removed from the schema since
    // they are specified via external name. You can omit only the top level fields.
    // No field is omitted by default.
    OmittedFields []string
    
    // DisableNameInitializer allows you to specify whether the name initializer
    // that sets external name to metadata.name if none specified should be disabled.
    // It needs to be disabled for resources whose external name includes information
    // more than the actual name of the resource, like subscription ID or region
    // etc. which is unlikely to be included in metadata.name
    DisableNameInitializer bool
}
```

Comments explain the purpose of each field but let's clarify further with some
examples.

Checking the [import section of aws_vpc], we see that this resource is being
imported with `vpc id`. When we check the [arguments list] and provided
[example usages], it is clear that this **id** is not something that user
provides, rather generated by AWS API. Hence, we need to disable name
initializer, which simply sets the external-name annotation to `metadata.name`
of the resource.

```go
DisableNameInitializer: true
```

Since we have no related fields in the [arguments list] that could be used to
build the external-name, we don't need to omit any fields (`OmittedFields`) or
need to use external name to set some arguments (`SetIdentifierArgumentFn`).
Hence, we end up the following external name configuration for `aws_vpc`
resource:

```go
func Configure(p *config.Provider) {
        p.AddResourceConfigurator("aws_vpc", func (r *config.Resource) {
        r.ExternalName = config.ExternalName{
            // Set to true explicitly since the value is calculated by AWS.
            DisableNameInitializer: true,
        }
    })
}
```

And for this specific case, where Provider assigns identifier of the resource
independent of resource specification, Terrajet has a default external name
configuration that is [IdentifierFromProvider] which we can simply use here
doing the same as above:

```go
func Configure(p *config.Provider) {
    p.AddResourceConfigurator("aws_vpc", func (r *config.Resource) {
        r.ExternalName = config.IdentifierFromProvider
    })
}
```

Let's check another resource, [aws_s3_bucket] which requires some other
configuration. Reading the [import section of s3 bucket] we see that bucket is
imported with its **name** which is provided with the [bucket] argument.
We can just use the CR name as the bucket name, and we don't have to disable
name initializer as we did above.

However, since we are using metadata name as `bucket` argument, we need the
following two:

- Fill `bucket` attribute using external-name annotation, so that Terraform
  knows the value we want to provide:

    ```go
    SetIdentifierArgumentFn: func(base map[string]interface{}, name string) {
    	base["bucket"] = name
    },
    ```

- Omit `bucket` and `bucket_prefix` from the crd spec, so that we don't have
  multiple inputs for the same thing (name of the bucket):
    ```go
    OmittedFields: []string{
        "bucket",
        "bucket_prefix",
    },
    ```

Hence, we end up the following external name configuration for `aws_s3_bucket`
resource:

```go
func Configure(p *config.Provider) {
    p.AddResourceConfigurator("aws_s3_bucket", func(r *config.Resource) {
        r.ExternalName = config.ExternalName{
            SetIdentifierArgumentFn: func(base map[string]interface{}, name string) {
                base["bucket"] = name
            },
            OmittedFields: []string{
                "bucket",
                "bucket_prefix",
            },
        }
    })
}
```

Please note, you can always check resource configurations of existing Providers
as further examples under `config/<group>/config.go`.

### Cross Resource Referencing

Crossplane uses cross resource referencing to [handle dependencies] between
managed resources. For example, if you have an IAM User defined as a Crossplane
managed resource, and you want to create an Access Key for that user, you would
need to refer to the User CR from the Access Key resource. This is handled by
cross resource referencing.

See how the [user] referenced at `forProvider.userRef.name` field of the
Access Key in the following example:

```yaml
apiVersion: iam.aws.tf.crossplane.io/v1alpha1
kind: User
metadata:
  name: sample-user
spec:
  forProvider: {}
---
apiVersion: iam.aws.tf.crossplane.io/v1alpha1
kind: AccessKey
metadata:
  name: sample-access-key
spec:
  forProvider:
    userRef:
      name: sample-user
  writeConnectionSecretToRef:
    name: sample-access-key-secret
    namespace: crossplane-system
```

Historically, reference resolution method were written by hand which requires
some effort, however, with the latest Crossplane code generation tooling, it is
now possible to [generate reference resolution methods] by just adding some
marker on the fields. Now, the only manual step for generating cross resource
references is to provide which field of a resource depends on which information
(e.g. `id`, `name`, `arn` etc.) from the other.

In Terrajet, we have a [configuration] to provide this information for a field:

```go
// Reference represents the Crossplane options used to generate
// reference resolvers for fields
type Reference struct {
    // Type is the type name of the CRD if it is in the same package or
    // <package-path>.<type-name> if it is in a different package.
    Type string
    // Extractor is the function to be used to extract value from the
    // referenced type. Defaults to getting external name.
    // Optional
    Extractor string
    // RefFieldName is the field name for the Reference field. Defaults to
    // <field-name>Ref or <field-name>Refs.
    // Optional
    RefFieldName string
    // SelectorFieldName is the field name for the Selector field. Defaults to
    // <field-name>Selector.
    // Optional
    SelectorFieldName string
}
```

For a resource that we want to generate, we need to check its argument list in
Terraform documentation and figure out which field needs reference to which
resource.

Let's check [iam_access_key] as an example. In the argument list, we see the
[user] field which requires a reference to a IAM user. So, we need to the
following referencing configuration:

```go
func Configure(p *config.Provider) {
    p.AddResourceConfigurator("aws_iam_access_key", func (r *config.Resource) {
        r.References["user"] = config.Reference{
            Type: "User",
        }
    })
}
```

Please note the value of `Type` field needs to be a string representing the Go
type of the resource. Since, `AccessKey` and `User` resources are under the same
go package, we don't need to provide the package path. However, this is not
always the case and referenced resources might be in different package. In that
case, we would need to provide the full path. Referencing to a [kms key] from
`aws_ebs_volume` resource is a good example here:

```go
func Configure(p *config.Provider) {
	p.AddResourceConfigurator("aws_ebs_volume", func(r *config.Resource) {
		r.References["kms_key_id"] = config.Reference{
			Type: "github.com/crossplane-contrib/provider-tf-aws/apis/kms/v1alpha1.Key",
		}
	})
}
```

### Additional Sensitive Fields and Custom Connection Details

Crossplane stores sensitive information of a managed resource in a Kubernetes
secret, together with some additional fields that would help consumption of the
resource, a.k.a. [connection details].

In Terrajet, we already [handle sensitive fields] that are marked as sensitive
in Terraform schema and no further action required for them. Terrajet will 
properly hide these fields from CRD spec and status by converting to a secret
reference or storing in connection details secret respectively. However, we
still have some custom configuration API that would allow including additional
fields into connection details secret no matter they are sensitive or not.

As an example, let's use `aws_iam_access_key`. Currently, Terrajet stores all
sensitive fields in Terraform schema as prefixed with `attribute.`, so without
any `AdditionalConnectionDetailsFn`, connection resource will have
`attribute.id` and `attribute.secret` corresponding to [id] and [secret] fields
respectively. To see them with more common keys, i.e. `aws_access_key_id` and 
`aws_secret_access_key`, we would need to make the following configuration:

```go
func Configure(p *config.Provider) {
	p.AddResourceConfigurator("aws_iam_access_key", func(r *config.Resource) {
		r.Sensitive = config.Sensitive{
			AdditionalConnectionDetailsFn: func(attr map[string]interface{}) (map[string][]byte, error) {
				conn := map[string][]byte{}
				if a, ok := attr["id"].(string); ok {
					conn["aws_access_key_id"] = []byte(a)
				}
				if a, ok := attr["secret"].(string); ok {
					conn["aws_secret_access_key"] = []byte(a)
				}
				return conn, nil
			},
		}
	})
}
```

This will produce a connection details secret as follows:

```yaml
apiVersion: v1
data:
  attribute.id: QUtJQVk0QUZUVFNFNDI2TlhKS0I=
  attribute.secret: ABCxyzRedacted==
  attribute.ses_smtp_password_v4: QQ00REDACTED==
  aws_access_key_id: QUtJQVk0QUZUVFNFNDI2TlhKS0I=
  aws_secret_access_key: ABCxyzRedacted==
kind: Secret
```

### Late Initialization Behavior
Terrajet runtime automatically performs late-initialization during
an [`external.Observe`] call with means of runtime reflection.
State of the world observed by Terraform CLI is used to initialize
any `nil`-valued pointer parameters in the managed resource's `spec`.
In most of the cases no custom configuration should be necessary for
late-initialization to work. However, there are certain cases where
you will want/need to customize late-initialization behaviour. Thus,
Terrajet provides an extensible [late-initialization customization API]
that controls late-initialization behaviour.

The associated resource struct is defined [here](https://github.com/crossplane/terrajet/blob/c9e21387298d8ed59fcd71c7f753ec401a3383a5/pkg/config/resource.go#L91) as follows:
```go
// LateInitializer represents configurations that control
// late-initialization behaviour
type LateInitializer struct {
    // IgnoredFields are the canonical field names to be skipped during
    // late-initialization
    IgnoredFields []string
}
```
Currently, it only involves a configuration option to specify
certain `spec` parameters to be ignored during late-initialization.
Each element of the `LateInitializer.IgnoredFields` slice represents
the canonical path relative to the parameters struct for the managed resource's `Spec`
using `Go` type names as path elements. As an example, with the following type definitions:
```go
type Subnet struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec              SubnetSpec   `json:"spec"`
    Status            SubnetStatus `json:"status,omitempty"`
}

type SubnetSpec struct {
    ForProvider     SubnetParameters `json:"forProvider"`
    ...
}

type DelegationParameters struct {
    // +kubebuilder:validation:Required
    Name *string `json:"name" tf:"name,omitempty"`
    ...
}

type SubnetParameters struct {
    // +kubebuilder:validation:Optional
    AddressPrefix *string `json:"addressPrefix,omitempty" tf:"address_prefix,omitempty"`
    // +kubebuilder:validation:Optional
    Delegation []DelegationParameters `json:"delegation,omitempty" tf:"delegation,omitempty"`
    ...
}
```
If you would like to have the late-initialization library *not* to process the
[`address_prefix`] parameter field, then the following configuration where we
specify the parameter field path is sufficient:

```go
func Configure(p *config.Provider) {
	p.AddResourceConfigurator("azurerm_subnet", func(r *config.Resource) {
		r.LateInitializer = config.LateInitializer{
			IgnoredFields: []string{"address_prefix"},
		}
	})
}
```

In most cases, custom late-initialization configuration will not be necessary. 
However, after generating a new managed resource and observing its behaviour 
(at runtime), it may turn out that late-initialization behaviour needs
customization. For certain resources like the `provider-tf-azure`'s
`PostgresqlServer` resource, we have observed that Terraform state contains
values for mutually exclusive parameters, e.g., for `PostgresqlServer`, both
`StorageMb` and `StorageProfile[].StorageMb` get late-initialized. Upon next
reconciliation, we generate values for both parameters in the Terraform
configuration, and although they happen to have the same value, Terraform
configuration validation requires them to be mutually exclusive. Currently, we
observe this behaviour at runtime, and upon observing that the resource cannot
transition to the `Ready` state and acquires the Terraform validation error
message in its `status.conditions`, we do the `LateInitializer.IgnoreFields`
custom configuration detailed above to skip one of the mutually exclusive fields
during late-initialization.

[comment]: <> (References)

[Terrajet]: https://github.com/crossplane/terrajet
[External name]: #external-name
[Cross Resource Referencing]: #cross-resource-referencing
[Additional Sensitive Fields and Custom Connection Details]: #additional-sensitive-fields-and-custom-connection-details
[Late Initialization Behavior]: #late-initialization-behavior
[the external name documentation]: https://crossplane.io/docs/v1.4/concepts/managed-resources.html#external-name
[import section]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_access_key#import
[the struct that holds the External Name configuration]: https://github.com/crossplane/terrajet/blob/08e5e93f8a93c6628a4302fb520cd4be4b6cab07/pkg/config/resource.go#L50
[aws_vpc]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc
[import section of aws_vpc]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc#import
[arguments list]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc#argument-reference
[example usages]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/vpc#example-usage
[IdentifierFromProvider]: https://github.com/crossplane/terrajet/blob/08e5e93f8a93c6628a4302fb520cd4be4b6cab07/pkg/config/defaults.go#L43
[aws_s3_bucket]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket
[import section of s3 bucket]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket#import
[bucket]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket#bucket
[handle dependencies]: https://crossplane.io/docs/v1.4/concepts/managed-resources.html#dependencies
[user]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_access_key#user
[generate reference resolution methods]: https://github.com/crossplane/crossplane-tools/pull/35
[configuration]: https://github.com/crossplane/terrajet/blob/874bb6ad5cff9741241fb790a3a5d71166900860/pkg/config/resource.go#L77
[iam_access_key]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_access_key#argument-reference
[kms key]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ebs_volume#kms_key_id
[connection details]: https://crossplane.io/docs/v1.4/concepts/managed-resources.html#connection-details
[handle sensitive fields]: https://github.com/crossplane/terrajet/pull/77
[id]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_access_key#id
[secret]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_access_key#secret
[`external.Observe`]: https://github.com/crossplane/terrajet/blob/874bb6ad5cff9741241fb790a3a5d71166900860/pkg/controller/external.go#L149
[late-initialization customization API]: https://github.com/crossplane/terrajet/blob/874bb6ad5cff9741241fb790a3a5d71166900860/pkg/resource/lateinit.go#L86
[`address_prefix`]: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/subnet#address_prefix