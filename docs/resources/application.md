---
subcategory: "Applications"
---

# Resource: azuread_application

Manages an application registration within Azure Active Directory.

## API Permissions

The following API permissions are required in order to use this resource.

When authenticated with a service principal, this resource requires the following application role: `Application.ReadWrite.All`

-> It is usually possible to create applications using this resource with just the `Application.ReadWrite.OwnedBy` application role, provided the principal being used to run Terraform is included in the `owners` property. However, this is not officially supported by the API so if you receive a `403` you need to investigate what API call is failing and add additional permissions as necessary. One commonly needed additional permission is `User.Read.All`, in case you specify additional `owners`.

When authenticated with a user principal, this resource requires one of the following directory roles: `Application Administrator` or `Global Administrator`

## Example Usage

*Create an application*

```terraform
data "azuread_client_config" "current" {}

resource "azuread_application" "example" {
  display_name     = "example"
  identifier_uris  = ["api://example-app"]
  logo_image       = filebase64("/path/to/logo.png")
  owners           = [data.azuread_client_config.current.object_id]
  sign_in_audience = "AzureADMultipleOrgs"

  api {
    mapped_claims_enabled          = true
    requested_access_token_version = 2

    known_client_applications = [
      azuread_application.known1.application_id,
      azuread_application.known2.application_id,
    ]

    oauth2_permission_scope {
      admin_consent_description  = "Allow the application to access example on behalf of the signed-in user."
      admin_consent_display_name = "Access example"
      enabled                    = true
      id                         = "96183846-204b-4b43-82e1-5d2222eb4b9b"
      type                       = "User"
      user_consent_description   = "Allow the application to access example on your behalf."
      user_consent_display_name  = "Access example"
      value                      = "user_impersonation"
    }

    oauth2_permission_scope {
      admin_consent_description  = "Administer the example application"
      admin_consent_display_name = "Administer"
      enabled                    = true
      id                         = "be98fa3e-ab5b-4b11-83d9-04ba2b7946bc"
      type                       = "Admin"
      value                      = "administer"
    }
  }

  app_role {
    allowed_member_types = ["User", "Application"]
    description          = "Admins can manage roles and perform all task actions"
    display_name         = "Admin"
    enabled              = true
    id                   = "1b19509b-32b1-4e9f-b71d-4992aa991967"
    value                = "admin"
  }

  app_role {
    allowed_member_types = ["User"]
    description          = "ReadOnly roles have limited query access"
    display_name         = "ReadOnly"
    enabled              = true
    id                   = "497406e4-012a-4267-bf18-45a1cb148a01"
    value                = "User"
  }

  feature_tags {
    enterprise = true
    gallery    = true
  }

  optional_claims {
    access_token {
      name = "myclaim"
    }

    access_token {
      name = "otherclaim"
    }

    id_token {
      name                  = "userclaim"
      source                = "user"
      essential             = true
      additional_properties = ["emit_as_roles"]
    }

    saml2_token {
      name = "samlexample"
    }
  }

  required_resource_access {
    resource_app_id = "00000003-0000-0000-c000-000000000000" # Microsoft Graph

    resource_access {
      id   = "df021288-bdef-4463-88db-98f22de89214" # User.Read.All
      type = "Role"
    }

    resource_access {
      id   = "b4e74841-8e56-480b-be8b-910348b18b4c" # User.ReadWrite
      type = "Scope"
    }
  }

  required_resource_access {
    resource_app_id = "c5393580-f805-4401-95e8-94b7a6ef2fc2" # Office 365 Management

    resource_access {
      id   = "594c1fb6-4f81-4475-ae41-0c394909246c" # ActivityFeed.Read
      type = "Role"
    }
  }

  web {
    homepage_url  = "https://app.example.net"
    logout_url    = "https://app.example.net/logout"
    redirect_uris = ["https://app.example.net/account"]

    implicit_grant {
      access_token_issuance_enabled = true
      id_token_issuance_enabled     = true
    }
  }
}
```

*Create application from a gallery template*

```terraform
data "azuread_application_template" "example" {
  display_name = "Marketo"
}

resource "azuread_application" "example" {
  display_name = "example"
  template_id  = data.azuread_application_template.example.template_id
}

resource "azuread_service_principal" "example" {
  application_id = azuread_application.example.application_id
  use_existing   = true
}
```

## Argument Reference

The following arguments are supported:

* `api` - (Optional) An `api` block as documented below, which configures API related settings for this application.
* `app_role` - (Optional) A collection of `app_role` blocks as documented below. For more information see [official documentation on Application Roles](https://docs.microsoft.com/en-us/azure/architecture/multitenant-identity/app-roles).
* `description` - (Optional) A description of the application, as shown to end users.
* `device_only_auth_enabled` - (Optional) Specifies whether this application supports device authentication without a user. Defaults to `false`.
* `display_name` - (Required) The display name for the application.
* `fallback_public_client_enabled` - (Optional) Specifies whether the application is a public client. Appropriate for apps using token grant flows that don't use a redirect URI. Defaults to `false`.
* `feature_tags` - (Optional) A `feature_tags` block as described below. Cannot be used together with the `tags` property.

-> **Features and Tags** Features are configured for an application using tags, and are provided as a shortcut to set the corresponding magic tag value for each feature. You cannot configure `feature_tags` and `tags` for an application at the same time, so if you need to assign additional custom tags it's recommended to use the `tags` property instead. Tag values also propagate to any linked service principals.

* `group_membership_claims` - (Optional) Configures the `groups` claim issued in a user or OAuth 2.0 access token that the app expects. Possible values are `None`, `SecurityGroup`, `DirectoryRole`, `ApplicationGroup` or `All`.
* `identifier_uris` - (Optional) A set of user-defined URI(s) that uniquely identify an application within its Azure AD tenant, or within a verified custom domain if the application is multi-tenant.
* `logo_image` - (Optional) A logo image to upload for the application, as a raw base64-encoded string. The image should be in gif, jpeg or png format. Note that once an image has been uploaded, it is not possible to remove it without replacing it with another image.
* `marketing_url` - (Optional) URL of the application's marketing page.
* `notes` - (Optional) User-specified notes relevant for the management of the application.
* `oauth2_post_response_required` - (Optional) Specifies whether, as part of OAuth 2.0 token requests, Azure AD allows POST requests, as opposed to GET requests. Defaults to `false`, which specifies that only GET requests are allowed.
* `optional_claims` - (Optional) An `optional_claims` block as documented below.
* `owners` - (Optional) A set of object IDs of principals that will be granted ownership of the application. Supported object types are users or service principals. By default, no owners are assigned.

-> **Ownership of Applications** It's recommended to always specify one or more application owners, including the principal being used to execute Terraform, such as in the example above.

* `prevent_duplicate_names` - (Optional) If `true`, will return an error if an existing application is found with the same name. Defaults to `false`.
* `privacy_statement_url` - (Optional) URL of the application's privacy statement.
* `public_client` - (Optional) A `public_client` block as documented below, which configures non-web app or non-web API application settings, for example mobile or other public clients such as an installed application running on a desktop device.
* `required_resource_access` - (Optional) A collection of `required_resource_access` blocks as documented below.
* `service_management_reference` - (Optional) References application context information from a Service or Asset Management database.
* `sign_in_audience` - (Optional) The Microsoft account types that are supported for the current application. Must be one of `AzureADMyOrg`, `AzureADMultipleOrgs`, `AzureADandPersonalMicrosoftAccount` or `PersonalMicrosoftAccount`. Defaults to `AzureADMyOrg`.

~> **Changing `sign_in_audience` for existing applications** When updating an existing application to use a `sign_in_audience` value of `AzureADandPersonalMicrosoftAccount` or `PersonalMicrosoftAccount`, your configuration may no longer be valid. Refer to [official documentation](https://docs.microsoft.com/en-gb/azure/active-directory/develop/supported-accounts-validation) to understand the differences in supported configurations. Where possible, the provider will attempt to validate your configuration and try to avoid applying unsupported settings to your application.

* `single_page_application` - (Optional) A `single_page_application` block as documented below, which configures single-page application (SPA) related settings for this application.
* `support_url` - (Optional) URL of the application's support page.
* `tags` - (Optional) A set of tags to apply to the application for configuring specific behaviours of the application and linked service principals. Note that these are not provided for use by practitioners. Cannot be used together with the `feature_tags` block.

-> **Tags and Features** Azure Active Directory uses special tag values to configure the behavior of applications. These can be specified using either the `tags` property or with the `feature_tags` block. If you need to set any custom tag values not supported by the `feature_tags` block, it's recommended to use the `tags` property. Tag values also propagate to any linked service principals.

* `template_id` - (Optional) Unique ID for a templated application in the Azure AD App Gallery, from which to create the application. Changing this forces a new resource to be created.
* `terms_of_service_url` - (Optional) URL of the application's terms of service statement.
* `web` - (Optional) A `web` block as documented below, which configures web related settings for this application.

-> **Application Name Uniqueness** Application names are not unique within Azure Active Directory. Use the `prevent_duplicate_names` argument to check for existing applications if you want to avoid name collisions.

---

`api` block supports the following:

* `known_client_applications` - (Optional) A set of application IDs (client IDs), used for bundling consent if you have a solution that contains two parts: a client app and a custom web API app.
* `mapped_claims_enabled` - (Optional) Allows an application to use claims mapping without specifying a custom signing key. Defaults to `false`.
* `oauth2_permission_scope` - (Optional) One or more `oauth2_permission_scope` blocks as documented below, to describe delegated permissions exposed by the web API represented by this application.
* `requested_access_token_version` - (Optional) The access token version expected by this resource. Must be one of `1` or `2`, and must be `2` when `sign_in_audience` is either `AzureADandPersonalMicrosoftAccount` or `PersonalMicrosoftAccount` Defaults to `1`.

---

`oauth2_permission_scope` blocks support the following:

* `admin_consent_description` - (Required) Delegated permission description that appears in all tenant-wide admin consent experiences, intended to be read by an administrator granting the permission on behalf of all users.
* `admin_consent_display_name` - (Required) Display name for the delegated permission, intended to be read by an administrator granting the permission on behalf of all users.
* `enabled` - (Optional) Determines if the permission scope is enabled. Defaults to `true`.
* `id` - (Required) The unique identifier of the delegated permission. Must be a valid UUID.

-> **Tip: Generating a UUID for the `id` field** To generate a value for the `id` field in cases where the actual UUID is not important, you can use the `random_uuid` resource. See the [application example](https://github.com/hashicorp/terraform-provider-azuread/tree/main/examples/application) in the provider repository.

* `type` - (Optional) Whether this delegated permission should be considered safe for non-admin users to consent to on behalf of themselves, or whether an administrator should be required for consent to the permissions. Defaults to `User`. Possible values are `User` or `Admin`.
* `user_consent_description` - (Optional) Delegated permission description that appears in the end user consent experience, intended to be read by a user consenting on their own behalf.
* `user_consent_display_name` - (Optional) Display name for the delegated permission that appears in the end user consent experience.
* `value` - (Optional) The value that is used for the `scp` claim in OAuth 2.0 access tokens.

~> **Default `user_impersonation` Scope** Unlike the Azure Portal, applications created with the Terraform AzureAD provider do not get assigned a default `user_impersonation` scope. You will need to include a block for the `user_impersonation` scope if you need it for your application.

-> **Roles and Permission Scopes** In Azure Active Directory, application roles (`app_role`) and permission scopes (`oauth2_permission_scope`) exported by an application share the same namespace and cannot contain duplicate `value`s. Terraform will attempt to detect this during a plan or apply operation.

---

`app_role` block supports the following:

* `allowed_member_types` - (Required) Specifies whether this app role definition can be assigned to users and groups by setting to `User`, or to other applications (that are accessing this application in a standalone scenario) by setting to `Application`, or to both.
* `description` - (Required) Description of the app role that appears when the role is being assigned and, if the role functions as an application permissions, during the consent experiences.
* `display_name` - (Required) Display name for the app role that appears during app role assignment and in consent experiences.
* `enabled` - (Optional) Determines if the app role is enabled. Defaults to `true`.
* `id` - (Required) The unique identifier of the app role. Must be a valid UUID.

-> **Tip: Generating a UUID for the `id` field** To generate a value for the `id` field in cases where the actual UUID is not important, you can use the `random_uuid` resource. See the [application example](https://github.com/hashicorp/terraform-provider-azuread/tree/main/examples/application) in the provider repository.

* `value` - (Optional) The value that is used for the `roles` claim in ID tokens and OAuth 2.0 access tokens that are authenticating an assigned service or user principal.

-> **Roles and Permission Scopes** In Azure Active Directory, application roles (`app_role`) and permission scopes (`oauth2_permission_scope`) exported by an application share the same namespace and cannot contain duplicate `value`s. Terraform will attempt to detect this during a plan or apply operation.

---

`feature_tags` block supports the following:

* `custom_single_sign_on` - (Optional) Whether this application represents a custom SAML application for linked service principals. Enabling this will assign the `WindowsAzureActiveDirectoryCustomSingleSignOnApplication` tag. Defaults to `false`.
* `enterprise` - (Optional) Whether this application represents an Enterprise Application for linked service principals. Enabling this will assign the `WindowsAzureActiveDirectoryIntegratedApp` tag. Defaults to `false`.
* `gallery` - (Optional) Whether this application represents a gallery application for linked service principals. Enabling this will assign the `WindowsAzureActiveDirectoryGalleryApplicationNonPrimaryV1` tag. Defaults to `false`.
* `hide` - (Optional) Whether this app is invisible to users in My Apps and Office 365 Launcher. Enabling this will assign the `HideApp` tag. Defaults to `false`.

---

`optional_claims` block supports the following:

* `access_token` - (Optional) One or more `access_token` blocks as documented below.
* `id_token` - (Optional) One or more `id_token` blocks as documented below.
* `saml2_token` - (Optional) One or more `saml2_token` blocks as documented below.

---

`access_token`, `id_token` and `saml2_token` blocks support the following:

* `additional_properties` - List of additional properties of the claim. If a property exists in this list, it modifies the behaviour of the optional claim.
* `essential` - Whether the claim specified by the client is necessary to ensure a smooth authorization experience.
* `name` - The name of the optional claim.
* `source` - The source of the claim. If `source` is absent, the claim is a predefined optional claim. If `source` is `user`, the value of `name` is the extension property from the user object.

---

`public_client` block supports the following:

* `redirect_uris` - (Optional) A set of URLs where user tokens are sent for sign-in, or the redirect URIs where OAuth 2.0 authorization codes and access tokens are sent. Must be a valid `https` or `ms-appx-web` URL.

---

`required_resource_access` block supports the following:

* `resource_access` - (Required) A collection of `resource_access` blocks as documented below, describing OAuth2.0 permission scopes and app roles that the application requires from the specified resource.
* `resource_app_id` - (Required) The unique identifier for the resource that the application requires access to. This should be the Application ID of the target application.

-> **Note:** Documentation on `resource_app_id` values for Microsoft APIs can be difficult to find, but you can use the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest#az_ad_sp_list) to find them. (e.g. `az ad sp list --display-name "Microsoft Graph" --query '[].{appDisplayName:appDisplayName, appId:appId}'`)

---

`resource_access` block supports the following:

* `id` - (Required) The unique identifier for an app role or OAuth2 permission scope published by the resource application.
* `type` - (Required) Specifies whether the `id` property references an app role or an OAuth2 permission scope. Possible values are `Role` or `Scope`.

---

`single_page_application` block supports the following:

* `redirect_uris` - (Optional) A set of URLs where user tokens are sent for sign-in, or the redirect URIs where OAuth 2.0 authorization codes and access tokens are sent. Must be a valid `https` URL.

---

`web` block supports the following:

* `homepage_url` - (Optional) Home page or landing page of the application.
* `implicit_grant` - (Optional) An `implicit_grant` block as documented above.
* `logout_url` - (Optional) The URL that will be used by Microsoft's authorization service to sign out a user using front-channel, back-channel or SAML logout protocols.
* `redirect_uris` - (Optional) A set of URLs where user tokens are sent for sign-in, or the redirect URIs where OAuth 2.0 authorization codes and access tokens are sent. Must be a valid `http` URL or a URN.

---

`implicit_grant` block supports the following:

* `access_token_issuance_enabled` - (Optional) Whether this web application can request an access token using OAuth 2.0 implicit flow.
* `id_token_issuance_enabled` - (Optional) Whether this web application can request an ID token using OAuth 2.0 implicit flow.

## Attributes Reference

In addition to all arguments above, the following attributes are exported:

* `app_role_ids` - A mapping of app role values to app role IDs, intended to be useful when referencing app roles in other resources in your configuration.
* `application_id` - The Application ID (also called Client ID).
* `disabled_by_microsoft` - Whether Microsoft has disabled the registered application. If the application is disabled, this will be a string indicating the status/reason, e.g. `DisabledDueToViolationOfServicesAgreement`
* `logo_url` - CDN URL to the application's logo, as uploaded with the `logo_image` property.
* `oauth2_permission_scope_ids` - A mapping of OAuth2.0 permission scope values to scope IDs, intended to be useful when referencing permission scopes in other resources in your configuration.
* `object_id` - The application's object ID.
* `publisher_domain` - The verified publisher domain for the application.

## Import

Applications can be imported using their object ID, e.g.

```shell
terraform import azuread_application.test 00000000-0000-0000-0000-000000000000
```
